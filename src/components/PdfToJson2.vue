<template>
  <div>
    <input type="file" @change="handleFileUpload" />
    <VuePdf v-if="pdfSrc" :src="pdfSrc" @loaded="handleLoaded" />
  </div>
</template>

<script setup>
import { ref } from "vue";
import VuePdf from "vue-pdf";

const pdfSrc = ref(null);
const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (file) {
    pdfSrc.value = URL.createObjectURL(file);
  }
};
const handleLoaded = (pdf) => {
  console.log("PDF loaded:", pdf);
  console.log("Number of pages:", pdf.numPages);
};
</script>
