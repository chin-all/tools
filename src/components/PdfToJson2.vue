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
 *  @description: 处理PDF文件并生成JSON数据(信用社)
 */
import { ref } from "vue";
import * as pdfjs from "pdfjs-dist";
import { random } from "lodash";
const jsonData = ref("");
const isLoading = ref(false);

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
  const startRegex = "对方户名";
  const endRegex =
    "===============================================================================================================================";
  // 匹配开始的数据
  const startIndex = data.findIndex((item) => item == startRegex);
  // 匹配到 第*页 结束的数据
  let endIndex = data.findIndex((item) => item == endRegex);
  if (endIndex === -1) {
    endIndex = data.length; // 如果找不到结束模式，则将结束索引设置为数组的长度
  }
  const validData =
    startIndex !== -1 ? data.slice(startIndex + 1, endIndex) : [];
  return validData;
};
// 分割数组
const splitByDate = (dataArray) => {
  const splitArrays = [];
  let tempArray = [];

  for (let i = 0; i < dataArray.length; i++) {
    // 此处的数字要根据pdf进行变化
    if (dataArray[i] === "2681") {
      if (tempArray.length > 0) {
        splitArrays.push(tempArray);
      }
      tempArray = [dataArray[i - 1], dataArray[i]];
    } else {
      tempArray.push(dataArray[i]);
    }
  }

  if (tempArray.length > 2) {
    splitArrays.push(tempArray);
  }

  // 删掉无用的数据，并去重
  splitArrays.shift();
  for (let i = 0; i < splitArrays.length; i++) {
    const childLength = splitArrays[i].length;
    if (!splitArrays[i + 1]) {
      continue;
    }
    if (splitArrays[i][childLength - 1] === splitArrays[i + 1][0]) {
      splitArrays[i].pop();
    }
    // 有个本行atm的数据有点问题，会对数据照成影响，特殊处理一下
    if (splitArrays[i].find((item) => item === "本行atm") && i > 0) {
      splitArrays[i - 1].push(splitArrays[i][0]);
      splitArrays[i][0] = "";
    }
  }
  return splitArrays;
};
// 获取数据内容
const formatTransaction = (transaction) => {
  let [date, balance, money, time, summary, username, account, type] = [
    transaction[5],
    transaction[2],
    transaction[3],
    transaction[5],
    transaction[0],
    "",
    "",
    transaction[4] == "借方" ? "支出" : "收入",
  ];
  if (transaction.length === 11) {
    username = transaction[9] + transaction[10];
    account = transaction[8];
  } else if (transaction.length === 10) {
    username = transaction[9];
    account = transaction[8];
  } else if (transaction.length === 9) {
    username = transaction[8];
    account = transaction[7];
  } else if (transaction.length === 7) {
    username = transaction[6];
  }

  const formattedTransaction = {
    date,
    balance,
    money,
    time,
    summary,
    username,
    account,
    type,
    code: random(100000, 999999),
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
  const pdfText = initial + "\n";
  const initData = JSON.stringify(pdfText, null, 2);
  const jsonList = initial.filter((item) => item !== "" && item !== " ");
  const dateData = getEffectData(jsonList);
  const splitList = splitByDate(dateData);
  const finallyData = splitList.map((item) => formatTransaction(item));
  console.log("dateData", dateData);
  console.log("splitList", splitList);
  console.log("finallyData", finallyData);
  return finallyData;
};

// 最后将数据转换为想要的格式
const dateToWant = (dataSource) => {
  return dataSource.reduce((acc, item) => {
    const existingDateIndex = acc.findIndex((obj) => obj.date === item.date);
    if (existingDateIndex !== -1) {
      acc[existingDateIndex].list.push({
        balance: item.balance,
        money: item.money,
        time: item.time,
        summary: item.summary,
        username: item.username,
        account: item.account,
        type: item.type,
        code: item.code,
      });
    } else {
      acc.push({
        date: item.date,
        list: [
          {
            balance: item.balance,
            money: item.money,
            time: item.time,
            summary: item.summary,
            username: item.username,
            account: item.account,
            type: item.type,
            code: item.code,
          },
        ],
      });
    }
    return acc;
  }, []);
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

      const allDate = [];
      pdfTextList.forEach((item) => {
        const target = getDateToTarget(item);
        allDate.push(...target);
      });

      jsonData.value = dateToWant(allDate.reverse());
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
  a.download = `信用社.json`;
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
