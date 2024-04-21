<script setup lang="ts">
// import PdfToJson from "./components/BasicsPdfToJson.vue";
// import PdfToJson from "./components/PdfToJson.vue";
import WordToJson from "./components/WordToJson.vue";
</script>

<template>
  <WordToJson />
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
