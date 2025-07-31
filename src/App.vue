<script setup>
import AudioPlayer from './components/AudioPlayer.vue'
import MicInput from './components/MicInput.vue'
import MicRecorder from './components/MicRecorder.vue'
import DualMicRecorder from './components/DualMicRecorder.vue';
import DualChannelRecorder from './components/DualChannelRecorder.vue';

</script>

<template>
  <div id="app">
    <h1>🎧 Proyecto WebAudio</h1>
    <!--<AudioPlayer />
    <MicInput />
    <MicRecorder />
    <DualMicRecorder-->
      <DualChannelRecorder/>
  </div>
</template>


<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
