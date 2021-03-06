<template>
    <div class="dsb-item"
         @click="singleClick"
         @dblclick="doubleClick"
         @mouseover="showTooltip($refs[`downloads-${download.downloadItem.id}`], download)"
         @mouseleave="hideTooltip"
         @contextmenu.prevent="showContextMenu"
         :ref="`downloads-${download.downloadItem.id}`"
         :class="[`dsb-theme-${options.theme}`, progressClass]"
    >
        <div v-if="isInProgress" class="dsb-progress-bar" :style="`width: ${percentDone}%`"></div>

        <div class="dsb-text-container">
            <div class="dsb-filename">{{ filename }}</div>
            <div class="dsb-download-info" v-if="!download.isCancelled() && options.showInfoText">
                <div class="dsb-progress">{{ progress }}</div>
                <div class="dsb-speed" v-if="isInProgress && !download.downloadItem.paused">{{ downloadSpeed }}</div>
                <div class="dsb-percent" v-else>{{ percentDone }}%</div>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
    import Vue from 'vue';
    import * as helpers from '../helpers';

    export default Vue.extend({
        name: 'download',
        props: {
            download: Object,
            options: Object,
        },
        data() {
            return {}
        },
        computed: {
            filename(): string {
                const maxLength = 15;
                const length = this.download.filename().length;
                let filename = (this.download.filename()).slice(0, maxLength);

                if (length > maxLength) {
                    filename += '…';
                }

                return filename;
            },
            status(): string {
                return this.download.status();
            },

            progress(): string {
                const downloadItem = this.download.downloadItem;
                const downloaded = helpers.formatFileSize(downloadItem.bytesReceived);

                if (downloadItem.state === 'complete') {
                    return `${helpers.formatFileSize(downloadItem.fileSize)}`;
                }

                return `${downloaded}`;
            },

            percentDone(): string {
                return this.download.percentDownloaded()
            },

            isInProgress(): boolean {
                return this.download.downloadItem.state === 'in_progress' || this.download.downloadItem.paused;
            },

            progressClass() {
                if (this.download.downloadItem.state === 'complete') {
                    return 'dsb-complete';
                }

                if (this.download.isCancelled()) {
                    return 'dsb-cancelled';
                }

                if (this.download.downloadItem.error && !this.download.downloadItem.paused) {
                    return 'dsb-error';
                }

                return 'dsb-in-progress';
            },

            downloadSpeed(): string {
                if (!this.download) {
                    return '';
                }

                return `${helpers.formatFileSize(this.download.calculateDownloadSpeed(), true)}/s`;
            },
        },

        methods: {
            singleClick() {
                this.$root.$contextMenu.close();
            },
            doubleClick() {
                this.$root.$emit('showDownload', this.download);
            },
            showContextMenu(event: MouseEvent) {
                const userAgent = window.navigator.userAgent;
                const inProgress = this.download.downloadItem.state === 'in_progress';
                const paused = this.download.downloadItem.state === 'interrupted' && this.download.downloadItem.paused;

                let showTitle = userAgent.includes('Windows') ? 'Show in Explorer' :
                    userAgent.includes('Mac') ? 'Reveal in Finder' : 'Show in Folder';

                let items = [
                    {
                        name: 'Clear Download',
                        icon: 'eye-slash',
                        clicked: () => {
                            this.$root.$emit('clearDownload', this.download);
                            this.$root.$contextMenu.close();
                        },
                    },
                ];

                if (!this.download.downloadItem.error) {
                    items.push({
                        name: showTitle,
                        icon: 'folder-open',
                        clicked: () => {
                            this.$root.$emit('showDownload', this.download);
                            this.$root.$contextMenu.close();
                        },
                    });
                }

                // In progress or paused
                if (inProgress || paused) {
                    items.push({
                        name: 'Cancel Download',
                        icon: 'times',
                        clicked: () => {
                            this.$root.$emit('cancelDownload', this.download);
                            this.$root.$contextMenu.close();
                        },
                    });
                }

                // Download in progress
                if (inProgress) {
                    items.push({
                        name: 'Pause Download',
                        icon: 'pause',
                        clicked: () => {
                            this.$root.$emit('pauseDownload', this.download);
                            this.$root.$contextMenu.close();
                        },
                    });
                }

                // Download is paused
                if (paused) {
                    items.push({
                        name: 'Resume Download',
                        icon: 'play',
                        clicked: () => {
                            this.$root.$emit('resumeDownload', this.download);
                            this.$root.$contextMenu.close();
                        },
                    });
                }

                if (this.download.downloadItem.state === 'complete') {
                    items.push({
                        name: 'Delete Download',
                        icon: 'trash-o',
                        clicked: () => {
                            this.$root.$emit('deleteDownload', this.download);
                            this.$root.$contextMenu.close();
                        },
                    });
                }

                this.$root.$contextMenu.open(items, {x: event.clientX, y: event.clientY});
            },
        },
    });
</script>

<style lang="scss" scoped>
    @import "../scss/variables";
    @import "../scss/mixins";
    @import "~bootstrap/scss/bootstrap-reboot";

    .dsb-item {
        background      : light-theme("download");
        border-radius   : 0;
        box-sizing      : border-box;
        color           : light-theme("text");
        cursor          : pointer;
        display         : flex;
        flex            : 0 1 auto;
        flex-direction  : column;
        font            : 400 normal 14px/1 Arial, sans-serif;
        justify-content : center;
        letter-spacing  : normal;
        margin          : 5px 5px 0 5px;
        max-width       : unset;
        min-height      : 30px;
        min-width       : unset;
        overflow        : hidden;
        padding         : 3px 6px;
        position        : relative;
        text-shadow     : none;
        width           : auto;

        * {
            position : relative;
            z-index  : 10;
        }

        &.dsb-complete {
            background : light-theme("progress");
        }

        &.dsb-error {
            background : light-theme("error");
        }

        &.dsb-theme-dark {
            background : dark-theme("download");
            color      : dark-theme("text");

            .dsb-progress-bar {
                background : dark-theme("progress");
            }

            &.dsb-complete {
                background : dark-theme("progress");
            }

            &.dsb-error {
                background : dark-theme("error");
            }

            .dsb-text-line {
                color : dark-theme("text");
            }
        }
    }

    .dsb-progress-bar {
        background    : light-theme("progress");
        border-radius : 0;
        display       : block;
        height        : 100%;
        left          : 0;
        position      : absolute;
        top           : 0;
        width         : 100%;
        z-index       : 0;
    }

    .dsb-text-container {
        display        : flex;
        flex-direction : row;
        align-items    : center;
        height         : 100%;
    }

    .dsb-text-line {
        background     : transparent;
        box-sizing     : content-box;
        color          : light-theme("text");
        display        : block;
        font           : 400 normal 10px/10px Arial, Helvetica, sans-serif;
        height         : auto;
        letter-spacing : normal;
        margin         : 0;
        padding        : 0;
        text-align     : right;
        position       : relative;

        + .dsb-text-line {
            margin-top : 2px
        }
    }

    .dsb-filename {
        @extend .dsb-text-line;
        text-align : left;
        font-size  : 14px;
    }

    .dsb-download-info {
        height : 100%;
        margin : 0 0 0 10px;
    }

    .dsb-progress {
        @extend .dsb-text-line;
    }

    .dsb-speed, .dsb-percent {
        @extend .dsb-text-line;
    }
</style>