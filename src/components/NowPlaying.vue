<template>
  <div id="app">
    <!-- NOW PLAYING (Active track) -->
    <div
      v-if="player && player.trackTitle"
      class="now-playing now-playing--active"
      :style="{ backgroundImage: 'url(' + player.trackAlbum.image + ')' }"
    >
      <div class="background-overlay"></div>

      <!-- Play/Pause indicator -->
      <div class="now-playing__status-container">
        <div class="now-playing__status">
          <span class="now-playing__status-text">
            {{ playerResponse.is_playing ? '▶' : '❚ ❚' }}
          </span>
        </div>
      </div>

      <!-- Title + Artist -->
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="formattedTrackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>

    <!-- IDLE (No music playing) -->
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
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    },
    formattedTrackTitle() {
      return this.player?.trackTitle || ''
    }
  },
  watch: {
    player: {
      handler(newPlayer) {
        if (newPlayer && newPlayer.trackTitle) {
          if (this.imageCycleInterval) {
            clearInterval(this.imageCycleInterval)
            this.imageCycleInterval = null
          }
        } else {
          if (!this.imageCycleInterval) this.startImageCycle()
        }
      },
      deep: true
    },
    auth(newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },
    playerResponse() {
      this.handleNowPlaying()
    },
    playerData() {
      this.getAlbumColours()
    }
  },
  mounted() {
    this.setDataInterval()
    if (!(this.player && this.player.trackTitle)) {
      this.startImageCycle()
    }
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
          this.playerData = data
          this.$emit('spotifyTrackUpdated', data)
          return
        }
        data = await response.json()
        this.playerResponse = data
      } catch (error) {
        console.error('getNowPlaying error:', error)
        this.handleExpiredToken()
        data = this.getEmptyPlayer()
        this.playerData = data
        this.$emit('spotifyTrackUpdated', data)
      }
    },
    getAlbumColours() {
      if (!this.player?.trackAlbum?.image) return
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => this.handleAlbumPalette(palette))
        .catch(err => console.error('Vibrant error:', err))
    },
    getEmptyPlayer() {
      return { trackAlbum: {}, trackArtists: [], trackId: '', trackTitle: '' }
    },
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => this.getNowPlaying(), 2500)
    },
    setAppColours() {
      document.documentElement.style.setProperty('--color-text-primary', this.colourPalette.text)
      document.documentElement.style.setProperty('--colour-background-now-playing', this.colourPalette.background)
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
      const newTrackData = {
        trackArtists: this.playerResponse.item.artists.map(a => a.name),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0]?.url || ''
        }
      }
      if (newTrackData.trackId !== this.playerData.trackId) {
        this.playerData = newTrackData
        this.$emit('spotifyTrackUpdated', this.playerData)
      }
    },
    async refreshAccessToken() {
      // ... your existing token refresh logic (unchanged)
    },
    async handleExpiredToken() {
      clearInterval(this.pollPlaying)
      const refreshed = await this.refreshAccessToken()
      if (!refreshed) this.$emit('authFailed')
    },
    hexToHSL(hex) {
      let r = 0, g = 0, b = 0
      if (hex.length === 7) {
        r = parseInt(hex.slice(1, 3), 16) / 255
        g = parseInt(hex.slice(3, 5), 16) / 255
        b = parseInt(hex.slice(5, 7), 16) / 255
      }
      const max = Math.max(r, g, b), min = Math.min(r, g, b)
      let l = (max + min) / 2
      return l * 100
    },
    handleAlbumPalette(palette) {
      const albumColours = Object.values(palette)
        .filter(Boolean)
        .map(c => ({ background: c.getHex() }))
      this.swatches = albumColours
      this.colourPalette = albumColours[Math.floor(Math.random() * albumColours.length)] || { background: '#000000' }
      const lightness = this.hexToHSL(this.colourPalette.background)
      this.colourPalette.text = lightness > 50 ? '#000000' : '#ffffff'
      this.setAppColours()
    }
  }
}
</script>

<style scoped>
html, body {
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

/* Active: Full-screen album art + blur */
.now-playing--active {
  width: 100%;
  height: 100%;
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  position: relative;
  overflow: hidden;
}

.background-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.45);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  z-index: 1;
}

/* Play/Pause icon (top) */
.now-playing__status-container {
  position: absolute;
  top: 40px;
  left: 0;
  width: 100%;
  text-align: center;
  z-index: 10;
}
.now-playing__status-text {
  font-size: 80px;
  color: white;
  text-shadow: 0 2px 10px rgba(0,0,0,0.8);
}

/* Title & Artist (bottom area) */
.now-playing__details {
  position: absolute;
  bottom: 100px;
  left: 50%;
  transform: translateX(-50%);
  width: 90%;
  text-align: center;
  z-index: 10;
  padding: 0 40px;
  box-sizing: border-box;
}
.now-playing__track {
  font-size: 68px;
  font-weight: 700;
  line-height: 1.2;
  margin: 0 0 20px 0;
  color: white;
  text-shadow: 0 4px 20px rgba(0,0,0,0.9);
  word-wrap: break-word;
}
.now-playing__artists {
  font-size: 48px;
  font-weight: 400;
  line-height: 1.3;
  margin: 0;
  color: rgba(255,255,255,0.95);
  text-shadow: 0 3px 15px rgba(0,0,0,0.8);
}

/* Idle state */
.now-playing--idle {
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
}
.idle-image-container {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0; left: 0;
}
.now-playing__idle-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  position: absolute;
  top: 0; left: 0;
}

/* Fade transition */
.fade-enter-active, .fade-leave-active { transition: opacity 2s ease-in-out; }
.fade-enter, .fade-leave-to { opacity: 0; }
</style>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
