<template>
    <video ref="video" autoplay playsinline></video>
</template>

<script lang="ts">
import BaseVideo from '@/components/video/BaseVideo';
import container from '@/model/ModelContainer';
import ISnackbarState from '@/model/state/snackbar/ISnackbarState';
import * as aribb24js from 'aribb24.js';
import { Component, Prop } from 'vue-property-decorator';
import Mpegts from 'mpegts.js';
import SubtitleUtil from '@/util/SubtitleUtil';

@Component({})
export default class LiveMpegTsVideo extends BaseVideo {
    @Prop({ required: true })
    public videoSrc!: string;

    private snackbarState: ISnackbarState = container.get<ISnackbarState>('ISnackbarState');
    private mepgtsPlayer: Mpegts.Player | null = null;
    private captionRenderer: aribb24js.CanvasB24Renderer | null = null;
    private superimposeRenderer: aribb24js.CanvasB24Renderer | null = null;

    public mounted(): void {
        super.mounted();
    }

    public async beforeDestroy(): Promise<void> {
        if (this.mepgtsPlayer !== null) {
            this.mepgtsPlayer.pause();
            this.mepgtsPlayer.unload();
            this.mepgtsPlayer.destroy();
            this.mepgtsPlayer = null;
        }

        if (this.captionRenderer !== null) {
            this.captionRenderer.detachMedia();
            this.captionRenderer.dispose();
            this.captionRenderer = null;
        }

        if (this.superimposeRenderer !== null) {
            this.superimposeRenderer.detachMedia();
            this.superimposeRenderer.dispose();
            this.superimposeRenderer = null;
        }

        super.beforeDestroy();
    }

    /**
     * video 再生初期設定
     */
    protected initVideoSetting(): void {
        // 対応しているか確認
        if (Mpegts.isSupported() === false || Mpegts.getFeatureList().mseLivePlayback === false) {
            this.snackbarState.open({
                color: 'error',
                text: '非対応ブラウザーです。',
            });

            throw new Error('UnsupportedBrowser');
        }

        if (this.video === null) {
            this.snackbarState.open({
                color: 'error',
                text: 'video 要素がありません。',
            });
            throw new Error('VideoIsNull');
        }

        // mpegts.js の設定
        Mpegts.LoggingControl.enableVerbose = true;
        const mpegtsConfig: Mpegts.Config = {
            enableWorker: true,
            liveBufferLatencyChasing: true,
            liveBufferLatencyMinRemain: 1.0,
            liveBufferLatencyMaxLatency: 2.0,
        };
        this.mepgtsPlayer = Mpegts.createPlayer(
            {
                type: 'mse',
                isLive: true,
                url: this.videoSrc,
            },
            mpegtsConfig,
        );

        this.mepgtsPlayer.attachMediaElement(this.video);
        this.mepgtsPlayer.load();
        this.mepgtsPlayer.play();

        // 字幕対応
        const captionOption = SubtitleUtil.getAribb24BaseOption();
        captionOption.data_identifer = 0x80;
        this.captionRenderer = new aribb24js.CanvasB24Renderer(captionOption);

        const superimposeOption = SubtitleUtil.getAribb24BaseOption();
        superimposeOption.data_identifer = 0x81;
        this.superimposeRenderer = new aribb24js.CanvasB24Renderer(superimposeOption);

        this.captionRenderer.attachMedia(this.video);
        this.superimposeRenderer.attachMedia(this.video);

        /**
         * 字幕スーパー用の処理
         * 元のソースは下記参照
         * https://twitter.com/magicxqq/status/1381813912539066373
         * https://github.com/l3tnun/EPGStation/commit/352bf9a69fdd0848295afb91859e1a402b623212#commitcomment-50407815
         */
        this.mepgtsPlayer.on(Mpegts.Events.PES_PRIVATE_DATA_ARRIVED, data => {
            if (data.stream_id === 0xbd && data.data[0] === 0x80 && this.captionRenderer !== null) {
                // private_stream_1, caption
                this.captionRenderer.pushData(data.pid, data.data, data.pts / 1000);
            } else if (data.stream_id === 0xbf && this.superimposeRenderer !== null) {
                // private_stream_2, superimpose
                let payload = data.data;
                if (payload[0] !== 0x81) {
                    payload = this.parseMalformedPES(data.data);
                }
                if (payload[0] !== 0x81) {
                    return;
                }
                this.superimposeRenderer.pushData(data.pid, payload, data.nearest_pts / 1000);
            }
        });
    }

    /**
     * 字幕スーパー用の処理
     * 元のソースは下記参照
     * https://twitter.com/magicxqq/status/1381813912539066373
     * https://github.com/l3tnun/EPGStation/commit/352bf9a69fdd0848295afb91859e1a402b623212#commitcomment-50407815
     */
    private parseMalformedPES(data: any): any {
        let pes_scrambling_control = (data[0] & 0x30) >>> 4;
        let pts_dts_flags = (data[1] & 0xc0) >>> 6;
        let pes_header_data_length = data[2];

        let payload_start_index = 3 + pes_header_data_length;
        let payload_length = data.byteLength - payload_start_index;

        let payload = data.subarray(payload_start_index, payload_start_index + payload_length);

        return payload;
    }

    /**
     * 動画の長さを返す (秒)
     * @return number
     */
    public getDuration(): number {
        return 0;
    }

    /**
     * 動画の現在再生位置を返す (秒)
     * @return number
     */
    public getCurrentTime(): number {
        return 0;
    }

    /**
     * 再生位置設定
     * @param time: number (秒)
     */
    public setCurrentTime(time: number): void {
        return;
    }

    /**
     * 字幕を表示させる
     */
    public showSubtitle(): void {
        super.showSubtitle();
        if (this.captionRenderer !== null) {
            this.captionRenderer.show();
        }

        if (this.superimposeRenderer !== null) {
            this.superimposeRenderer.show();
        }
    }

    /**
     * 字幕を非表示にする
     */
    public disabledSubtitle(): void {
        super.disabledSubtitle();

        if (this.captionRenderer !== null) {
            this.captionRenderer.hide();
        }

        if (this.superimposeRenderer !== null) {
            this.superimposeRenderer.hide();
        }
    }
}
</script>
