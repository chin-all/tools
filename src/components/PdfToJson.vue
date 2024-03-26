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
 *  @description: 处理PDF文件并生成JSON数据(可以多页转换版)
 */
import { ref } from "vue";
import * as pdfjs from "pdfjs-dist";

const jsonData = ref("");
const isLoading = ref(false);
const dateValue = [];

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

// 获取从开始到结束的有效数据
const getEffectData = (data) => {
  const startRegex = /\d{8}\s\d{6}/; // 例 "20230320 131631"
  const startRegexShot = /\b\d{8}\b/; // 例 "20230320"
  // 匹配开始的数据
  const startIndex = data.findIndex(
    (item) =>
      startRegex.test(item) ||
      (startRegexShot.test(item) && !/\d{8}-\d{8}/.test(item))
  );
  // 匹配到 第*页 结束的数据
  let endIndex = data.findIndex((item) => /第\d+页\/共\d+页/.test(item));
  if (endIndex === -1) {
    endIndex = data.length; // 如果找不到结束模式，则将结束索引设置为数组的长度
  }
  const validData = startIndex !== -1 ? data.slice(startIndex, endIndex) : [];
  return validData;
};
// 分割数组
const splitByDate = (data) => {
  const result = [];
  let currentSubarray = [];

  for (const item of data) {
    if (/^\d{8}(?:\s\d{6})?$/.test(item)) {
      // 如果当前项是日期格式，则将当前子数组推送到结果数组中，并创建一个新的子数组
      if (currentSubarray.length > 0) {
        result.push(currentSubarray);
        currentSubarray = [];
      }
    }
    currentSubarray.push(item);
  }

  // 推送最后一个子数组到结果数组中
  if (currentSubarray.length > 0) {
    result.push(currentSubarray);
  }

  return result;
};
// 获取数据内容
const formatTransaction = (transaction) => {
  const [
    time, // 日期时间
    tradingNetwork, // 交易网点
    name, // 短摘要
    balance, // 本次余额
    logNum, // 日志号
    channel, // 渠道
    money, // 交易金额
    info, // 对方账号户名 or 附言
    desc, // 附言
  ] = transaction;

  const receive = info?.split(" ") ?? [];

  let account = "";
  let receiveNm = "";
  if (receive.length > 1) {
    account = receive[0];
    receiveNm = receive[1];
  }

  const formattedTransaction = {
    time: formatTime(time),
    name,
    money,
    balance,
    channel,
    account,
    receiveNm: receiveNm || "",
    desc: desc ?? info ?? "",
    type: "转账",
    billType: parseFloat(money) < 0 ? 1 : 2,
  };

  return formattedTransaction;
};
// 格式化时间
const formatTime = (timeString) => {
  const year = timeString.slice(0, 4);
  const month = timeString.slice(4, 6);
  const day = timeString.slice(6, 8);
  const hour = timeString.slice(9, 11) || "00";
  const minute = timeString.slice(11, 13) || "00";
  const second = timeString.slice(13, 15) || "00";
  return `${year}-${month}-${day} ${hour}:${minute}:${second}`;
};

// 获取有效数据并转换为目标数据
const getDateToTarget = (initial) => {
  const dateRangeRegex = /\b\d{8}-\d{8}\b/; // 匹配日期范围格式，例如 "20230320-20240320"
  // 获取日期数据
  dateValue.value = initial.find((item) => dateRangeRegex.test(item));
  // console.log("dateValue.value", dateValue.value);
  const pdfText = initial + "\n";
  const initData = JSON.stringify(pdfText, null, 2);
  const jsonList = initial.filter((item) => item !== "" && item !== " ");
  const dateData = getEffectData(jsonList);
  const splitList = splitByDate(dateData);
  const finallyData = splitList.map((item) => formatTransaction(item));
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
      console.log("最终数据：", allDate);
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
