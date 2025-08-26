<script setup>
import { ref, watch, onMounted } from 'vue'
import { format } from "prettier/standalone";
import pluginSql from "prettier-plugin-sql";
import hljs from 'highlight.js/lib/core';
import sqlLang from 'highlight.js/lib/languages/sql';
import 'highlight.js/styles/github.css';

hljs.registerLanguage('sql', sqlLang);

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
                parser: language.value, // Use the selected language as the parser
                plugins: [pluginSql],
                // sql-formatter options
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

watch([sql, language, keywordCase, tabWidth, linesBetweenQueries], () => {
    formatAndHighlight(sql.value);
});

onMounted(() => {
    formatAndHighlight(sql.value);
});

</script>

<template>
  <div class="container mt-4 mb-5">
    <h1 class="text-center mb-4">SQL Formatter</h1>
    <div class="row">
      <div class="col-12 mb-4">
        <div class="card shadow-sm">
          <div class="card-header">
            <h5 class="card-title mb-0">Parser Options</h5>
          </div>
          <div class="card-body">
            <div class="row">
              <!-- <div class="col-md-3 mb-3">
                <label for="language" class="form-label">Language</label>
                <select id="language" v-model="language" class="form-select">
                  <option v-for="lang in languageOptions" :key="lang" :value="lang">{{ lang }}</option>
                </select>
              </div> -->
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
                <input type="number" id="linesBetweenQueries" v-model.number="linesBetweenQueries" class="form-control" min="0">
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
    background-color: #f8f9fa;
}
</style>