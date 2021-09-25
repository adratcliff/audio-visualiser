<template>
  <div id="app" @click="getFile">
    <audio
      :src="audioSrc"
      ref="audioPlayer"
      controls />
    <button
      type="button"
      @click="play">
      Play
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
      <span
        class="visualiser-indicator"
        ref="indicator"
        :style="{
          left: `${indicatorPos}px`,
        }" />
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
      buffer: null,
      context: null,
      samples: 100,
      indicatorPos: 24,
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
  },
  methods: {
    getFile() {
      this.context = new AudioContext();
      return fetch(this.audioSrc)
        .then(response => response.arrayBuffer())
        .then(arrayBuffer => this.context.decodeAudioData(arrayBuffer))
        .then(audioBuffer => this.buffer = audioBuffer);
    },
    play() {
      this.getFile().then(() => {
        console.log(this.buffer)
        this.$refs.audioPlayer.play();
      })
    },
  },
  beforeDestroy() {
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
    .visualiser-indicator {
      width: 2px;
      height: 90%;
      background-color: red;
      position: absolute;
    }
  }
}
</style>
