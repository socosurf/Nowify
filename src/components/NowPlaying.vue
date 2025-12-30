<template>
  <div id="app">
    <!-- NOW PLAYING - Shows if we have a title -->
    <div
      v-if="hasTrack"
      class="now-playing now-playing--active"
      :style="{ backgroundImage: albumArtUrl ? 'url(' + albumArtUrl + ')' : '' }"
    >
      <div class="background-overlay"></div>

      <div class="now-playing__details">
        <h1 class="now-playing__track">{{ playerData.trackTitle }}</h1>
        <h2 class="now-playing__artists">{{ trackArtistsJoined }}</h2>
      </div>

      <!-- DEBUG OVERLAY - Remove this whole div when confirmed working -->
      <div class="debug-info">
        DEBUG: Playing "{{ playerData.trackTitle }}" by {{ trackArtistsJoined }}
      </div>
    </div>

    <!-- IDLE -->
    <div v-else class="now-playing now-playing--idle">
      <transition-group name="fade" tag="div" class="idle-image-container">
        <img
          :key="currentImageIndex"
          :src="idleImages[currentImageIndex]"
          alt="No music playing"
          class="now-playing__idle-image"
        />
      </transition-group>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'
import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',
  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },
  data() {
    return {
      pollPlaying: '',
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: [],
      idleImages: [
        require('@/assets/no-music-1.jpg'),
        require('@/assets/no-music-2.jpg'),
        require('@/assets/no-music-3.jpg'),
        require('@/assets/no-music-4.jpg'),
        require('@/assets/no-music-5.jpg')
      ],
      currentImageIndex: 0,
      imageCycleInterval: null
    }
  },
  computed: {
    hasTrack() {
      return this.playerData && this.playerData.trackTitle && this.playerData.trackTitle.trim() !== ''
    },
    albumArtUrl() {
      return this.playerData?.trackAlbum?.image || ''
    },
    trackArtistsJoined() {
      return this.playerData.trackArtists ? this.playerData.trackArtists.join(', ') : ''
    }
  },
  watch: {
    playerData: {
      handler(newData) {
        if (newData && newData.trackTitle) {
          if (this.imageCycleInterval) {
            clearInterval(this.imageCycleInterval)
            this.imageCycleInterval = null
          }
        } else {
          if (!this.imageCycleInterval) this.startImageCycle()
        }
        this.getAlbumColours()
      },
      deep: true
    },
    playerResponse() {
      this.handleNowPlaying()
    },
    auth(newVal) {
      if (newVal.status === false) clearInterval(this.pollPlaying)
    }
  },
  mounted() {
    this.setDataInterval()
    if (!this.hasTrack) this.startImageCycle()
  },
  beforeDestroy() {
    clearInterval(this.pollPlaying)
    if (this.imageCycleInterval) clearInterval(this.imageCycleInterval)
  },
  methods: {
    startImageCycle() {
      if (this.idleImages.length === 0) return
      this.imageCycleInterval = setInterval(() => {
        this.currentImageIndex = (this.currentImageIndex + 1) % this.idleImages.length
      }, 60000)
    },
    async getNowPlaying() {
      let data = {}
      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          { headers: { Authorization: `Bearer ${this.auth.accessToken}` } }
        )
        if (!response.ok) throw new Error(`Error: ${response.status}`)
        if (response.status === 204) {
          data = this.getEmptyPlayer()
        } else {
          data = await response.json()
        }
        this.playerResponse = data
      } catch (error) {
        this.handleExpiredToken()
        this.playerData = this.getEmptyPlayer()
        this.$emit('spotifyTrackUpdated', this.playerData)
      }
    },
    getAlbumColours() {
      if (!this.albumArtUrl) return
      Vibrant.from(this.albumArtUrl)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => this.handleAlbumPalette(palette))
        .catch(() => {})
    },
    getEmptyPlayer() {
      return { trackAlbum: { image: '' }, trackArtists: [], trackId: '', trackTitle: '' }
    },
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => this.getNowPlaying(), 2500)
    },
    setAppColours() {
      document.documentElement.style.setProperty('--color-text-primary', this.colourPalette.text || '#ffffff')
      document.documentElement.style.setProperty('--colour-background-now-playing', this.colourPalette.background || '#000000')
    },
    handleNowPlaying() {
      if (this.playerResponse.error?.status === 401 || this.playerResponse.error?.status === 400) {
        this.handleExpiredToken()
        return
      }
      if (!this.playerResponse.item) {
        this.playerData = this.getEmptyPlayer()
        this.$emit('spotifyTrackUpdated', this.playerData)
        return
      }
      // ALWAYS update playerData when we have a valid item
      const newTrackData = {
        trackArtists: this.playerResponse.item.artists.map(a => a.name),
        trackTitle: this.playerResponse.item.name || 'Unknown Track',
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0]?.url || ''
        }
      }
      this.playerData = newTrackData
      this.$emit('spotifyTrackUpdated', this.playerData)
    },
    async refreshAccessToken() {
      // ← Your existing refresh logic unchanged
    },
    async handleExpiredToken() {
      clearInterval(this.pollPlaying)
      const refreshed = await this.refreshAccessToken()
      if (!refreshed) this.$emit('authFailed')
    },
    hexToHSL(hex) {
      let r = 0, g = 0, b = 0
      if (hex && hex.length === 7) {
        r = parseInt(hex.slice(1, 3), 16) / 255
        g = parseInt(hex.slice(3, 5), 16) / 255
        b = parseInt(hex.slice(5, 7), 16) / 255
      }
      const max = Math.max(r, g, b), min = Math.min(r, g, b)
      return ((max + min) / 2) * 100
    },
    handleAlbumPalette(palette) {
      const albumColours = Object.values(palette)
        .filter(Boolean)
        .map(c => ({ background: c.getHex() }))
      this.colourPalette = albumColours[Math.floor(Math.random() * albumColours.length)] || { background: '#000000' }
      const lightness = this.hexToHSL(this.colourPalette.background)
      this.colourPalette.text = lightness > 50 ? '#000000' : '#ffffff'
      this.setAppColours()
    }
  }
}
</script>

<style scoped>
/* ← All your previous styles unchanged (container, background-size: contain, blur, centered text, etc.) */
html,
body {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  background: #000;
}

#app {
  width: 1024px;
  height: 768px;
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  overflow: hidden;
  background: #000;
  font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
}

.now-playing--active {
  width: 100%;
  height: 100%;
  background-size: contain;
  background-position: center;
  background-repeat: no-repeat;
  background-color: #000;
  position: relative;
  overflow: hidden;
}

.background-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.65);
  backdrop-filter: blur(32px);
  -webkit-backdrop-filter: blur(32px);
  z-index: 1;
}

.now-playing__details {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  width: 90%;
  max-width: 940px;
  z-index: 10;
  padding: 0 40px;
  box-sizing: border-box;
}

.now-playing__track {
  font-size: 84px;
  font-weight: 700;
  line-height: 1.15;
  margin: 0 0 24px 0;
  color: white;
  text-shadow: 0 4px 30px rgba(0, 0, 0, 0.95);
  word-wrap: break-word;
}

.now-playing__artists {
  font-size: 58px;
  font-weight: 400;
  line-height: 1.3;
  margin: 0;
  color: rgba(255, 255, 255, 0.95);
  text-shadow: 0 3px 20px rgba(0, 0, 0, 0.9);
}

.debug-info {
  position: absolute;
  bottom: 20px;
  left: 20px;
  font-size: 20px;
  color: yellow;
  background: rgba(0,0,0,0.6);
  padding: 10px 15px;
  border-radius: 8px;
  z-index: 20;
  max-width: 90%;
  word-wrap: break-word;
}

.now-playing--idle,
.idle-image-container,
.now-playing__idle-image {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
  object-fit: cover;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 2s ease-in-out;
}

.fade-enter,
.fade-leave-to {
  opacity: 0;
}
</style>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
