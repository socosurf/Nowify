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
    console.log('NowPlaying component mounted'); // Debug
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
      if (!this.player.trackAlbum?.image) {
        return
      }
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
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
      console.log('handleNowPlaying called, playerResponse:', this.playerResponse); // Debug
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
        return
      }

      this.playerData = {
        trackArtists: this.playerResponse.item.artists.map(artist => artist.name),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
      console.log('Updated playerData:', this.playerData); // Debug
    },

    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => item !== null)
        .map(colour => {
          return {
            text: palette[colour].getTitleTextColor(),
            background: palette[colour].getHex()
          }
        })

      this.swatches = albumColours
      this.colourPalette = albumColours[Math.floor(Math.random() * albumColours.length)]
      this.setAppColours()
    },

    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },

  watch: {
    auth: function(newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    playerResponse: function() {
      this.handleNowPlaying()
    },

    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData)
      this.getAlbumColours()
    }
  }
}
</script>

<style scoped>
#app {
  width: 1080px;
  height: 1920px;
  overflow: hidden;
}

.now-playing {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.now-playing__status {
  text-align: center;
  margin-bottom: 10px;
}

.now-playing__status-text {
  font-size: 1.5rem;
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

.now-playing__cover {
  margin-bottom: 20px;
}

.now-playing__details {
  text-align: center;
}

/* Ensure idle state is full-screen */
.now-playing--idle {
  position: relative;
}
</style>
