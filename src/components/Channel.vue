<template>
    <ErrorHandler v-if="channel && channel.error" :message="channel.message" :error="channel.error" />

    <div v-if="channel" v-show="!channel.error">
        <h1 class="uk-text-center">
            <img height="48" width="48" class="uk-border-circle" v-bind:src="channel.avatarUrl" />{{ channel.name }}
        </h1>
        <img v-if="channel.bannerUrl" v-bind:src="channel.bannerUrl" style="width: 100%" loading="lazy" />
        <p style="white-space: pre-wrap"><span v-html="purifyHTML(urlify(channel.description))"></span></p>

        <button
            v-if="authenticated"
            @click="subscribeHandler"
            class="uk-button uk-button-small"
            style="background: #222"
            type="button"
        >
            {{ subscribed ? $t("actions.unsubscribe") : $t("actions.subscribe") }}
        </button>

        <hr />

        <div class="uk-grid-xl" uk-grid="parallax: 0">
            <div
                class="uk-width-1-2 uk-width-1-3@m uk-width-1-4@l uk-width-1-5@xl"
                v-bind:key="video.url"
                v-for="video in this.channel.relatedStreams"
            >
                <VideoItem :video="video" height="94" width="168" hideChannel />
            </div>
        </div>
    </div>
</template>

<script>
import ErrorHandler from "@/components/ErrorHandler.vue";
import VideoItem from "@/components/VideoItem.vue";

export default {
    data() {
        return {
            channel: null,
            subscribed: false,
        };
    },
    mounted() {
        this.getChannelData();
    },
    activated() {
        if (this.channel && !this.channel.error) document.title = this.channel.name + " - Piped";
        window.addEventListener("scroll", this.handleScroll);
        if (this.channel && !this.channel.error) this.updateWatched(this.channel.relatedStreams);
    },
    deactivated() {
        window.removeEventListener("scroll", this.handleScroll);
    },
    unmounted() {
        window.removeEventListener("scroll", this.handleScroll);
    },
    methods: {
        async fetchSubscribedStatus() {
            this.fetchJson(
                this.apiUrl() + "/subscribed",
                {
                    channelId: this.channel.id,
                },
                {
                    headers: {
                        Authorization: this.getAuthToken(),
                    },
                },
            ).then(json => {
                this.subscribed = json.subscribed;
            });
        },
        async fetchChannel() {
            const url = this.apiUrl() + "/" + this.$route.params.path + "/" + this.$route.params.channelId;
            return await this.fetchJson(url);
        },
        async getChannelData() {
            this.fetchChannel()
                .then(data => (this.channel = data))
                .then(() => {
                    if (!this.channel.error) {
                        document.title = this.channel.name + " - Piped";
                        if (this.authenticated) this.fetchSubscribedStatus();
                        this.updateWatched(this.channel.relatedStreams);
                    }
                });
        },
        handleScroll() {
            if (this.loading || !this.channel || !this.channel.nextpage) return;
            if (window.innerHeight + window.scrollY >= document.body.offsetHeight - window.innerHeight) {
                this.loading = true;
                this.fetchJson(this.apiUrl() + "/nextpage/channel/" + this.channel.id, {
                    nextpage: this.channel.nextpage,
                }).then(json => {
                    this.channel.nextpage = json.nextpage;
                    this.loading = false;
                    this.updateWatched(json.relatedStreams);
                    json.relatedStreams.map(stream => this.channel.relatedStreams.push(stream));
                });
            }
        },
        subscribeHandler() {
            this.fetchJson(this.apiUrl() + (this.subscribed ? "/unsubscribe" : "/subscribe"), null, {
                method: "POST",
                body: JSON.stringify({
                    channelId: this.channel.id,
                }),
                headers: {
                    Authorization: this.getAuthToken(),
                    "Content-Type": "application/json",
                },
            });
            this.subscribed = !this.subscribed;
        },
    },
    components: {
        ErrorHandler,
        VideoItem,
    },
};
</script>
