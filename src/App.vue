<template>
  <div class="app">
    <div class="left"></div>
    <div class="right">
      <Scrrenshot />
    </div>
  </div>
</template>

<script>
import Scrrenshot from './components/Scrrenshot.vue'
export default {
  components: {
    Scrrenshot
  },
}
</script>

<style scoped>
.app {
  display: flex;
  height: 100vh;
  width: 100vw;
  background-color: antiquewhite;
}

.left {
  display: flex;
  flex: 1;
  background-color: beige;
}

.right {
  display: flex;
  flex: 0 0 200px;
  background-color: honeydew;
  flex-direction: column;
  padding-top: 20px;
}
</style>
