<template>
  <div id="app" @click="getFile">
    <audio
      :src="audioSrc"
      ref="audioPlayer"
      controls
      @ended="endPlaying" />
    <button
      type="button"
      @click="togglePlay">
      <span v-if="!audioStatus.status || ['ENDED', 'PAUSED'].includes(audioStatus.status)">Play</span>
      <span v-if="audioStatus.status === 'PLAYING'">Pause</span>
    </button>
    <div class="visualiser" v-if="!!processedBuffer.length">
      <span
        v-for="bar in samples"
        :key="`visualiser-bar-${bar}`"
        :style="{
          width: `calc(${ (1 / samples * 100) }% - 1px)`,
          height: `${ processedBuffer[bar] * 100 }%`,
        }"
        class="visualiser-bar" />
      <div class="visualiser-indicator-container">
        <span
          class="visualiser-indicator"
          ref="indicator"
          :style="{
            left: `${indicatorProgress}%`
          }" />
      </div>
    </div>
  </div>
</template>

<script>

export default {
  name: 'app',
  components: {},
  data() {
    return {
      audioSrc: require('./assets/short.wav'),
      audioStatus: {},
      buffer: null,
      context: null,
      lastTick: null,
      lastTickInterval: null,
      samples: 10,
    };
  },
  computed: {
    processedBuffer() {
      if (!this.buffer) return [];

      const rawData = this.buffer.getChannelData(0);
      if (!rawData.length) return [];

      const divisionSize = Math.floor(rawData.length / this.samples);
      // Filter from thousands of data to a few by averaging over blocks
      const filteredData = rawData.reduce((acc, cur, i) => {
        const division = Math.floor(i / divisionSize);
        if (division >= acc.length) {
          acc[acc.length - 1] += Math.abs(cur);
          return acc;
        }

        acc[division] += Math.abs(cur);
        return acc;
      }, Array.from(new Array(this.samples)).map(() => 0)).map(val => val / divisionSize);

      // Normalise
      const max = Math.max(...filteredData);
      return filteredData.map(val => val * 1 / max);
    },
    indicatorProgress() {
      if (!('offset' in this.audioStatus)) return 0;
      if (!('start' in this.audioStatus)) return this.audioStatus.offset;
      if ('status' in this.audioStatus) {
        if (this.audioStatus.status === 'ENDED') return 0;
        if (this.audioStatus.status === 'PAUSED') return this.audioStatus.offset;
      }

      const timePassed = (this.lastTick - this.audioStatus.start) / 1000;
      return Math.min(this.audioStatus.offset + (timePassed / this.buffer.duration) * 100, 100);
    },
  },
  methods: {
    getFile() {
      this.context = new AudioContext();
      return fetch(this.audioSrc)
        .then(response => response.arrayBuffer())
        .then(arrayBuffer => this.context.decodeAudioData(arrayBuffer))
        .then(audioBuffer => this.buffer = audioBuffer);
    },
    togglePlay() {
      const promise = !Object.keys(this.buffer).length ? this.getFile() : Promise.resolve();
      promise.then(() => {
        if (!('status' in this.audioStatus)) {
          this.play();
          return;
        }

        switch (this.audioStatus.status) {
          case 'PLAYING':
            this.pause();
            break;
          case 'PAUSED':
            this.resumePlay();
            break;
          case 'ENDED':
          default:
            this.play();
            break;
        }
      });
    },
    play() {
      this.$refs.audioPlayer.play();
      this.audioStatus = {
        start: +new Date(),
        status: 'PLAYING',
        offset: 0,
      };
    },
    pause() {
      this.$refs.audioPlayer.pause();
      this.audioStatus = {
        status: 'PAUSED',
        offset: this.indicatorProgress,
      };
    },
    resumePlay() {
      this.$refs.audioPlayer.play();
      this.audioStatus = {
        ...this.audioStatus,
        status: 'PLAYING',
        start: +new Date(),
      };
    },
    endPlaying() {
      this.audioStatus.status = 'ENDED';
    },
  },
  mounted() {
    this.lastTickInterval = setInterval(() => this.lastTick = +new Date());
  },
  beforeDestroy() {
    clearInterval(this.lastTickInterval);
    this.lastTickInterval = null;
    if (!this.context) return;
    this.context.close();
  },
}
</script>

<style lang="scss">
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-flow: column;
  .visualiser {
    width: 60vw;
    height: 100px;
    margin: 24px;
    padding: 24px;
    border-radius: 2rem;
    background-color: #2c3e50;
    position: relative;
    display: flex;
    align-items: center;
    justify-content: space-evenly;
    .visualiser-bar {
      max-height: 90px;
      background-color: #fafafacc;
    }
    .visualiser-indicator-container {
      width: calc(100% - 48px);
      height: 100%;
      position: absolute;
      bottom: 0;
      display: flex;
      align-items: center;
      .visualiser-indicator {
        width: 2px;
        height: 90%;
        background-color: red;
        position: absolute;
      }
    }
  }
}
</style>
