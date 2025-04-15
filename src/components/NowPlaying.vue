<template>
  <div id="app">
    <div
      v-if="player && player.trackTitle"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__status-container">
        <div class="now-playing__status">
          <h3 class="now-playing__status-text">
            {{ playerResponse.is_playing ? 'Now Playing' : 'Currently Paused' }}
          </h3>
        </div>
      </div>
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1
          class="now-playing__track"
          v-text="formattedTrackTitle"
        ></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>
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
            console.log('Stopped image cycling')
          }
        } else {
          if (!this.imageCycleInterval) {
            this.startImageCycle()
          }
        }
      },
      deep: true
    },

    auth(newVal) {
      if (newVal.status === false) {
        console.log('Auth invalid, stopping poll')
        clearInterval(this.pollPlaying)
      }
    },

    playerResponse() {
      this.handleNowPlaying()
    },

    playerData() {
      console.log('playerData changed:', this.playerData)
      this.getAlbumColours()
    }
  },

  mounted() {
    console.log('NowPlaying mounted')
    this.setDataInterval()
    if (!(this.player && this.player.trackTitle)) {
      this.startImageCycle()
    }
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
    if (this.imageCycleInterval) {
      clearInterval(this.imageCycleInterval)
    }
  },

  methods: {
    startImageCycle() {
      if (this.idleImages.length === 0) return
      this.imageCycleInterval = setInterval(() => {
        this.currentImageIndex = (this.currentImageIndex + 1) % this.idleImages.length
        console.log('Cycled to image index:', this.currentImageIndex)
      }, 60000) // Change every 60 seconds
      console.log('Started image cycling')
    },

    async getNowPlaying() {
      let data = {}
      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        if (!response.ok) {
          throw new Error(`An error has occurred: ${response.status}`)
        }

        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data
          console.log('No active device (204)', data)
          this.$emit('spotifyTrackUpdated', data)
          return
        }

        data = await response.json()
        this.playerResponse = data
        console.log('Fetched playerResponse:', data)
      } catch (error) {
        console.error('getNowPlaying error:', error)
        this.handleExpiredToken()
        data = this.getEmptyPlayer()
        this.playerData = data
        this.$emit('spotifyTrackUpdated', data)
      }
    },

    getNowPlayingClass() {
      const playerClass = this.player && this.player.trackTitle ? 'active' : 'idle'
      console.log('getNowPlayingClass:', playerClass)
      return `now-playing--${playerClass}`
    },

    getAlbumColours() {
      if (!this.player?.trackAlbum?.image) {
        console.log('No album image for colors')
        return
      }
      console.log('Extracting colors from:', this.player.trackAlbum.image)
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          console.log('Got palette:', palette)
          this.handleAlbumPalette(palette)
        })
        .catch(err => console.error('Vibrant error:', err))
    },

    getEmptyPlayer() {
      return {
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      }
    },

    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    setAppColours() {
      console.log('Setting colors:', this.colourPalette)
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      )
      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      )
    },

    handleNowPlaying() {
      console.log('handleNowPlaying, playerResponse:', this.playerResponse)
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()
        return
      }

      if (!this.playerResponse.item) {
        this.playerData = this.getEmptyPlayer()
        console.log('No track, set empty player:', this.playerData)
        this.$emit('spotifyTrackUpdated', this.playerData)
        return
      }

      const newTrackData = {
        trackArtists: this.playerResponse.item.artists.map(artist => artist.name),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0]?.url || ''
        }
      }

      if (newTrackData.trackId !== this.playerData.trackId) {
        this.playerData = newTrackData
        console.log('Updated playerData:', this.playerData)
        this.$emit('spotifyTrackUpdated', this.playerData)
      }
    },

    // Convert hex to HSL and return lightness (0-100)
    hexToHSL(hex) {
      // Remove # and convert to RGB
      let r = 0
      let g = 0
      let b = 0
      if (hex.length === 7) {
        r = parseInt(hex.slice(1, 3), 16)
        g = parseInt(hex.slice(3, 5), 16)
        b = parseInt(hex.slice(5, 7), 16)
      }
      r /= 255
      g /= 255
      b /= 255

      // Find min and max for lightness
      const max = Math.max(r, g, b)
      const min = Math.min(r, g, b)
      let l = (max + min) / 2

      // Convert lightness to percentage
      l = l * 100
      return l
    },

    handleAlbumPalette(palette) {
      const albumColours = Object.keys(palette)
        .filter(item => item !== null)
        .map(colour => ({
          background: palette[colour].getHex()
        }))
      this.swatches = albumColours
      this.colourPalette = albumColours[Math.floor(Math.random() * albumColours.length)] || { background: '#000000' }

      // Set text color based on background lightness
      const lightness = this.hexToHSL(this.colourPalette.background)
      console.log('Background lightness:', lightness)
      this.colourPalette.text = lightness > 50 ? '#000000' : '#ffffff'

      this.setAppColours()
    },

    handleExpiredToken() {
      console.log('Handling expired token')
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  }
}
</script>

<style scoped>
/* Styles for full-screen slideshow and text adjustments */
html, body {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

#app {
  width: 1080px;
  height: 1920px;
  min-width: 1080px;
  min-height: 1920px;
  overflow: hidden;
  position: fixed;
  top: 0;
  left: 0;
  margin: 0;
  padding: 0;
  background: transparent;
}

.now-playing {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
  position: relative !important; /* For absolute positioning */
}

.now-playing--idle {
  width: 1080px;
  height: 1920px;
  margin: 0;
  padding: 0;
  background: transparent;
  overflow: hidden;
}

.now-playing__status-container {
  position: absolute;
  top: 0;
  height: 250px; /* From screen top to album art top */
  width: 100%;
}

.now-playing__status {
  position: absolute;
  top: 50%; /* Center in 0–250px */
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  width: 100%;
  z-index: 10;
}

.now-playing__status-text {
  font-size: 3.2rem !important;
  font-weight: 400;
  color: var(--color-text-primary);
}

.now-playing__cover {
  position: absolute;
  top: 250px; /* Upper quarter */
  left: 50%;
  transform: translateX(-50%);
  z-index: 5;
}

.now-playing__image {
  width: 300px; /* Spotify image size */
  height: 300px;
  object-fit: cover;
}

.now-playing__details {
  position: absolute;
  top: 1200px !important; /* Raised from 1350px, clears album art */
  left: 50% !important;
  transform: translateX(-50%) !important;
  text-align: center !important;
  z-index: 15 !important;
  margin: 0 !important;
  padding: 0 !important;
}

.now-playing__track {
  font-size: 6rem !important;
  color: var(--color-text-primary);
  text-shadow: 3px 3px 5px rgba(0, 0, 0, 0.5);
  white-space: normal;
  word-break: break-word;
  line-height: 1.2;
  margin: 0 0 20px 0 !important;
  width: 100%;
}

.now-playing__artists {
  font-size: 4rem !important;
  color: var(--color-text-primary);
  white-space: normal;
  word-break: break-word;
  line-height: 1.2;
  margin: 0 !important;
  width: 100%;
}

.idle-image-container {
  width: 1080px;
  height: 1920px;
  min-width: 1080px;
  min-height: 1920px;
  position: absolute;
  top: 0;
  left: 0;
  margin: 0;
  padding: 0;
  overflow: hidden;
  background: transparent;
}

.now-playing__idle-image {
  width: 1080px;
  height: 1920px;
  min-width: 1080px;
  min-height: 1920px;
  object-fit: cover;
  position: absolute;
  top: 0;
  left: 0;
  margin: 0;
  padding: 0;
  border: none;
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
