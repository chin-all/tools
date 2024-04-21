<template>
  <div>
    <input type="file" @change="handleFileChange" accept=".doc,.docx" />
    <button @click="convertToJson" :disabled="!fileContent">
      Convert to JSON
    </button>
    <pre v-if="jsonData">{{ jsonData }}</pre>
  </div>
</template>

<script setup>
import { ref } from "vue";
import * as mammoth from "mammoth";

const fileContent = ref("");
const jsonData = ref("");
// 处理文件选择事件
const handleFileChange = (event) => {
  const file = event.target.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = () => {
      fileContent.value = reader.result;
    };
    reader.readAsArrayBuffer(file);
  }
};
// 辅助函数，用于找到指定值在数组中的所有索引
const getAllIndexes = (arr, val) => {
  const indexes = [];
  for (let i = 0; i < arr.length; i++) {
    if (arr[i] === val || arr[i].includes(val)) {
      indexes.push(i);
    }
  }
  return indexes;
};
// 通过日期字符进行拆分，分成多个字符串数组
const splitByDateFormat = (lines) => {
  const resultArray = [];
  let tempArray = [];
  for (const line of lines) {
    if (line.match(/\d{4}-\d{2}-\d{2}/)) {
      if (tempArray.length > 0) {
        resultArray.push(tempArray);
        tempArray = [];
      }
    }
    tempArray.push(line);
  }
  if (tempArray.length > 0) {
    resultArray.push(tempArray);
  }
  return resultArray;
};
// 把数据数组进行转换，换成想要的数据格式
/*
{
  "date": "2023/04", // 整个月的数据
    "list": [
      {
        dealPay:'1,090.00' ,// 交易金额	
        collectionNm:'某某公司或者姓名'  // 收款
        collectionAccount:'卡号' // 收款账户
        payNm :'某某公司或者姓名'  // 对方名称 
        payAccount :'卡号' // 付款账户
        dealTime:'2023/04/02 20:38:32' // 交易时间
        chargeUpTime:'2023/04/02 20:38:32'// 记账时间
        dealType:'网上支付', // 交易类型
        balance: "10,162.19", // 余额
        channel:'在线' , // 渠道
        postscript:'网银' // 附言
        billType: 2, // 1 收入 2 支出
      }
    ]
}
*/
const convertData = (dataArray) => {
  const [
    date, // 记账日期  "2024-04-15"
    time, // 记账时间  "13:52:13"
    type, // 币别  "人民币"
    dealPay, // 金额  "-950.00"
    balance, // 余额  "11,699.49"
    dealType, // 交易名称  "跨行转账"
    channel, // 渠道  "网上银行"
    line, // 网点名称  "-----------------"
    postscript, // 附言  "IBPS30758400799820240415 91578965"
    payNm, // 对方账户名  "蒋清泉"
    collectionAccount, // 对方卡号/账号  "6236693380000372336"
    collectionNm, // 对方开户行  "中国建设银行股份有限公 司总行"
  ] = dataArray;
  return {
    dealPay,
    collectionNm,
    collectionAccount,
    payNm,
    payAccount: "", // 付款账户
    dealTime: `${date} ${time}`,
    chargeUpTime: `${date} ${time}`,
    dealType,
    balance,
    channel,
    postscript,
    billType: 2, // 1 收入 2 支出
  };
};
// 处理转换后的数据
const handleConvertedData = (data) => {
  // 找到所有 "交易区间" 和 "counterparty account" 所在的位置
  const startIndexes = getAllIndexes(data, "交易区间： ");
  const endIndexes = getAllIndexes(data, "counterparty account");
  // 删除所有 "交易区间" 和 "counterparty account" 之间的数据
  if (startIndexes.length > 0 && endIndexes.length > 0) {
    // 逆向遍历每一对目标位置，将其之间的数据都删除
    for (let i = startIndexes.length - 1; i >= 0; i--) {
      const startIndex = startIndexes[i];
      const endIndex = endIndexes.find((index) => index > startIndex);
      if (endIndex !== undefined && endIndex >= 0) {
        data.splice(startIndex, endIndex - startIndex + 1);
      }
    }
  }
  const filterString = "---------------------END---------------------";
  const filterString2 =
    "温馨提示：1.记账日期/时间为系统进行记账处理的日期/时间，可能与实际交易提交时间存在差异。";

  const filterData = data.filter(
    (line) => !line.includes(filterString) && !line.includes(filterString2)
  );
  const convertToData = splitByDateFormat(filterData);
  const convert = filterData.map((data) => convertData(convertToData));
  return convert;
};
// 转换为 JSON
const convertToJson = async () => {
  try {
    const result = await mammoth.extractRawText({
      arrayBuffer: fileContent.value,
    });
    // 将提取到的文本内容按照换行符进行分割，形成一个字符串数组，并过滤掉空字符串
    // 有效数据
    const hasValidData = result.value
      .split("\n")
      .filter((line) => line.trim() !== "");
    // 根据具体需求进行解析和转换
    jsonData.value = handleConvertedData(hasValidData);
    console.log("处理前的数据", hasValidData);
    console.log("最终数据", jsonData.value);
  } catch (error) {
    console.error("Error converting to JSON:", error);
  }
};
</script>
