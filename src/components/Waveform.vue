<template>
  <div class="visualiser-container" @click="getFile">
    <div class="visualiser-controls">
      <audio
        :src="audioSrc"
        ref="audioPlayer"
        @ended="endPlaying" />
      <button
        type="button"
        @click="togglePlay">
        <span v-if="!audioStatus.status || ['ENDED', 'PAUSED'].includes(audioStatus.status)">Play</span>
        <span v-if="audioStatus.status === 'PLAYING'">Pause</span>
      </button>
    </div>
    <div
      class="visualiser"
      ref="visualiser"
      @mousedown="jumpTo" >
      <span
        v-for="bar in divisions"
        :key="`visualiser-bar-${bar}`"
        :style="{
          width: `calc(${ (1 / divisions * 100) }% - 1px)`,
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
</template>

<script>
export default {
  name: 'Waveform',
    data() {
    return {
      audioSrc: require('../assets/short.wav'),
      audioStatus: {
        status: '',
      },
      buffer: null,
      context: null,
      lastTick: null,
      lastTickInterval: null,
      divisions: 15,
      scrubbing: false,
      scrubPosition: null,
    };
  },
  computed: {
    processedBuffer() {
      if (!this.buffer) return [];

      const rawData = this.buffer.getChannelData(0);
      if (!rawData.length) return [];

      const divisionSize = Math.floor(rawData.length / this.divisions);
      // Filter from thousands of data to a few by averaging over blocks
      const filteredData = rawData.reduce((acc, cur, i) => {
        const division = Math.floor(i / divisionSize);
        if (division >= acc.length) {
          acc[acc.length - 1] += Math.abs(cur);
          return acc;
        }

        acc[division] += Math.abs(cur);
        return acc;
      }, Array.from(new Array(this.divisions)).map(() => 0)).map(val => val / divisionSize);

      // Normalise
      const max = Math.max(...filteredData);
      return filteredData.map(val => val * 1 / max);
    },
    indicatorProgress() {
      if (!('offset' in this.audioStatus)) return 0;
      if (!('start' in this.audioStatus)) return this.audioStatus.offset;
      if (this.audioStatus.status === 'ENDED') return 0;
      if (this.audioStatus.status === 'PAUSED') return this.audioStatus.offset;

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
      const promise = (!this.buffer || !Object.keys(this.buffer).length) ? this.getFile() : Promise.resolve();
      promise.then(() => {
        switch (this.audioStatus.status) {
          case 'PLAYING':
            this.pause();
            break;
          case 'PAUSED':
            this.play(this.audioStatus.offset / 100);
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
    setDivisions() {
      this.divisions = Math.max(this.$refs.visualiser.clientWidth - 100, 15);
    },
  },
  mounted() {
    this.lastTickInterval = setInterval(() => this.lastTick = +new Date());

    this.setDivisions();

    document.addEventListener('mouseup', this.scrubEnd);
    document.addEventListener('mousemove', this.scrubMove);

    window.addEventListener('resize', this.setDivisions);
  },
  beforeDestroy() {
    clearInterval(this.lastTickInterval);
    this.lastTickInterval = null;
    if (!this.context) return;
    this.context.close();
  },
};
</script>

<style lang="scss">
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
</style>