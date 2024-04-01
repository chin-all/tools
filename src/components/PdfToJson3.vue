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
 *  @description: 处理PDF文件并生成JSON数据(建行)
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
const normalType = ["消费", "支付机构提现", "跨行转出", "代理付款", "转账支取"];
const otherType = ["利息存入", "ATM取款"];
const splitByDate = (dataArray) => {
  const splitArrays = [];
  let tempArray = [];

  for (let i = 0; i < dataArray.length; i++) {
    if (normalType.includes(dataArray[i])) {
      if (tempArray.length > 0) {
        splitArrays.push(tempArray);
      }
      // 加一条判断，如果第一个里面没有数字，则再多添加一条进去
      if (!/\d/.test(dataArray[i - 1]) && dataArray[i - 2]) {
        tempArray = [dataArray[i - 2] + dataArray[i - 1], dataArray[i]];
      } else {
        tempArray = [dataArray[i - 1], dataArray[i]];
      }
    } else if (otherType.includes(dataArray[i])) {
      if (tempArray.length > 0) {
        splitArrays.push(tempArray);
      }
      tempArray = [dataArray[i]];
    } else {
      tempArray.push(dataArray[i]);
    }
  }

  if (tempArray.length > 0) {
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
    if (
      splitArrays[i][childLength - 2] &&
      splitArrays[i][childLength - 2] + splitArrays[i][childLength - 1] ===
        splitArrays[i + 1][0]
    ) {
      splitArrays[i].pop();
      splitArrays[i].pop();
    }
  }
  return splitArrays;
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

// 把"1000050001/微信转账" 转换为 "1000***0001 微信转账"
const formatWhere = (transaction) => {
  // 使用正则表达式匹配数字部分并替换
  const formattedTransaction = transaction.replace(
    /(\d{4})\d+(\d{4})/,
    "$1***$2"
  );
  // 替换斜杠为空格
  return formattedTransaction.replace("/", " ");
};

// 获取数据内容
const formatTransaction = (transaction) => {
  // 在属于otherType数据的数组头部添加空字符串
  if (otherType.includes(transaction[0])) {
    transaction.unshift("");
  }

  const dateObject = moment(transaction[4], "YYYYMMDD");
  const moneyType = parseFloat(transaction[2].replace(/,/g, ""));
  let [account, summary, money, balance, date, billType, where, title] = [
    transaction[0],
    transaction[1],
    transaction[2],
    transaction[3],
    dateObject.format("YYYY/MM/DD"),
    moneyType >= 0 ? 1 : 2,
    "",
    convertToChineseWeekday(dateObject.format("YYYY/MM/DD dddd")),
  ];
  if (normalType.includes(summary)) {
    // account = transaction.slice(6).join("");
    // where = formatWhere(where);
    account = formatWhere(account);
    where = transaction.slice(6).join("");
  }

  const formattedTransaction = {
    where,
    summary,
    money,
    balance,
    date,
    billType,
    account,
    title,
    dateTime: "",
  };

  return formattedTransaction;
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
  console.log("splitList", splitList);
  // console.log("finallyData", finallyData);
  return finallyData;
};

// 最后将数据转换为想要的格式
const dateToWant = (dataSource) => {
  return dataSource.reduce((acc, item) => {
    const existingDateIndex = acc.findIndex((obj) => obj.title === item.title);
    if (existingDateIndex !== -1) {
      acc[existingDateIndex].list.push({
        ...item,
      });
    } else {
      acc.push({
        title: item.title,
        date: item.date,
        list: [
          {
            ...item,
          },
        ],
      });
    }
    return acc;
  }, []);
};

// 增加dateTime
const addRandomTime = (dateString, count) => {
  const dates = [];
  // 解析输入的日期字符串
  const dateObject = new Date(dateString);
  // 生成随机时间并按顺序添加到数组中
  for (let i = 0; i < count; i++) {
    const randomHour = Math.floor(Math.random() * 24);
    const randomMinute = Math.floor(Math.random() * 60);
    const randomSecond = Math.floor(Math.random() * 60);
    dateObject.setHours(randomHour);
    dateObject.setMinutes(randomMinute);
    dateObject.setSeconds(randomSecond);
    // 复制日期对象，避免引用问题
    const newDateObject = new Date(dateObject);
    dates.push(newDateObject);
  }
  // 对数组中的日期对象进行从晚到早的排序
  dates.sort((a, b) => b - a);
  // 将日期对象格式化为字符串
  const formattedDates = dates.map((dateObject) => {
    return dateObject.toISOString().slice(0, 19).replace("T", " ");
  });

  return formattedDates;
};
const addDateTime = (dataSource) => {
  return dataSource.map((item) => {
    const countLength = item.list.length;
    const randomTimes = addRandomTime(item.list[0].date, countLength);
    item.list = item.list.map((subItem, index) => {
      return {
        ...subItem,
        dateTime: randomTimes[index],
      };
    });
    return item;
  });
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
      const lastDate = dateToWant(allDate.reverse());
      jsonData.value = addDateTime(lastDate);
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
  a.download = `建行.json`;
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
