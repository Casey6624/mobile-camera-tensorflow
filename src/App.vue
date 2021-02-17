<template>
  <div id="app">
    <h1>TensorFlow camera capturing</h1>
    <CameraCapture />
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
// components
import CameraCapture from "./components/CameraCapture.vue";

@Component({ components: { CameraCapture } })
export default class App extends Vue {}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
