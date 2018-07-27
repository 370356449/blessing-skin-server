<template>
    <div class="item" :class="{ 'item-selected': selected }">
        <div class="item-body" @click="$emit('select')">
            <img :src="previewLink">
        </div>
        <div class="item-footer">
            <p class="texture-name">
                <span :title="name">{{ textureName }} <small>({{ type }})</small></span>
            </p>

            <a :href="linkToSkinlib" :title="$t('user.viewInSkinlib')" class="more" data-toggle="tooltip" data-placement="bottom"><i class="fa fa-share"></i></a>
            <span :title="$t('general.more')" class="more" data-toggle="dropdown" aria-haspopup="true" id="more-button"><i class="fa fa-cog"></i></span>

            <ul class="dropup dropdown-menu" aria-labelledby="more-button">
                <li><a @click="rename" v-t="'user.renameItem'"></a></li>
                <li><a @click="remove" v-t="'user.removeItem'"></a></li>
                <li><a @click="setAsAvatar" v-t="'user.setAsAvatar'"></a></li>
            </ul>
        </div>
    </div>
</template>

<script>
import swal from 'sweetalert2';
import toastr from 'toastr';
import axios from 'axios';

export default {
    name: 'ClosetItem',
    props: {
        tid: {
            type: Number,
            required: true,
        },
        type: {
            type: String,
            validator: value => ['steve', 'alex', 'cape'].includes(value)
        },
        name: {
            type: String,
            required: true
        },
        selected: Boolean
    },
    computed: {
        previewLink() {
            return `${blessing.base_url}/preview/${this.tid}.png`;
        },
        linkToSkinlib() {
            return `${blessing.base_url}/skinlib/show/${this.tid}`;
        }
    },
    data() {
        return {
            textureName: this.name
        };
    },
    methods: {
        async rename() {
            let newTextureName = '';

            try {
                newTextureName = await swal({
                    title: this.$t('user.renameClosetItem'),
                    input: 'text',
                    inputValue: this.textureName,
                    showCancelButton: true,
                    inputValidator: value => (new Promise((resolve, reject) => {
                        value ? resolve() : reject(this.$t('skinlib.emptyNewTextureName'));
                    }))
                });
            } catch (error) {
                return;
            }

            const { data: { errno, msg } } = await axios.post(
                '/user/closet/rename',
                { tid: this.tid, new_name: newTextureName }
            );

            if (errno === 0) {
                this.textureName = newTextureName;
                toastr.success(msg);
            } else {
                toastr.warning(msg);
            }
        },
        async remove() {
            try {
                await swal({
                    text: this.$t('user.removeFromClosetNotice'),
                    type: 'warning',
                    showCancelButton: true
                });
            } catch (error) {
                return;
            }

            const { data: { errno, msg } } = await axios.post(
                '/user/closet/remove',
                { tid: this.tid }
            );

            if (errno === 0) {
                this.$emit('item-removed', this.tid);
                swal({ type: 'success', html: msg });
            } else {
                toastr.warning(msg);
            }
        },
        async setAsAvatar() {
            try {
                await swal({
                    title: this.$t('user.setAvatar'),
                    text: this.$t('user.setAvatarNotice'),
                    type: 'question',
                    showCancelButton: true
                });
            } catch (error) {
                return;
            }

            const { data: { errno, msg } } = await axios.post(
                '/user/profile/avatar',
                { tid: this.tid }
            );

            if (errno === 0) {
                toastr.success(msg);

                // Refresh avatars
                $('[alt="User Image"]').each(function () {
                    $(this).prop('src', $(this).attr('src') + '?' + new Date().getTime());
                });
            } else {
                toastr.warning(msg);
            }
        }
    }
};
</script>