<script setup>
import { ref, watch, onMounted } from 'vue'
import { format } from "prettier/standalone";
import pluginSql from "prettier-plugin-sql";
import hljs from 'highlight.js/lib/core';
import sqlLang from 'highlight.js/lib/languages/sql';
import 'highlight.js/styles/github.css';
import axios from 'axios';

hljs.registerLanguage('sql', sqlLang);



const response_data = ref();
const api_url = ref("")
const sql = ref('SELECT * FROM users WHERE id = 1 AND name = \'John Doe\';');
const codeBlock = ref(null);

// Parser Options
const language = ref('sql');
const keywordCase = ref('upper');
const tabWidth = ref(2);
const linesBetweenQueries = ref(1);

const languageOptions = ['sql', 'bigquery', 'db2', 'hive', 'mariadb', 'mysql', 'n1ql', 'plsql', 'postgresql', 'redshift', 'singlestoredb', 'spark', 'sqlite', 'transactsql', 'trino', 'tsql'];
const keywordCaseOptions = ['upper', 'lower', 'preserve'];

const formatAndHighlight = async (sqlString) => {
  if (codeBlock.value) {
    try {
      const formatted = await format(sqlString, {
        parser: "sql", // Use the selected language as the parser
        plugins: [pluginSql],
        // sql-formatter options
        language:language.value,
        keywordCase: keywordCase.value,
        tabWidth: tabWidth.value,
        linesBetweenQueries: linesBetweenQueries.value,
      });
      codeBlock.value.innerHTML = hljs.highlight(formatted, { language: 'sql' }).value;
    } catch (error) {
      codeBlock.value.textContent = error.message;
    }
  }
}




const Get_Api_Data = async (http_method, api_url, columns) => {
  console.log(http_method)
  console.log(columns)
  console.log(api_url)
  try {
    // 發送請求
    const response = await axios({
      method: http_method,
      url: api_url
    });

    const data = response.data;
    let Return_list = [];
    // 假設 API 返回的資料是一個物件或陣列
    // 如果是陣列（多筆資料），對每個元素做篩選
    if (Array.isArray(data)) {
      return data.map(item => {
        const result = {};
        columns.forEach(key => {
          if (item.hasOwnProperty(key)) {
            //result[key] = item[key];

            console.log(item[key])
            Return_list.push(item[key])

          }
        });
        return Return_list;
      });
    } else if (typeof data === 'object' && data !== null) {
      // 如果返回的是一個物件，提取出該物件中 columns 定義的 key
      const result = {};
      columns.forEach(key => {
        if (data.hasOwnProperty(key)) {
          result[key] = data[key];
          console.log(data[key])
          Return_list.push(data[key])
        }
      });
      return Return_list;
    } else {
      // 其他情況：直接回傳
      return Return_list;
    }
  } catch (error) {
    console.error('API 請求錯誤:', error);
    throw error;
  }
};


const fetchData = async (http_method, api_url, columns ) => {


  response_data.value = await Get_Api_Data(http_method, api_url, columns);
  if (response_data.value.length > 0) sql.value = response_data.value[0].join(" ")
  //sql
};

watch([sql, language, keywordCase, tabWidth, linesBetweenQueries], () => {
  formatAndHighlight(sql.value);
});

onMounted(() => {
  formatAndHighlight(sql.value);

  // 取得 URL 的 query 參數
  const params = new URLSearchParams(window.location.search)
  // 取得 'url' 參數值（經過 encoding 的網址）
  const encodedUrl = params.get('api_url') || ""
  // 解碼後存入 api_url
  api_url.value = decodeURIComponent(encodedUrl)
  const columns_string = params.get('columns') || ""
  const columns = columns_string.split(",")
  const http_method = params.get('http_method') || ""
  const sql_Language = params.get('sql_Language') || ""

  language.value = sql_Language;
  fetchData(http_method, api_url.value, columns );
});

//?http_method=get&columns=SQL_TEXT1,SQL_TEXT2,SQL_TEXT3&api_url=http%3A%2F%2Ftw100043809.corpnet.auo.com%2FData_Grage%2FAPI_Result_Data.ashx%3FAPI_NAME%3DORACLE_Connect_Users_SQL%26API_Group%3DArray_INT%255cUser_Query%255cIT%26SQL_ID%3D60rm7356y4wvw%26Data_Only%3Dtrue
</script>

<template>
  <!-- {{ response_data }} -->

  <div class="container mt-4 mb-5">
    <div class="title-div rounded ">
      <h1 class="text-center mb-4 "> 
        <a href="./"> SQL Formatter</a> </h1>
    </div>
    <div class="d-flex justify-content-center">
      <small class="text-secondary">格式化SQL, 可以URL帶上API進行轉換, API 記得開CORS</small>
    </div>

    <div class="input-group">
    <span class="input-group-text">Simple</span>
    <input type="text" class="form-control" placeholder="Query String" value="?http_method=get&columns=SQL_TEXT1,SQL_TEXT2,SQL_TEXT3,SQL_TEXT4&sql_Language=plsql&api_url={Encode Url, Return Json}">
  </div>

  <div class="input-group">
    <span class="input-group-text">API_URL</span>
    <input type="text" class="form-control" placeholder="api_url" :value="api_url">
  </div>

    
    <hr>
    </hr>

    <div class="row">
      <div class="col-12 mb-4">
        <div class="card shadow-sm">
          <div class="card-header">
            <h5 class="card-title mb-0">Parser Options</h5>
          </div>
          <div class="card-body">
            <div class="row">
              <div class="col-md-3 mb-3">
                <label for="language" class="form-label">Language</label>
                <select id="language" v-model="language" class="form-select">
                  <option v-for="lang in languageOptions" :key="lang" :value="lang">{{ lang }}</option>
                </select>
              </div> 
              <div class="col-md-3 mb-3">
                <label for="keywordCase" class="form-label">Keyword Case</label>
                <select id="keywordCase" v-model="keywordCase" class="form-select">
                  <option v-for="kc in keywordCaseOptions" :key="kc" :value="kc">{{ kc }}</option>
                </select>
              </div>
              <div class="col-md-3 mb-3">
                <label for="tabWidth" class="form-label">Tab Width</label>
                <input type="number" id="tabWidth" v-model.number="tabWidth" class="form-control" min="0">
              </div>
              <div class="col-md-3 mb-3">
                <label for="linesBetweenQueries" class="form-label">Lines Between Queries</label>
                <input type="number" id="linesBetweenQueries" v-model.number="linesBetweenQueries" class="form-control"
                  min="0">
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="col-12 mb-4">
        <div class="card shadow-sm">
          <div class="card-header">
            <h5 class="card-title mb-0">Enter SQL</h5>
          </div>
          <div class="card-body">
            <textarea id="sql-input" v-model="sql" class="form-control" rows="5"></textarea>
          </div>
        </div>
      </div>
      <div class="col-12">
        <div class="card shadow-sm">
          <div class="card-header">
            <h5 class="card-title mb-0">Formatted SQL</h5>
          </div>
          <div class="card-body bg-light">
            <pre class="m-0"><code ref="codeBlock" class="sql"></code></pre>
          </div>
        </div>
      </div>
    </div>
  </div>

</template>

<style>
body {
  background-color: #e9f6f0;
}


.title-div {
   /* background: linear-gradient(to right, red, orange, yellow, green, blue, indigo, violet); */
  /* color: ; */
} 
</style>