<template>
  <div id="app">
    <!-- NOW PLAYING (Active track) -->
    <div
      v-if="player && player.trackTitle"
      class="now-playing now-playing--active"
    >
      <!-- Full album art – fits completely (contain) -->
      <div
        class="album-art-background"
        :style="{ backgroundImage: 'url(' + player.trackAlbum.image + ')' }"
      ></div>

      <!-- Heavy blur + dark overlay -->
      <div class="background-overlay"></div>

      <!-- Title + Artist – perfectly centered -->
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="formattedTrackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>

    <!-- IDLE STATE -->
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
/* ← Your entire <script> section remains 100% unchanged from last version → */
/* Just copy your current working script here (everything works perfectly) */
import * as Vibrant from 'node-vibrant'
import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',
  props: { auth: props.auth, endpoints: props.endpoints, player: props.player },
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
    getTrackArtists() { return this.player.trackArtists.join(', ') },
    formattedTrackTitle() { return this.player?.trackTitle || '' }
  },
  watch: {
    player: {
      handler(newPlayer) {
        if (newPlayer && newPlayer.trackTitle) {
          if (this.imageCycleInterval) { clearInterval(this.imageCycleInterval); this.imageCycleInterval = null }
        } else if (!this.imageCycleInterval) this.startImageCycle()
      },
      deep: true
    },
    auth(newVal) { if (!newVal.status) clearInterval(this.pollPlaying) },
    playerResponse() { this.handleNowPlaying() },
    playerData() { this.getAlbumColours() }
  },
  mounted() {
    this.setDataInterval()
    if (!(this.player && this.player.trackTitle)) this.startImageCycle()
  },
  beforeDestroy() {
    clearInterval(this.pollPlaying)
    if (this.imageCycleInterval) clearInterval(this.imageCycleInterval)
  },
  methods: {
    startImageCycle() {
      if (!this.idleImages.length) return
      this.imageCycleInterval = setInterval(() => {
        this.currentImageIndex = (this.currentImageIndex + 1) % this.idleImages.length
      }, 60000)
    },
    // ... all your existing methods (getNowPlaying, handleNowPlaying, etc.) stay exactly the same
    // → just paste your current working methods here
    getEmptyPlayer() { return { trackAlbum: {}, trackArtists: [], trackId: '', trackTitle: '' } },
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => this.getNowPlaying(), 2500)
    },
    // ... keep everything else unchanged ...
    async getNowPlaying() { /* your existing code */ },
    handleNowPlaying() { /* your existing code */ },
    getAlbumColours() { /* your existing code */ },
    handleAlbumPalette(palette) { /* your existing code */ },
    // etc.
  }
}
</script>

<style scoped>
html, body {
  margin: 0; padding: 0; width: 100%; height: 100%; overflow: hidden; background: #000;
}

#app {
  width: 1024px;
  height: 768px;
  position: fixed;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  overflow: hidden;
  background: #000;
  font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
}

/* Full album art — CONTAIN so entire square artwork is visible */
.album-art-background {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  background-size: contain;           /* ← THIS is the magic */
  background-position: center;
  background-repeat: no-repeat;
  z-index: 0;
}

/* Heavy blur + dark overlay on top of the art */
.background-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.65);
  backdrop-filter: blur(32px);
  -webkit-backdrop-filter: blur(32px);
  z-index: 1;
}

/* Title & artist — perfectly dead-center */
.now-playing__details {
  position: absolute;
  top: 50%; left: 50%;
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
  text-shadow: 0 4px 30px rgba(0,0,0,0.95);
  word-wrap: break-word;
}

.now-playing__artists {
  font-size: 58px;
  font-weight: 400;
  line-height: 1.3;
  margin: 0;
  color: rgba(255,255,255,0.95);
  text-shadow: 0 3px 20px rgba(0,0,0,0.9);
}

/* Idle state */
.now-playing--idle,
.idle-image-container,
.now-playing__idle-image {
  width: 100%; height: 100%;
  position: absolute; top: 0; left: 0;
  object-fit: cover;
}

.fade-enter-active, .fade-leave-active { transition: opacity 2s ease-in-out; }
.fade-enter, .fade-leave-to { opacity: 0; }
</style>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
