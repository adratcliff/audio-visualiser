<template>
  <div id="app" @click="getFile">
    <audio
      :src="audioSrc"
      ref="audioPlayer"
      @ended="endPlaying" />
    <div class="visualiser-container">
      <div class="visualiser-controls">
        <button
          type="button"
          @click="togglePlay">
          <span v-if="!audioStatus.status || ['ENDED', 'PAUSED'].includes(audioStatus.status)">Play</span>
          <span v-if="audioStatus.status === 'PLAYING'">Pause</span>
        </button>
      </div>
      <div
        v-if="!!processedBuffer.length"
        class="visualiser"
        ref="visualiser"
        @mousedown="jumpTo" >
        <span
          v-for="bar in samples"
          :key="`visualiser-bar-${bar}`"
          :style="{
            width: `calc(${ (1 / samples * 100) }% - 1px)`,
            height: `${ processedBuffer[bar] * 100 }%`,
          }"
          class="visualiser-bar" />
        <span
          class="visualiser-indicator"
          ref="indicator"
          :style="{
            left: scrubPosition ? `${scrubPosition}px` : `${indicatorProgress}%`,
          }"
          :class="{
            scrubbing: scrubPosition !== null,
          }"
          @mousedown.stop="startScrub" />
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
      samples: 300,
      scrubbing: false,
      scrubPosition: null,
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
      return Math.min((this.audioStatus.offset + (timePassed / this.buffer.duration)) * 100, 100);
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
    play(offset=0) {
      this.$refs.audioPlayer.currentTime = this.buffer.duration * offset;
      this.$refs.audioPlayer.play();

      this.audioStatus = {
        start: +new Date(),
        status: 'PLAYING',
        offset,
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
    startScrub() {
      this.scrubbing = true;
      this.$refs.audioPlayer.pause();
    },
    setScrubPosition(evt) {
      this.scrubPosition = Math.min(Math.max(evt.pageX - this.$refs.visualiser.offsetLeft, 0), this.$refs.visualiser.clientWidth);
    },
    scrubMove(evt) {
      if (!this.scrubbing) return;
      this.setScrubPosition(evt);
    },
    scrubEnd() {
      if (!this.scrubbing) return;
      this.play(this.scrubPosition / this.$refs.visualiser.clientWidth);
      this.scrubbing = false;
      this.scrubPosition = null;
    },
    jumpTo(evt) {
      this.startScrub();
      this.setScrubPosition(evt);
    },
  },
  mounted() {
    this.lastTickInterval = setInterval(() => this.lastTick = +new Date());
    document.addEventListener('mouseup', this.scrubEnd);
    document.addEventListener('mousemove', this.scrubMove);
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
  margin-top: 20vh;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-flow: column;
  .visualiser-container {
    width: 80vw;
    height: 100px;
    padding: 24px;
    border-radius: 2rem;
    background-color: #2c3e50;
    display: flex;
    flex-flow: row;
    .visualiser-controls {
      min-width: 100px;
      height: 100%;
      margin: 0 24px;
      display: flex;
      align-items: center;
      justify-content: center;
      button {
        height: 60px;
        width: 60px;
        border-radius: 50%;
      }
    }
    .visualiser {
      width: 100%;
      position: relative;
      display: flex;
      align-items: center;
      justify-content: space-evenly;
      .visualiser-bar {
        max-height: 90px;
        background-color: #fafafa;
      }
      .visualiser-indicator {
        width: 2px;
        height: 90%;
        background-color: #e81313;
        position: absolute;
        pointer-events: all;
        cursor: grab;
        transition: box-shadow ease 100ms;
        &.scrubbing {
          cursor: grabbing;
        }
        &:not(.scrubbing):hover {
          box-shadow: 0px 0px 10px 1px #7f0000;
        }
      }
    }
  }
}
</style>
