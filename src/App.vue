<template>
  <div id="app">
    <waveform />
  </div>
</template>

<script>
import Waveform from './components/Waveform';

export default {
  name: 'app',
  components: {
    Waveform,
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
}
</style>
