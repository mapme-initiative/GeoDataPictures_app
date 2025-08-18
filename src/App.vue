<template>
  <div id="app">
    <MapWithExif />
  </div>
</template>

<script>
import MapWithExif from "./components/MapWithExif.vue"

export default {
  name: "App",
  components: {
    MapWithExif,
  }
};
</script>

<style>
#app {
  font-family: Arial, sans-serif;
  text-align: center;
  margin-top: 20px;
}
</style>
