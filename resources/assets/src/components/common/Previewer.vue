<template>
    <div class="box box-default">
        <div class="box-header with-border">
            <h3 class="box-title" style="width: 100%;">
                <span v-html="$t(title)"></span>
                <span data-toggle="tooltip" class="badge bg-light-blue">{{ indicator }}</span>
                <div class="operations">
                    <i
                        @click="toggleRun"
                        data-toggle="tooltip"
                        data-placement="bottom"
                        :title="$t('general.walk') + ' / ' + $t('general.run')"
                        class="fas fa-forward"
                    ></i>
                    <i
                        @click="toggleRotate"
                        data-toggle="tooltip"
                        data-placement="bottom"
                        :title="$t('general.rotation')"
                        class="fas fa-redo-alt"
                    ></i>
                    <i
                        @click="togglePause"
                        data-toggle="tooltip"
                        data-placement="bottom"
                        :title="$t('general.pause')"
                        class="fas"
                        :class="{ 'fa-pause': !paused, 'fa-play': paused }"
                    ></i>
                    <i
                        @click="reset"
                        data-toggle="tooltip"
                        data-placement="bottom"
                        :title="$t('general.reset')"
                        class="fas fa-stop"
                    ></i>
                </div>
            </h3>
        </div><!-- /.box-header -->
        <div class="box-body">
            <div class="previewer-3d" ref="previewer">
                <!-- Container for 3D Preview -->
            </div>
        </div><!-- /.box-body -->
        <div v-if="$slots.footer" class="box-footer">
            <slot name="footer"></slot>
        </div>
    </div>
</template>

<script>
import * as skinview3d from 'skinview3d';
import { emit } from '../../js/event';
import SkinSteve from '../../images/textures/steve.png';

export default {
    name: 'Previewer',
    props: {
        skin: String,
        cape: String,
        closetMode: Boolean,
        title: {
            default: 'general.texturePreview'
        },
        initPositionZ: {
            default: 70
        }
    },
    data: () => ({
        /** @type {skinview3d.SkinViewer} */
        viewer: null,
        handles: {
            walk: null,
            run: null,
            rotate: null,
        },
        control: null,
        paused: false,
    }),
    computed: {
        indicator() {
            if (!this.closetMode) {
                return '';
            }

            if (this.skin && this.cape) {
                return `${this.$t('general.skin')} & ${this.$t('general.cape')}`;
            } else if (this.skin) {
                return this.$t('general.skin');
            } else if (this.cape) {
                return this.$t('general.cape');
            } else {
                return '';
            }
        }
    },
    watch: {
        skin(url) {
            this.viewer.skinUrl = url || SkinSteve;
        },
        cape(url) {
            this.viewer.capeUrl = url;
        }
    },
    mounted() {
        this.initPreviewer();
        emit('skinViewerMounted', this.$refs.previewer);
    },
    beforeDestroy() {
        this.viewer.dispose();
    },
    methods: {
        initPreviewer() {
            this.viewer = new skinview3d.SkinViewer({
                domElement: this.$refs.previewer,
                width: this.$refs.previewer.clientWidth,
                height: this.$refs.previewer.clientHeight,
                skinUrl: this.skin || SkinSteve,
                capeUrl: this.cape
            });
            this.viewer.camera.position.z = this.initPositionZ;
            this.viewer.animation = new skinview3d.CompositeAnimation();
            this.handles.walk = this.viewer.animation.add(skinview3d.WalkingAnimation);
            this.handles.run = this.viewer.animation.add(skinview3d.RunningAnimation);
            this.handles.rotate = this.viewer.animation.add(skinview3d.RotatingAnimation);
            this.handles.run.paused = true;
            this.control = skinview3d.createOrbitControls(this.viewer);
        },
        togglePause() {
            this.paused = !this.paused;
            this.viewer.animationPaused = !this.viewer.animationPaused;
        },
        toggleRun() {
            this.handles.run.paused = !this.handles.run.paused;
            this.handles.walk.paused = !this.handles.walk.paused;
        },
        toggleRotate() {
            this.handles.rotate.paused = !this.handles.rotate.paused;
        },
        reset() {
            this.viewer.dispose();
            this.handles = {};
            this.control = null;
            this.initPreviewer();
            this.handles.walk.paused = true;
            this.handles.run.paused = true;
            this.handles.rotate.paused = true;
            this.viewer.camera.position.z = 70;
        }
    },
};
</script>

<style lang="stylus">
@media (min-width: 992px) {
    .previewer-3d {
        min-height: 500px;
    }
}

.previewer-3d canvas {
    cursor: move;
}

.operations {
    display: inline;
    float: right;

    i {
        padding: .5em .5em;
        display: inline;
    }

    i:hover {
        color: #555;
        cursor: pointer;
    }
}
</style>
