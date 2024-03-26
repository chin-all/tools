<template>
  <div>
    <input type="file" @change="handleFileChange" accept=".pdf" />
    <button @click="convertToJSON">Convert to JSON</button>
    <!-- <pre>{{ jsonData }}</pre> -->
  </div>
</template>

<script>
import { ref } from "vue";
import * as pdfjs from "pdfjs-dist";

export default {
  name: "PdfToJson",
  setup() {
    const jsonData = ref("");

    // 设置全局 Worker 文件路径
    pdfjs.GlobalWorkerOptions.workerSrc =
      "node_modules/pdfjs-dist/build/pdf.worker.mjs"; // 这里需要根据实际的路径修改

    const handleFileChange = (event) => {
      const file = event.target.files[0];
      if (file) {
        convertToJSON(file);
      }
    };

    const convertToJSON = (file) => {
      const reader = new FileReader();
      reader.onload = async (event) => {
        const data = event.target.result;
        const typedArray = new Uint8Array(data);
        const pdf = await pdfjs.getDocument(typedArray).promise;
        const numPages = pdf.numPages;
        let pdfText = "";

        for (let i = 1; i <= numPages; i++) {
          const page = await pdf.getPage(i);
          const content = await page.getTextContent();
          const text = content.items.map((item) => item.str).join("\n");
          pdfText += text + "\n";
        }

        const json = { pdfText }; // You can parse pdfText further into JSON as needed
        jsonData.value = JSON.stringify(json, null, 2);
        console.log("转换后的数据", JSON.stringify(json));
      };

      reader.readAsArrayBuffer(file);
    };

    return {
      jsonData,
      handleFileChange,
      convertToJSON,
    };
  },
};
</script>
