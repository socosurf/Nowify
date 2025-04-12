<template>
  <div id="app">
    <div
      v-if="player && player.trackTitle"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__status">
        <h3 class="now-playing__status-text">
          {{ playerResponse.is_playing ? 'Now Playing' : 'Currently Paused' }}
        </h3>
      </div>
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>
    <div v-else class="now-playing now-playing--idle">
      <img
        src="@/assets/no-music.jpg"
        alt="No music playing"
        class="now-playing__idle-image"
      />
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
      swatches: []
    }
  },

  computed: {
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    console.log('NowPlaying mounted'); // Debug
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
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
          console.log('No active device (204)', data); // Debug
          this.$emit('spotifyTrackUpdated', data)
          return
        }

        data = await response.json()
        this.playerResponse = data
        console.log('Fetched playerResponse:', data); // Debug
      } catch (error) {
        console.error('getNowPlaying error:', error); // Debug
        this.handleExpiredToken()
        data = this.getEmptyPlayer()
        this.playerData = data
        this.$emit('spotifyTrackUpdated', data)
      }
    },

    getNowPlayingClass() {
      const playerClass = this.player && this.player.trackTitle ? 'active' : 'idle'
      console.log('getNowPlayingClass:', playerClass); // Debug
      return `now-playing--${playerClass}`
    },

    getAlbumColours() {
      if (!this.player?.trackAlbum?.image) {
        console.log('No album image for colors'); // Debug
        return
      }
      console.log('Extracting colors from:', this.player.trackAlbum.image); // Debug
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          console.log('Got palette:', palette); // Debug
          this.handleAlbumPalette(palette)
        })
        .catch(err => console.error('Vibrant error:', err)); // Debug
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
      console.log('Setting colors:', this.colourPalette); // Debug
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
      console.log('handleNowPlaying, playerResponse:', this.playerResponse); // Debug
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()
        return
      }

      if (!this.playerResponse.item) {
        this.playerData = this.getEmptyPlayer()
        console.log('No track, set empty player:', this.playerData); // Debug
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

      // Only update if track changed to avoid unnecessary renders
      if (newTrackData.trackId !== this.playerData.trackId) {
        this.playerData = newTrackData
        console.log('Updated playerData:', this.playerData); // Debug
        this.$emit('spotifyTrackUpdated', this.playerData)
      }
    },

    handleAlbumPalette(palette) {
      const albumColours = Object.keys(palette)
        .filter(item => item !== null)
        .map(colour => ({
          text: palette[colour].getTitleTextColor(),
          background: palette[colour].getHex()
        }))
      this.swatches = albumColours
      this.colourPalette = albumColours[Math.floor(Math.random() * albumColours.length)] || { text: '#ffffff', background: '#000000' }
      this.setAppColours()
    },

    handleExpiredToken() {
      console.log('Handling expired token'); // Debug
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },

  watch: {
    auth(newVal) {
      if (newVal.status === false) {
        console.log('Auth invalid, stopping poll'); // Debug
        clearInterval(this.pollPlaying)
      }
    },

    playerResponse() {
      this.handleNowPlaying()
    },

    playerData() {
      console.log('playerData changed:', this.playerData); // Debug
      this.getAlbumColours()
    }
  }
}
</script>

<style scoped>
/* Minimal inline styles to avoid SCSS conflicts */
.now-playing__status {
  text-align: center;
  margin-bottom: 10px;
}

.now-playing__status-text {
  font-size: 1.5rem; /* Adjust if needed after checking SCSS */
  font-weight: 400;
  color: var(--color-text-primary);
}

.now-playing__idle-image {
  width: 1080px;
  height: 1920px;
  object-fit: cover;
  display: block;
  position: absolute;
  top: 0;
  left: 0;
}

/* Rely on now-playing.scss for other styles */
</style>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
