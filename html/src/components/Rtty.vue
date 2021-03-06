<template>
    <div id="rtty">
        <div ref="terminal" class="terminal" @contextmenu="$vuecontextmenu()"></div>
        <VueContextMenu :menulists="menulists" @contentmenu-click="contentmenuClick"></VueContextMenu>
        <Modal v-model="upfile.modal" width="380" :mask-closable="false" @on-cancel="cancelUpfile">
            <p slot="header"><span>{{ $t('Upload file to device') }}</span></p>
            <Upload :before-upload="beforeUpload" action="">
                <Button type="ghost" icon="Upload">{{ $t('Please select the file to upload') }}</Button>
            </Upload>
            <div v-if="upfile.file !== null">{{ upfile.file.name }}</div>
            <div slot="footer">
                <Button type="primary" size="large" long @click="doUpload">{{ $t('Click to upload') }}</Button>
            </div>
        </Modal>
    </div>
</template>

<script>

import * as Socket from 'simple-websocket';
import { Terminal } from 'xterm'
import 'xterm/lib/xterm.css'
import * as fit from 'xterm/lib/addons/fit/fit';
import * as overlay from '@/overlay';
import Utf8ArrayToStr from '@/utf8array_str'
import 'zmodem.js/dist/zmodem.devel'

Terminal.applyAddon(fit);
Terminal.applyAddon(overlay);

const Pbf = require('pbf');
const rttyMsg = require('@/rtty.proto').rtty_message;

function rttyMsgInit(type, msg) {
    let pbf = new Pbf();

    msg.version = 2;
    msg.type = rttyMsg.Type[type].value;
    rttyMsg.write(msg, pbf);

    return pbf.finish();
}

export default {
    name: 'Rtty',
    data() {
        return {
            menulists: [{
                    name: 'increasefontsize',
                    caption: this.$t('Increase font size')
                },{
                    name: 'decreasefontsize',
                    caption: this.$t('Decrease font size')
                }
            ],
            upfile: {modal: false, file: null},
        }
    },
    methods: {
        logout() {
            if (this.ws) {
                this.ws.destroy();
                delete this.ws;
            }

            if (this.term) {
                this.term.destroy();
                delete this.term;
            }

            this.$router.push('/');
        },
        contentmenuClick(name) {
            let changeFontSize = 0;

            if (!this.term)
                return;

            if (name == 'increasefontsize') {
                changeFontSize = 1;
            } else if (name == 'decreasefontsize') {
                changeFontSize = -1;
            }

            window.setTimeout(() => {
                let size = this.term.getOption('fontSize');

                this.term.setOption('fontSize', size + changeFontSize);
                this.term.fit();
                this.term.focus();
                this.term.refresh();
            }, 50);
        },
        beforeUpload (file) {
            if (file.size > 500 * 1024 * 1024) {
                this.$Message.warning(this.$t('Cannot be greater than 500MB'));
                return false;
            }
            this.upfile.file = file;
            return false;
        },
        cancelUpfile() {
            let zsession = this.zsentry.get_confirmed_session();
            if (zsession) {
                zsession.abort();
                this.term.focus();
            }
        },
        formatTime(ts) {
            let td = 0;
            let th = 0;
            let tm = 0;

            if (ts > 60) {
                tm = Math.floor(ts / 60);
                ts = (ts % 60);
            }

            if (tm > 60) {
                th = Math.floor(tm / 60);
                tm = (tm % 60);
            }

            if (th > 24) {
                td = Math.floor(th / 24);
                th = (th % 24);
            }

            return (td > 0) ? '%02d:%02d:%02d:%02d'.format(td, th, tm, ts) : '%02d:%02d:%02d'.format(th, tm, ts);
        },
        updateProgress(offset, size, start) {
            let now = Math.floor(new Date().getTime() / 1000);
            let percent = 100 * offset / size;
            let consumed = now - start;
            offset /= 1024;
            this.term.write('   %d%%    %d KB    %d KB/sec    %s\r'.format(percent, offset, offset / consumed, this.formatTime(consumed)));
        },
        doUpload() {
            if (!this.upfile.file) {
                this.$Message.error(this.$t('Select the file to upload'));
                return;
            }

            this.upfile.modal = false;
            this.term.focus();

            this.handleSendSession(this.zsentry.get_confirmed_session(), this.upfile.file);
        },
        readFile(file, fr, offset, size) {
            let blob = file.slice(offset, offset + size);
            fr.readAsArrayBuffer(blob);
        },
        handleReceiveSession(zsession) {
            zsession.on("offer", (xfer) => {
                let start_time = Math.floor(new Date().getTime() / 1000);
                let fileInfo = xfer.get_details();
                let size = fileInfo.size;
                let buffer = [];

                this.term.write('Transferring ' + fileInfo.name + '...\n\r');
                this.updateProgress(0, size, start_time);

                xfer.on("input", (payload) => {
                    this.updateProgress(xfer.get_offset(), size, start_time);
                    buffer.push(new Uint8Array(payload));
                });

                xfer.accept().then(() => {
                    Zmodem.Browser.save_to_disk(buffer, fileInfo.name);

                    /* Maybe lose the 'OO' from the sz command. */
                    setTimeout(() => {
                        let zsession = this.zsentry.get_confirmed_session();
                        if (zsession)
                            zsession.abort();
                    }, 100);
                });
            });

            zsession.on("session_end", () => {
                this.term.write('\n');
                let msg = rttyMsgInit('TTY', {sid: this.sid, data: Buffer.from('\n')});
                this.ws.send(msg);
            });

            zsession.start();
        },
        handleSendSession(zsession, file) {
            let start_time = Math.floor(new Date().getTime() / 1000);
            let batch = {
                obj: file,
                name: file.name,
                size: file.size,
                mtime: new Date(file.lastModified),
                files_remaining: 1,
                bytes_remaining: file.size
            };

            zsession.send_offer(batch).then((xfer) => {
                this.term.write('Transferring ' + file.name + '...\n\r');
                if (xfer) {
                    this.updateProgress(0, batch.size, start_time);
                } else {
                    this.term.write(file.name + ' was skipped\n\r');
                    zsession.close().then(() => {
                        this.term.write('\n');
                    });
                    return;
                }

                let reader = new FileReader();

                //This really shouldn’t happen … so let’s
                //blow up if it does.
                reader.onerror = (e) => {
                    console.error("file read error", e);
                    throw("File read error: " + e);
                };

                reader.onload = (e) => {
                    let piece;

                    if (zsession.aborted())
                        return;

                    if (e.target.result.byteLength > 0) {
                        piece = new Uint8Array(e.target.result, xfer, piece);
                        xfer.send(piece);

                        this.updateProgress(xfer.get_offset(), batch.size, start_time);
                    }

                    if (xfer.get_offset() == batch.size) {
                        xfer.end(piece).then(() => {
                            zsession.close().then(() => {
                                this.term.write('\n');
                            });
                        });
                        return;
                    }

                    this.readFile(file, reader, xfer.get_offset(), 8192);
                };

                this.readFile(file, reader, 0, 8192);
            });
        }
    },
    mounted() {
        let devid = this.$route.query.devid;
        let protocol = (location.protocol === 'https:') ? 'wss://' : 'ws://';

        this.username = this.$route.query.username;
        this.password = this.$route.query.password;

        let ws = new Socket(protocol + location.host + '/ws?devid=' + devid);
        this.ws = ws;

        ws.on('connect', () => {
            let term = new Terminal({
                cursorBlink: true,
                fontSize: 16
            });

            term.open(this.$refs['terminal']);
            term.fit();
            term.focus();
            term.showOverlay(term.cols + 'x' + term.rows);

            window.addEventListener('resize', () => {
                clearTimeout(window.resizedFinished);
                window.resizedFinished = setTimeout(() => {
                    term.fit();
                }, 250);
            });

            term.on('resize', (size) => {
                setTimeout(() => {
                    let msg = rttyMsgInit('WINSIZE', {sid: this.sid, cols: size.cols, rows: size.rows});
                    ws.send(msg);
                    term.showOverlay(size.cols + 'x' + size.rows);
                }, 500);
            });

            this.term = term;

            let zsentry = new Zmodem.Sentry({
                to_terminal: (octets) => {
                    this.term.write(Utf8ArrayToStr(octets));
                },
                sender: (octets) => {
                    let msg = rttyMsgInit('TTY', {sid: this.sid, data: Buffer.from(octets)});
                    this.ws.send(msg);
                },
                on_retract: () => {
                    this.upfile.modal = false;
                },
                on_detect: (detection) => {
                    let zsession = detection.confirm();
                    setTimeout(() => {
                        term.write('\n\rStarting zmodem transfer.  Press Ctrl+C to cancel.\n\r');

                        if (zsession.type === "send")
                            this.upfile = {modal: true, file: null};
                        else
                            this.handleReceiveSession(zsession);
                    }, 10);
                }
            });

            this.zsentry = zsentry;

            ws.on('data', (data) => {
                let pbf = new Pbf(data);
                let msg = rttyMsg.read(pbf);

                if (msg.type == rttyMsg.Type.LOGINACK.value) {
                    if (msg.code == rttyMsg.LoginCode.OFFLINE.value) {
                        this.$Message.error(this.$t('Device offline'));
                        this.logout();
                        return;
                    }

                    this.sid = msg.sid;

                    msg = rttyMsgInit('WINSIZE', {sid: this.sid, cols: term.cols, rows: term.rows});
                    ws.send(msg);

                    term.on('data', (data) => {
                        let zsession = zsentry.get_confirmed_session();
                        if (zsession) {
                            if (zsession.aborted())
                                return;

                            /* Ctrl + C */
                            if (data.length == 1 && data.charCodeAt(0) == 3)
                                zsession.abort();
                            return;
                        }

                        let msg = rttyMsgInit('TTY', {sid: this.sid, data: Buffer.from(data)});
                        ws.send(msg);
                    });
                } else if (msg.type == rttyMsg.Type.TTY.value) {
                    if (!this.recvTTYCnt)
                        this.recvTTYCnt = 0;
                    this.recvTTYCnt++;

                    if (this.recvTTYCnt < 4) {
                        let data = Utf8ArrayToStr(msg.data);

                        if (data.match('login:') && this.username && this.username != '') {
                            let msg = rttyMsgInit('TTY', {sid: this.sid, data: Buffer.from(this.username + '\n')});
                            ws.send(msg);
                            return;
                        }

                        if (data.match('Password:') && this.password && this.password != '') {
                            let msg = rttyMsgInit('TTY', {sid: this.sid, data: Buffer.from(this.password + '\n')});
                            ws.send(msg);
                            return;
                        }
                    }

                    zsentry.consume(msg.data);
                }
            });
        });

        ws.on('error', () => {
            this.$Message.error(this.$t('Connect failed'));
            this.logout();
        });

        ws.on('close', () => {
            this.logout();
        });
    }
};
</script>

<style>
    #rtty {
        width: 100%;
        height: 100%;
    }

    .terminal {
        height: 100%;
        padding: 10px;
    }
</style>
