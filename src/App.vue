<template>
  <div id="app">
    <header>UNIVERSAL BASIC GALLERY</header>
    <!-- <header>ʎɹǝllɐ⅁ ɔısɐᙠ lɐsɹǝʌıu∩</header> -->
    <LiveNFT/>
  </div>
</template>

<script>
import LiveNFT from './components/LiveNFT'

export default {
  name: 'App',
  components: {
    LiveNFT
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
