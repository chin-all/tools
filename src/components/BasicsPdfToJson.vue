<template>
  <div>
    <input type="file" @change="handleFileChange" accept=".pdf" />
    <div v-if="isLoading">正在加载...</div>
    <button @click="exportJSON" v-if="jsonData">导出JSON</button>
    <pre v-if="jsonData">{{ jsonData }}</pre>
    <button @click="clearJsonData" v-if="jsonData">清除数据</button>
  </div>
</template>

<script setup>
/**
 *  @description: 处理PDF文件并生成JSON数据(仅包括解析)
 */
import { ref } from "vue";
import * as pdfjs from "pdfjs-dist";
import moment from "moment";

const jsonData = ref("");
const isLoading = ref(false);
const dateValue = "bill";

// 设置全局 Worker 文件路径
pdfjs.GlobalWorkerOptions.workerSrc =
  "node_modules/pdfjs-dist/build/pdf.worker.mjs"; // 这里需要根据实际的路径修改

const handleFileChange = (event) => {
  const file = event.target.files[0];
  if (file) {
    clearJsonData();
    isLoading.value = true;
    convertToJSON(file);
  }
};

// 分割数组
// 根据摘要类型进行分割

const splitByDate = (dataArray) => {
  return dataArray;
};
// 中英文星期转换
const convertToChineseWeekday = (dateString) => {
  const weekdays = [
    "Sunday",
    "Monday",
    "Tuesday",
    "Wednesday",
    "Thursday",
    "Friday",
    "Saturday",
  ];
  const chineseWeekdays = [
    "星期日",
    "星期一",
    "星期二",
    "星期三",
    "星期四",
    "星期五",
    "星期六",
  ];
  // 查找英文星期在数组中的索引
  const index = weekdays.indexOf(dateString.split(" ")[1]);
  // 返回对应的中文星期
  return dateString.replace(weekdays[index], chineseWeekdays[index]);
};

// 获取数据内容
const formatTransaction = (transaction) => {
  return transaction;
};

// 获取从开始到结束的有效数据
const getEffectData = (data) => {
  const startRegex = "序号";
  const endRegex = /- 第(\d+)页\/共(\d+)页 -/;
  // 匹配开始的数据
  const startIndex = data.findIndex((item) => item == startRegex);
  // 匹配到 第*页 结束的数据
  let endIndex = data.findIndex((item) => item == endRegex);
  if (endIndex === -1) {
    endIndex = data.length; // 如果找不到结束模式，则将结束索引设置为数组的长度
  }
  const validData = startIndex !== -1 ? data.slice(startIndex + 1, -3) : [];
  return validData;
};

// 获取有效数据并转换为目标数据
const getDateToTarget = (initial) => {
  const jsonList = initial.filter((item) => item !== "" && item !== " ");
  const dateData = getEffectData(jsonList);
  const splitList = splitByDate(dateData);
  const finallyData = splitList.map((item) => formatTransaction(item));
  // console.log("splitList", splitList);
  // console.log("finallyData", finallyData);
  return finallyData;
};

// pdf转换为json
const convertToJSON = (file) => {
  const reader = new FileReader();
  reader.onload = async (event) => {
    try {
      const data = event.target.result;
      const typedArray = new Uint8Array(data);
      const pdf = await pdfjs.getDocument(typedArray).promise;
      const numPages = pdf.numPages;
      const pdfTextList = [];
      for (let i = 1; i <= numPages; i++) {
        const page = await pdf.getPage(i);
        const content = await page.getTextContent();
        const text = content.items.map((item) => item.str);
        pdfTextList.push(text);
      }
      // const finallyData = pdfTextList.map((item) => getDateToTarget(item));
      // jsonData.value = finallyData;

      const allDate = [];
      pdfTextList.forEach((item) => {
        const target = getDateToTarget(item);
        allDate.push(...target);
      });
      jsonData.value = allDate.reverse();
      console.log("最终数据：", jsonData.value);
    } catch (error) {
      console.error("PDF 转换出错:", jsonData.value);
    } finally {
      isLoading.value = false;
    }
  };

  reader.onerror = (error) => {
    console.error("文件读取出错:", error);
    isLoading.value = false;
  };

  reader.readAsArrayBuffer(file);
};

// 清空数据
const clearJsonData = () => {
  jsonData.value = "";
};

// 导出json
const exportJSON = () => {
  const data = JSON.stringify(jsonData.value, null, 2); // 将数组对象转换为 JSON 字符串
  const blob = new Blob([data], { type: "application/json" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `${dateValue.value}.json`;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
};
</script>

<style scoped>
pre {
  white-space: pre-wrap;
}
</style>
