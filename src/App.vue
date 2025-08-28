<script setup>
import { ref, watch, onMounted, nextTick } from 'vue'
import { format } from "prettier/standalone";
import pluginSql from "prettier-plugin-sql";
import { EditorView, basicSetup } from 'codemirror';
import { EditorState } from '@codemirror/state';
import { sql } from '@codemirror/lang-sql';
import axios from 'axios';

const response_data = ref();
const api_url = ref("")
const sql_ref = ref('SELECT * FROM users WHERE id = 1 AND name = \'John Doe\';');
const codeBlock = ref(null);
const sqlEditorRef = ref(null);
let editor;
let sqlEditor;
const tab = ref('all');
const isLoading = ref(false);

// Test Form Refs
const test_http_method = ref('get');
const test_columns = ref('SQL_TEXT1,SQL_TEXT2,SQL_TEXT3');
const test_sql_Language = ref('sql');
const test_api_url = ref('');


// Parser Options
const language = ref('sql');
const keywordCase = ref('upper');
const tabWidth = ref(2);
const linesBetweenQueries = ref(1);

const languageOptions = ['sql', 'bigquery', 'db2', 'hive', 'mariadb', 'mysql', 'n1ql', 'plsql', 'postgresql', 'redshift', 'singlestoredb', 'spark', 'sqlite', 'transactsql', 'trino', 'tsql'];
const keywordCaseOptions = ['upper', 'lower', 'preserve'];

const formatAndHighlight = async (sqlString) => {
  try {
    const formatted = await format(sqlString, {
      parser: "sql", // Use the selected language as the parser
      plugins: [pluginSql],
      // sql-formatter options
      language: language.value,
      keywordCase: keywordCase.value,
      tabWidth: tabWidth.value,
      linesBetweenQueries: linesBetweenQueries.value,
    });
    editor.dispatch({
      changes: { from: 0, to: editor.state.doc.length, insert: formatted },
    });
  } catch (error) {
    editor.dispatch({
      changes: { from: 0, to: editor.state.doc.length, insert: error.message },
    });
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

const fetchData = async (http_method, api_url, columns) => {
  if (!api_url) return;
  isLoading.value = true;
  try {
    response_data.value = await Get_Api_Data(http_method, api_url, columns);
    if (response_data.value.length > 0) {
      const newSql = response_data.value[0].join(" ");
      sql_ref.value = newSql;
      sqlEditor.dispatch({
        changes: { from: 0, to: sqlEditor.state.doc.length, insert: newSql }
      });
    }
  } finally {
    isLoading.value = false;
  }
};

const handleTestSubmit = () => {
  const columns = test_columns.value.split(',').map(s => s.trim());
  language.value = test_sql_Language.value;
  fetchData(test_http_method.value, test_api_url.value, columns);
};

watch(tab, async (newTab) => {
  await nextTick();
  if (newTab === 'enter' || newTab === 'all') {
    if (sqlEditor) {
      sqlEditor.dispatch({});
    }
  }
  if (newTab === 'formatted' || newTab === 'all') {
    if (editor) {
      editor.dispatch({});
    }
  }
});

watch([language, keywordCase, tabWidth, linesBetweenQueries], () => {
  formatAndHighlight(sql_ref.value);
});

onMounted(() => {
  const state = EditorState.create({
    doc: sql_ref.value,
    extensions: [basicSetup, sql()],
  });

  editor = new EditorView({
    state,
    parent: codeBlock.value,
  });

  const sqlEditorState = EditorState.create({
    doc: sql_ref.value,
    extensions: [
      basicSetup,
      sql(),
      EditorView.updateListener.of(update => {
        if (update.docChanged) {
          const newSql = update.state.doc.toString();
          sql_ref.value = newSql;
          formatAndHighlight(newSql);
        }
      })
    ],
  });

  sqlEditor = new EditorView({
    state: sqlEditorState,
    parent: sqlEditorRef.value
  });


  formatAndHighlight(sql_ref.value);

  // 取得 URL 的 query 參數
  const params = new URLSearchParams(window.location.search)
  // 取得 'url' 參數值（經過 encoding 的網址）
  const encodedUrl = params.get('api_url') || ""
  // 解碼後存入 api_url
  api_url.value = decodeURIComponent(encodedUrl)
  const columns_string = params.get('columns') || ""
  const columns = columns_string.split(",")
  const http_method = params.get('http_method') || "get"
  const sql_Language = params.get('sql_Language') || "sql"

  // Populate Test API form
  if (encodedUrl) {
    test_api_url.value = decodeURIComponent(encodedUrl);
  }
  if (columns_string) {
    test_columns.value = columns_string;
  }
  if (http_method) {
    test_http_method.value = http_method;
  }
  if (sql_Language) {
    test_sql_Language.value = sql_Language;
    language.value = sql_Language;
  }

  fetchData(http_method, api_url.value, columns);
});

//?http_method=get&columns=SQL_TEXT1,SQL_TEXT2,SQL_TEXT3&api_url=http%3A%2F%2Ftw100043809.corpnet.auo.com%2FData_Grage%2FAPI_Result_Data.ashx%3FAPI_NAME%3DORACLE_Connect_Users_SQL%26API_Group%3DArray_INT%255cUser_Query%255cIT%26SQL_ID%3D60rm7356y4wvw%26Data_Only%3Dtrue
</script>

<template>
  <!-- {{ response_data }} -->

  <div class="container mt-3 mb-3">
    <div class="title-div rounded ">
      <h1 class="text-center mb-4 ">
        <a href="./"> SQL Formatter</a>
      </h1>
    </div>
    <div class="d-flex justify-content-center">
      <small class="text-secondary">格式化SQL, 可以URL帶上API進行轉換, API 記得開CORS</small>
    </div>

    <div class="mb-3">
      <button class="btn btn-outline-info w-100" type="button" data-bs-toggle="collapse" data-bs-target="#queryguide">
        Query Guide
      </button>
    </div>

    <div class="collapse" id="queryguide">
      <div class="card card-body">
        <p>You can provide an API endpoint to fetch the SQL query dynamically. The API should return a JSON response.
          The following query parameters are supported:</p>
        <ul>
          <li><strong>http_method</strong>: The HTTP method to use for the API request (e.g., <code>get</code>,
            <code>post</code>).</li>
          <li><strong>columns</strong>: A comma-separated list of keys to extract from the JSON response. The values of
            these keys will be concatenated to form the SQL query.</li>
          <li><strong>sql_Language</strong>: The SQL dialect to use for formatting (e.g., <code>plsql</code>,
            <code>mysql</code>).</li>
          <li><strong>api_url</strong>: The URL of the API endpoint. This URL must be URL-encoded.</li>
        </ul>
        <p><strong>Example:</strong></p>
        <pre><code>?http_method=get&columns=SQL_TEXT1,SQL_TEXT2&sql_Language=plsql&api_url=ENCODED_URL_HERE</code></pre>
        
        <hr>

        <h5>Test API</h5>
         <!-- @submit.prevent="handleTestSubmit" -->
        <form action="." method="get" >
          <div class="mb-3">
            <label for="test_api_url" class="form-label">API URL</label>
            <input type="text" class="form-control" id="test_api_url" name="api_url" v-model="test_api_url">
          </div>
          <div class="mb-3">
            <label for="test_http_method" class="form-label">HTTP Method</label>
            <select class="form-select" id="test_http_method" name="http_method" v-model="test_http_method">
              <option value="get">GET</option>
              <option value="post">POST</option>
            </select>
          </div>
          <div class="mb-3">
            <label for="test_columns" class="form-label">Columns (comma-separated)</label>
            <input type="text" class="form-control" id="test_columns" name="columns" v-model="test_columns">
          </div>
          <div class="mb-3">
            <label for="test_sql_Language" class="form-label">SQL Language</label>
            <select class="form-select" id="test_sql_Language" name="sql_Language" v-model="test_sql_Language">
              <option v-for="lang in languageOptions" :key="lang" :value="lang">{{ lang }}</option>
            </select>
          </div>
          <button type="submit" class="btn btn-primary">Query Format</button>
        </form>
      </div>

      <div class="input-group mt-3 mb-3">
        <span class="input-group-text">API_URL</span>
        <input type="text" class="form-control" placeholder="api_url" :value="api_url">
      </div>

    </div>



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
    </div>

    <ul class="nav nav-tabs nav-justified">
      <li class="nav-item">
        <a class="nav-link" :class="{ active: tab === 'all' }" @click="tab = 'all'" href="#">ALL</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" :class="{ active: tab === 'enter' }" @click="tab = 'enter'" href="#">Enter SQL</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" :class="{ active: tab === 'formatted' }" @click="tab = 'formatted'" href="#">Formatted
          SQL</a>
      </li>
    </ul>
  </div>

  <div class="container-fluid">
    <div class="row">
      <div :class="{ 'col-md-6': tab === 'all', 'col-12': tab !== 'all' }" v-show="tab === 'enter' || tab === 'all'">
        <div class="card shadow-sm">
          <div class="card-header">
            <h5 class="card-title mb-0">Enter SQL</h5>
          </div>
          <div class="card-body">
            <div ref="sqlEditorRef"></div>
          </div>
        </div>
      </div>
      <div :class="{ 'col-md-6': tab === 'all', 'col-12': tab !== 'all' }"
        v-show="tab === 'formatted' || tab === 'all'">
        <div class="card shadow-sm position-relative">
          <div class="card-header">
            <h5 class="card-title mb-0">Formatted SQL</h5>
          </div>
          <div class="card-body bg-light">
            <div v-if="isLoading" class="loading-overlay">
              <div class="spinner-border text-primary" role="status">
                <span class="visually-hidden">Loading...</span>
              </div>
            </div>
            <div ref="codeBlock"></div>
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

.cm-editor {
  height: 100%;
  font-size: 16px;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(255, 255, 255, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
}
</style>