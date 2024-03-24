<template>
  <div>
    <input type="file" @change="onFileChange" />
  </div>
</template>

<script>
import { ref } from "vue";
import { getDocument } from "pdfjs-dist";

export default {
  setup() {
    const onFileChange = async (e) => {
      const file = e.target.files[0];
      if (file) {
        const pdf = await getDocument(URL.createObjectURL(file)).promise;
        const numPages = pdf.numPages;

        for (let i = 1; i <= numPages; i++) {
          const page = await pdf.getPage(i);
          const content = await page.getTextContent();
          const strings = content.items.map((item) => item.str);
          console.log(strings.join(" "));
        }
      }
    };

    return {
      onFileChange,
    };
  },
};
</script>
