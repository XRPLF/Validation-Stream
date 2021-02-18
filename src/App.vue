<template>
  <div id="app" class="container-fluid pt-2">
    <Stream />
  </div>
</template>

<script>
import Stream from './components/Stream.vue'

export default {
  name: 'App',
  components: {
    Stream
  }
}
</script>

<style lang="scss">
  #app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
  }
</style>
