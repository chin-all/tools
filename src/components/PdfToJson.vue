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
 *  @description: 处理PDF文件并生成JSON数据(农业银行)
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
// 20113201040002869柳州市香妃美容用品有限公司 数字+汉字组合
function validateString(input) {
  // 正则表达式匹配数字加汉字
  const regex = /^[\d]+[\u4e00-\u9fa5]+$/;
  return regex.test(input);
}
// 拆分组合
function splitChineseAndDigits(input) {
  // 使用正则表达式将汉字替换为空字符串
  const digitsString = input.replace(/[\u4e00-\u9fa5]/g, "");

  // 使用正则表达式将数字替换为空字符串
  const chineseString = input.replace(/\d/g, "");

  return { digitsString, chineseString };
}
// 根据数字和非数字字符进行分割
function splitStringByNumbersAndNonNumbers(str) {
  // 使用正则表达式匹配数字和非数字字符
  const regex = /(\d+)|(\D+)/g;
  const matches = str.match(regex);
  // 过滤掉匹配结果中的 undefined
  const result = matches.filter((match) => match !== undefined);
  return result;
}
// 获取数据内容
const formatTransaction = (transaction) => {
  let [
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
  // let payNm = ""; // 新增字段 收款方
  if (receive.length > 1) {
    // 防止这样的数据 20113201040002869柳州市香妃美容用品有限公司 数字+汉字组合
    if (validateString(receive[0])) {
      const { digitsString, chineseString } = splitChineseAndDigits(receive[0]);
      account = digitsString;
      receiveNm = chineseString + receive[1];
      // receiveNm = chineseString;
      desc = "";
    } else {
      account = receive[0];
      receiveNm = receive[1];
      // desc = receive[1];
    }
  }
  // 防止这样的数据 6231330100049464693叶正清
  if (validateString(info)) {
    const { digitsString, chineseString } = splitChineseAndDigits(info);
    account = digitsString;
    receiveNm = chineseString;
    // payNm = chineseString;
    desc = "";
  }

  // if (transaction.includes("工资")) {
  //   // console.log("transaction", transaction);
  //   const lengthM = transaction.length;
  //   if (validateString(transaction[lengthM - 1])) {
  //     // 不做操作
  //   } else {
  //     account = transaction[lengthM - 2];
  //     receiveNm = transaction[lengthM - 1];
  //     desc = "";
  //   }
  // }

  // 养老金
  if (info && info.includes("养老保险基金")) {
    const stringList = splitStringByNumbersAndNonNumbers(info);
    console.log("养老保险基金", stringList);
    account = stringList[0];
    receiveNm = stringList[1];
    desc = "";
  }

  // 转支 和 工资 和 转存 如下
  const formattedTransaction = {
    time: formatTime(time), // 交易时间
    name, // 交易摘要
    money, // 交易金额
    balance, // 本次余额
    channel, // 交易渠道
    // account,
    // receiveNm: receiveNm || "",
    /* ⬇️新增⬇️ */
    // collectionNm: parseFloat(money) > 0 ? receiveNm : "", // 收款方
    // collectionAccount: parseFloat(money) > 0 ? account : "", // 收款账户
    // payNm: parseFloat(money) < 0 ? receiveNm : "", // 付款方
    // payAccount: parseFloat(money) < 0 ? account : "", // 付款账户
    /* ⬆️新增⬆️ */
    desc: desc ?? info ?? "", // 交易附言
    type: "转账", // 交易类型
    // 1 支付（负） 2 收入（正）
    // 类型 1 付款方 付款账户
    // 类型 2 收款方 收款账户
    billType: parseFloat(money) < 0 ? 1 : 2,
  };
  const otherType = ["工资", "转支", "转存", "养老金"];
  if (!otherType.includes(name)) {
    // formattedTransaction.account = account;
    // formattedTransaction.receiveNm = receiveNm || "";
    formattedTransaction.account = "";
    formattedTransaction.receiveNm = "";
  } else {
    formattedTransaction.collectionNm = parseFloat(money) > 0 ? receiveNm : "";
    formattedTransaction.collectionAccount =
      parseFloat(money) > 0 ? account : "";
    formattedTransaction.payNm = parseFloat(money) < 0 ? receiveNm : "";
    formattedTransaction.payAccount = parseFloat(money) < 0 ? account : "";
  }

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
  console.log("splitList", splitList);
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
  a.download = `农业银行.json`;
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
