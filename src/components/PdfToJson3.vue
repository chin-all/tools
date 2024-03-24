<template>
  <div>
    <input type="file" @change="onFileChange" />
  </div>
</template>

<script>
import { ref } from "vue";
import PDFParser from "pdf2json";

export default {
  setup() {
    const onFileChange = async (e) => {
      const file = e.target.files[0];
      if (file) {
        const pdfParser = new PDFParser();

        pdfParser.on("pdfParser_dataError", (errData) =>
          console.error(errData.parserError)
        );
        pdfParser.on("pdfParser_dataReady", (pdfData) => {
          console.log(JSON.stringify(pdfData));
        });

        pdfParser.loadPDF(URL.createObjectURL(file));
      }
    };

    return {
      onFileChange,
    };
  },
};
</script>
