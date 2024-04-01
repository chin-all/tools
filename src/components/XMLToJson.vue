<template>
  <div>
    <input type="file" @change="handleFileChange" accept=".xml" />
    <button @click="convertXMLToJson">Convert</button>
  </div>
</template>

<script>
import xmljs from "xml-js";

export default {
  data() {
    return {
      xmlData: null,
    };
  },
  methods: {
    handleFileChange(event) {
      const file = event.target.files[0];
      const reader = new FileReader();

      reader.onload = (e) => {
        this.xmlData = e.target.result;
      };

      reader.readAsText(file);
    },
    convertXMLToJson() {
      if (!this.xmlData) {
        console.error("No XML data found.");
        return;
      }

      const json = xmljs.xml2json(this.xmlData, { compact: true, spaces: 2 });
      const jsonBlob = new Blob([json], { type: "application/json" });
      const jsonUrl = URL.createObjectURL(jsonBlob);

      // 下载JSON文件
      const a = document.createElement("a");
      a.href = jsonUrl;
      a.download = "output.json";
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);

      // 清除文件选择
      document.querySelector('input[type="file"]').value = null;
    },
  },
};
</script>
