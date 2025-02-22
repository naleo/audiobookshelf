<template>
  <modals-modal v-model="show" name="rss-feed-modal" :width="600" :height="'unset'" :processing="processing">
    <template #outer>
      <div class="absolute top-0 left-0 p-5 w-2/3 overflow-hidden">
        <p class="font-book text-3xl text-white truncate">{{ title }}</p>
      </div>
    </template>
    <div ref="wrapper" class="px-8 py-6 w-full text-sm rounded-lg bg-bg shadow-lg border border-black-300 relative overflow-hidden">
      <div v-if="currentFeed" class="w-full">
        <p class="text-lg font-semibold mb-4">{{ $strings.HeaderRSSFeedIsOpen }}</p>

        <div class="w-full relative">
          <ui-text-input v-model="currentFeed.feedUrl" readonly />

          <span class="material-icons absolute right-2 bottom-2 p-0.5 text-base transition-transform duration-100 text-gray-300 hover:text-white transform hover:scale-125 cursor-pointer" @click="copyToClipboard(currentFeed.feedUrl)">content_copy</span>
        </div>
      </div>
      <div v-else class="w-full">
        <p class="text-lg font-semibold mb-4">{{ $strings.HeaderOpenRSSFeed }}</p>

        <div class="w-full relative mb-2">
          <ui-text-input-with-label v-model="newFeedSlug" :label="$strings.LabelRSSFeedSlug" />
          <p class="text-xs text-gray-400 py-0.5 px-1">{{ $getString('MessageFeedURLWillBe', [demoFeedUrl]) }}</p>
        </div>

        <p v-if="isHttp" class="w-full pt-2 text-warning text-xs">{{ $strings.NoteRSSFeedPodcastAppsHttps }}</p>
      </div>
      <div v-show="userIsAdminOrUp" class="flex items-center pt-6">
        <div class="flex-grow" />
        <ui-btn v-if="currentFeed" color="error" small @click="closeFeed">{{ $strings.ButtonCloseFeed }}</ui-btn>
        <ui-btn v-else color="success" small @click="openFeed">{{ $strings.ButtonOpenFeed }}</ui-btn>
      </div>
    </div>
  </modals-modal>
</template>

<script>
export default {
  props: {
    value: Boolean,
    collection: {
      type: Object,
      default: () => null
    },
    feed: {
      type: Object,
      default: () => null
    }
  },
  data() {
    return {
      processing: false,
      newFeedSlug: null,
      currentFeed: null
    }
  },
  watch: {
    show: {
      immediate: true,
      handler(newVal) {
        if (newVal) {
          this.init()
        }
      }
    }
  },
  computed: {
    show: {
      get() {
        return this.value
      },
      set(val) {
        this.$emit('input', val)
      }
    },
    collectionId() {
      return this.collection.id
    },
    title() {
      return this.collection.name
    },
    userIsAdminOrUp() {
      return this.$store.getters['user/getIsAdminOrUp']
    },
    demoFeedUrl() {
      return `${window.origin}/feed/${this.newFeedSlug}`
    },
    isHttp() {
      return window.origin.startsWith('http://')
    }
  },
  methods: {
    openFeed() {
      if (!this.newFeedSlug) {
        this.$toast.error('Must set a feed slug')
        return
      }

      const sanitized = this.$sanitizeSlug(this.newFeedSlug)
      if (this.newFeedSlug !== sanitized) {
        this.newFeedSlug = sanitized
        this.$toast.warning('Slug had to be modified - Run again')
        return
      }

      const payload = {
        serverAddress: window.origin,
        slug: this.newFeedSlug
      }
      if (this.$isDev) payload.serverAddress = `http://localhost:3333${this.$config.routerBasePath}`

      console.log('Payload', payload)
      this.$axios
        .$post(`/api/feeds/collection/${this.collectionId}/open`, payload)
        .then((data) => {
          console.log('Opened RSS Feed', data)
          this.currentFeed = data.feed
        })
        .catch((error) => {
          console.error('Failed to open RSS Feed', error)
          const errorMsg = error.response ? error.response.data : null
          this.$toast.error(errorMsg || 'Failed to open RSS Feed')
        })
    },
    copyToClipboard(str) {
      this.$copyToClipboard(str, this)
    },
    closeFeed() {
      this.processing = true
      this.$axios
        .$post(`/api/feeds/${this.currentFeed.id}/close`)
        .then(() => {
          this.$toast.success(this.$strings.ToastRSSFeedCloseSuccess)
          this.show = false
        })
        .catch((error) => {
          console.error('Failed to close RSS feed', error)
          this.$toast.error(this.$strings.ToastRSSFeedCloseFailed)
        })
        .finally(() => {
          this.processing = false
        })
    },
    init() {
      if (!this.collection) return
      this.newFeedSlug = this.collectionId
      this.currentFeed = this.feed
    }
  },
  mounted() {}
}
</script>
