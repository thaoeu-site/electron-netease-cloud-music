<template>
    <div class="video-detail">
        <div class="desc">
            <span class="title">{{mv.name}}</span>
            <span class="by">by</span>
            <router-link v-for="ar in mv.creator"
                :key="ar.id"
                class="creator"
                :to="{ name: 'artist', params: { id: ar.id } }"
                tag="a">{{ar.name}}</router-link>
        </div>
        <video ref="videoEl"
            :src="mvSrc"
            :poster="mvPoster"
            preload="none"
            controls></video>
        <div class="actions">
            <mu-button flat
                small
                @click="handleLike">
                <mu-icon left
                    :color="threadInfo.liked ? 'primary': ''"
                    value="thumb_up"></mu-icon>
                <span>{{btnLikeText}}</span>
            </mu-button>
            <mu-button flat
                small
                @click="handleSubscribe">
                <mu-icon left
                    :value="shouldSub ? 'star': 'star_border'"
                    :color="shouldSub ? 'amber': ''"></mu-icon>
                <span>{{btnFavText}}</span>
            </mu-button>
            <mu-button flat
                small
                :to="{ name: 'comment', params: { type: 'mv', id: mv.id } }">
                <mu-icon left
                    value="comment"></mu-icon>
                <span>{{btnCommentText}}</span>
            </mu-button>
        </div>
    </div>
</template>

<script>
import { mapActions, mapState } from 'vuex';

import Api from '@/api/ipc';
import { sizeImg, HiDpiPx } from '@/util/image';

export default {
    props: {
        mv: {
            required: true
        }
    },
    data() {
        return {
            internalShouldSub: null,
            subCntOffset: 0,
            threadInfo: {
                commentCount: '...',
                liked: false,
                likedCount: '...',
                shareCount: '...'
            }
        };
    },
    computed: {
        ...mapState(['user']),
        mvPoster() {
            return sizeImg(this.mv.picUrl, HiDpiPx(720), HiDpiPx(405));
        },
        mvSrc() {
            const src = Object.entries(this.mv.brs).sort((a, b) => b[0] - a[0])[0][1];
            return src.replace(/^http:/, 'https:');
        },
        btnLikeText() {
            const t = this.threadInfo.liked ? '已赞' : '赞';
            return `${t} (${this.threadInfo.likedCount})`;
        },
        shouldSub() {
            if (this.internalShouldSub === null) return this.mv.subed;
            return this.internalShouldSub;
        },
        btnFavText() {
            const t = this.shouldSub ? '已收藏' : '收藏';
            const n = this.mv.subCount + this.subCntOffset;
            return `${t} (${n})`;
        },
        btnCommentText() {
            return `评论 (${this.threadInfo.commentCount})`;
        }
    },
    methods: {
        ...mapActions([
            'likeResource',
            'unlikeResource',
            'subscribeVideo',
            'unsubscribeVideo'
        ]),
        refreshThreadInfo() {
            Api.getCommentThreadInfoE(this.mv.commentThreadId)
                .then(res => this.threadInfo = res);
        },
        async handleLike() {
            if (!this.user.loginValid) {
                this.$toast.message('汝还没有登录呀      (눈‸눈)');
                return;
            }
            try {
                if (this.threadInfo.liked) {
                    await this.unlikeResource(this.mv.commentThreadId);
                    this.threadInfo.liked = false;
                    this.threadInfo.likedCount -= 1;
                } else {
                    await this.likeResource(this.mv.commentThreadId);
                    this.threadInfo.liked = true;
                    this.threadInfo.likedCount += 1;
                }
                setTimeout(() => this.refreshThreadInfo(), 1000);
            } catch (e) {
                this.$toast.message(e.message);
            }
        },
        async handleSubscribe() {
            if (!this.user.loginValid) {
                this.$toast.message('汝还没有登录呀      (눈‸눈)');
                return;
            }
            try {
                if (this.shouldSub) {
                    await this.unsubscribeVideo(this.mv);
                    this.internalShouldSub = false;
                    this.subCntOffset -= 1;
                } else {
                    await this.subscribeVideo(this.mv);
                    this.internalShouldSub = true;
                    this.subCntOffset += 1;
                }
            } catch (e) {
                this.$toast.message(e.message);
            }
        }
    },
    mounted() {
        this.refreshThreadInfo();
    }
};
</script>
