<script setup>

import { ref, computed, watch, onMounted, nextTick } from 'vue';
import {
  BarChart, Activity, FileText, AlertCircle, Play,
  Clipboard, Check, Sparkles, Bot, Loader2, TrendingUp, Grid
} from 'lucide-vue-next';

// --- 狀態管理 (State) ---
const defaultData = `{
  "0": {
    "precision": 0.85,
    "recall": 0.91,
    "f1-score": 0.88,
    "support": 150
  },
  "1": {
    "precision": 0.78,
    "recall": 0.65,
    "f1-score": 0.71,
    "support": 90
  },
  "2": {
    "precision": 0.92,
    "recall": 0.94,
    "f1-score": 0.93,
    "support": 120
  },
  "accuracy": 0.86,
  "macro avg": {
    "precision": 0.85,
    "recall": 0.83,
    "f1-score": 0.84,
    "support": 360
  },
  "weighted avg": {
    "precision": 0.86,
    "recall": 0.86,
    "f1-score": 0.85,
    "support": 360
  }
}`;

const defaultCm = `[
  [136, 10, 4],
  [2, 58, 30],
  [5, 10, 105]
]`;

const inputText = ref(defaultData);
const cmInputText = ref(defaultCm);
const parsedData = ref(null);
const parsedCm = ref(null);
const error = ref(null);
const cmError = ref(null);
const copied = ref(false);
const aucScore = ref(0.88);

// AI 相關
const aiAnalysis = ref(null);
const isAiLoading = ref(false);
const aiError = ref(null);

// --- 邏輯處理 (Logic) ---

// 數字動畫 (簡易版 Tween)
const tweenedAccuracy = ref(0);
const tweenedAuc = ref(0);

watch(() => parsedData.value?.accuracy, (newVal) => {
  if (newVal != null) {
    animateValue(tweenedAccuracy, 0, newVal * 100, 1500);
  }
});

watch(aucScore, (newVal) => {
  animateValue(tweenedAuc, 0, newVal, 1000);
});

function animateValue(refVal, start, end, duration) {
  let startTimestamp = null;
  const step = (timestamp) => {
    if (!startTimestamp) startTimestamp = timestamp;
    const progress = Math.min((timestamp - startTimestamp) / duration, 1);
    const easeOut = 1 - Math.pow(1 - progress, 4); // EaseOutQuart
    refVal.value = start + (end - start) * easeOut;
    if (progress < 1) {
      window.requestAnimationFrame(step);
    }
  };
  window.requestAnimationFrame(step);
}

// 資料解析主函式
const processData = () => {
  error.value = null;
  cmError.value = null;
  aiAnalysis.value = null;
  aiError.value = null;

  // 1. 解析 Report
  try {
    let text = inputText.value;
    // 簡單的 Python dict 轉 JSON 處理
    let jsonString = text.replace(/'/g, '"').replace(/None/g, 'null').replace(/True/g, 'true').replace(/False/g, 'false');
    const data = JSON.parse(jsonString);

    const classes = [];
    let accuracy = null;
    const summaries = [];

    // 自動讀取 auc
    if (data.roc_auc || data.auc) {
      aucScore.value = parseFloat(data.roc_auc || data.auc);
    }

    Object.keys(data).forEach(key => {
      if (key === 'accuracy') {
        accuracy = data[key];
      } else if (key === 'macro avg' || key === 'weighted avg') {
        summaries.push({ name: key, ...data[key] });
      } else if (key !== 'roc_auc' && key !== 'auc') {
        classes.push({ name: key, ...data[key] });
      }
    });

    parsedData.value = { classes, accuracy, summaries };
  } catch (e) {
    // 備用文字解析器 (Fallback)
    try {
      const lines = inputText.value.trim().split('\n');
      const classes = [];
      const summaries = [];
      let accuracy = null;
      let isParsingClasses = true;

      lines.forEach(line => {
        if (!line.trim() || line.includes('precision') || line.includes('---')) return;
        const parts = line.trim().split(/\s+/);

        if (line.includes('accuracy')) {
          const accVal = parts.find(p => !isNaN(parseFloat(p)) && parseFloat(p) <= 1);
          if (accVal) accuracy = parseFloat(accVal);
          isParsingClasses = false;
          return;
        }

        if (parts.length >= 4) {
          const support = parseInt(parts[parts.length - 1]);
          const f1 = parseFloat(parts[parts.length - 2]);
          const recall = parseFloat(parts[parts.length - 3]);
          const precision = parseFloat(parts[parts.length - 4]);
          const name = parts.slice(0, parts.length - 4).join(" ");
          const item = { name, precision, recall, "f1-score": f1, support };

          if (name === 'macro avg' || name === 'weighted avg') {
            summaries.push(item);
          } else if (isParsingClasses && name !== 'accuracy') {
            classes.push(item);
          }
        }
      });

      if (classes.length === 0 && !accuracy) throw new Error("無法解析");
      parsedData.value = { classes, accuracy, summaries };
    } catch (err) {
      error.value = "無法解析數據。請確保您貼上的是 'output_dict=True' 的 JSON 格式，或標準的文字輸出。";
      parsedData.value = null;
    }
  }

  // 2. 解析混淆矩陣
  try {
    if (cmInputText.value.trim()) {
      const cm = JSON.parse(cmInputText.value);
      if (Array.isArray(cm) && Array.isArray(cm[0])) {
        parsedCm.value = cm;
      } else {
        cmError.value = "格式錯誤：必須是二維陣列";
        parsedCm.value = null;
      }
    } else {
      parsedCm.value = null;
    }
  } catch (e) {
    cmError.value = "混淆矩陣無法解析 (請使用標準 JSON 陣列格式)";
    parsedCm.value = null;
  }
};

// 顏色映射
const getColorClass = (value) => {
  if (value === undefined || value === null) return 'bg-slate-700 text-slate-400';
  if (value < 0.5) return 'bg-rose-600 text-white shadow-[0_0_10px_rgba(225,29,72,0.4)]';
  if (value < 0.7) return 'bg-orange-500 text-white shadow-[0_0_10px_rgba(249,115,22,0.4)]';
  if (value < 0.8) return 'bg-amber-400 text-slate-900 font-bold shadow-[0_0_10px_rgba(251,191,36,0.4)]';
  if (value < 0.9) return 'bg-cyan-400 text-slate-900 font-bold shadow-[0_0_10px_rgba(34,211,238,0.4)]';
  return 'bg-violet-500 text-white font-bold shadow-[0_0_15px_rgba(139,92,246,0.6)] border border-violet-400';
};

// 最大 Support 計算
const maxSupport = computed(() => {
  if (!parsedData.value) return 100;
  const classMax = Math.max(...parsedData.value.classes.map(c => c.support));
  const summaryMax = Math.max(...parsedData.value.summaries.map(c => c.support));
  return Math.max(classMax, summaryMax);
});

// ROC 曲線計算
const rocData = computed(() => {
  const val = aucScore.value;
  const p = val >= 1 ? 0.001 : (1 / Math.max(val, 0.01)) - 1;
  const points = [];
  const resolution = 50;
  for (let i = 0; i <= resolution; i++) {
    const x = i / resolution;
    const y = Math.pow(x, p);
    const svgX = 40 + x * 240;
    const svgY = 180 - y * 160;
    points.push(`${svgX},${svgY}`);
  }
  const linePath = points.join(" ");
  const areaPath = `40,180 ${linePath} 280,180 Z`;
  return { linePath, areaPath };
});

// 混淆矩陣計算
const cmDisplayData = computed(() => {
  if (!parsedCm.value) return null;
  const matrix = parsedCm.value;
  const flatValues = matrix.flat();
  const maxValue = Math.max(...flatValues);
  const size = matrix.length;

  // 嘗試從解析的資料中獲取標籤，否則使用數字
  const labels = parsedData.value && parsedData.value.classes.length === size
    ? parsedData.value.classes.map(c => c.name)
    : Array.from({ length: size }, (_, i) => i.toString());

  return { matrix, maxValue, labels };
});

// Gemini API
const analyzeWithGemini = async () => {
  if (!parsedData.value) return;
  isAiLoading.value = true;
  aiError.value = null;
  aiAnalysis.value = null;

  const apiKey = ""; // The execution environment provides the key at runtime.
  const prompt = `
    請擔任一位資深資料科學家與機器學習專家。
    請根據以下的 Scikit-learn Classification Report 數據進行深入分析（請使用繁體中文回答，語氣專業但易懂）：
    
    數據內容 JSON:
    ${JSON.stringify(parsedData.value)}

    ROC-AUC 分數：${aucScore.value}。

    混淆矩陣數據:
    ${parsedCm.value ? JSON.stringify(parsedCm.value) : "未提供"}

    請包含以下部分：
    1. **整體評估**：評論準確率、ROC-AUC 以及混淆矩陣顯示的分類傾向。
    2. **混淆矩陣分析**：指出模型最容易混淆哪兩個類別？
    3. **亮點與弱點**：指出表現最好的類別與表現最差的類別。
    4. **具體改進建議**：根據上述分析，給出具體的改進方向。
    請使用 Markdown 格式輸出。
  `;

  try {
    const response = await fetch(
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ contents: [{ parts: [{ text: prompt }] }] }),
      }
    );
    if (!response.ok) throw new Error("API failed");
    const result = await response.json();
    aiAnalysis.value = result.candidates?.[0]?.content?.parts?.[0]?.text;
  } catch (err) {
    aiError.value = "AI 分析暫時無法使用，請稍後再試。";
  } finally {
    isAiLoading.value = false;
  }
};

const copyToClipboard = () => {
  navigator.clipboard.writeText(inputText.value);
  copied.value = true;
  setTimeout(() => copied.value = false, 2000);
};

const renderMarkdown = (text) => {
  if (!text) return [];
  return text.split('\n').filter(line => line.trim() !== '');
};

// 初始化
onMounted(() => {
  processData();
});
</script>

<template>
  <div
    class="min-h-screen bg-slate-950 p-4 md:p-8 font-sans text-slate-200 selection:bg-violet-500 selection:text-white">
    <div class="max-w-6xl mx-auto space-y-6">

      <!-- Header -->
      <div
        class="flex flex-col md:flex-row justify-between items-start md:items-center bg-slate-900 p-6 rounded-xl shadow-lg border border-slate-800">
        <div>
          <h1 class="text-2xl font-black flex items-center gap-3 text-white tracking-tight">
            <div class="p-2 bg-violet-600 rounded-lg shadow-lg shadow-violet-900/50">
              <Activity class="text-white" :size="24" />
            </div>
            Model Performance Report
          </h1>
          <p class="text-slate-400 mt-2 text-sm font-medium">
            高對比度互動式報表 / Interactive Classification Metrics (Vue Version)
          </p>
        </div>
        <div class="mt-4 md:mt-0 flex flex-wrap gap-2">
          <button @click="analyzeWithGemini" :disabled="!parsedData || isAiLoading"
            class="px-5 py-2.5 rounded-lg text-sm font-bold transition-all shadow-md flex items-center gap-2"
            :class="!parsedData ? 'bg-slate-800 text-slate-500 cursor-not-allowed' : isAiLoading ? 'bg-violet-800 text-violet-200 cursor-wait' : 'bg-gradient-to-r from-violet-600 to-fuchsia-600 hover:from-violet-500 hover:to-fuchsia-500 text-white shadow-violet-900/50'">
            <Loader2 v-if="isAiLoading" :size="16" class="animate-spin" />
            <Sparkles v-else :size="16" />
            {{ isAiLoading ? 'AI 分析中...' : '✨ AI 智能洞察' }}
          </button>
          <button @click="() => { inputText = defaultData; cmInputText = defaultCm; setTimeout(processData, 0); }"
            class="px-5 py-2.5 bg-slate-800 hover:bg-slate-700 text-slate-300 border border-slate-700 hover:border-slate-600 rounded-lg text-sm font-bold transition-all shadow-md">
            載入範例數據
          </button>
        </div>
      </div>

      <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">

        <!-- Left Column: Input -->
        <div class="lg:col-span-1 space-y-4">

          <!-- Report Input -->
          <div
            class="bg-slate-900 rounded-xl shadow-lg border border-slate-800 overflow-hidden flex flex-col h-[450px]">
            <div class="p-4 border-b border-slate-800 bg-slate-900/50 flex justify-between items-center">
              <span class="font-bold text-slate-300 flex items-center gap-2">
                <FileText :size="18" class="text-violet-400" />
                Report (JSON)
              </span>
              <button @click="processData"
                class="p-2 bg-violet-600 hover:bg-violet-500 text-white rounded-lg transition-all shadow-lg shadow-violet-900/30">
                <Play :size="16" fill="currentColor" />
              </button>
            </div>
            <div class="relative flex-1 bg-slate-950">
              <textarea
                class="w-full h-full p-4 font-mono text-sm resize-none focus:outline-none bg-slate-950 text-slate-300 placeholder-slate-600 border-none"
                placeholder="Paste Python output here..." v-model="inputText" spellcheck="false"></textarea>
              <button @click="copyToClipboard"
                class="absolute top-2 right-2 p-2 bg-slate-800/80 backdrop-blur border border-slate-700 rounded-md text-slate-400 hover:text-white transition-colors">
                <Check v-if="copied" :size="14" class="text-green-400" />
                <Clipboard v-else :size="14" />
              </button>
            </div>
            <div v-if="error"
              class="p-3 bg-red-900/20 text-red-400 text-xs border-t border-red-900/50 flex items-start gap-2">
              <AlertCircle :size="14" class="mt-0.5 shrink-0" />
              {{ error }}
            </div>
          </div>

          <!-- ROC-AUC Control -->
          <div class="p-4 bg-slate-900 rounded-xl border border-slate-800">
            <div class="flex justify-between items-center mb-2">
              <label class="text-xs font-bold text-slate-400 uppercase tracking-widest flex items-center gap-2">
                <TrendingUp :size="14" class="text-violet-500" />
                ROC-AUC 分數
              </label>
              <span class="text-sm font-bold text-violet-400">{{ aucScore }}</span>
            </div>
            <input type="range" min="0.5" max="1.0" step="0.01" v-model.number="aucScore"
              class="w-full h-2 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-violet-500 hover:accent-violet-400" />
          </div>

          <!-- Confusion Matrix Input -->
          <div
            class="bg-slate-900 rounded-xl shadow-lg border border-slate-800 overflow-hidden flex flex-col h-[200px]">
            <div class="p-3 border-b border-slate-800 bg-slate-900/50 flex justify-between items-center">
              <span class="font-bold text-slate-300 flex items-center gap-2 text-sm">
                <Grid :size="16" class="text-violet-400" />
                混淆矩陣 (JSON)
              </span>
            </div>
            <div class="relative flex-1 bg-slate-950">
              <textarea
                class="w-full h-full p-4 font-mono text-sm resize-none focus:outline-none bg-slate-950 text-slate-300 placeholder-slate-600 border-none"
                placeholder="e.g. [[10, 2], [3, 20]]" v-model="cmInputText" spellcheck="false"></textarea>
            </div>
            <div v-if="cmError" class="p-2 bg-red-900/20 text-red-400 text-xs border-t border-red-900/50">
              {{ cmError }}
            </div>
          </div>

        </div>

        <!-- Right Column: Visualization -->
        <div class="lg:col-span-2 space-y-6">

          <div v-if="!parsedData && !error"
            class="h-full flex flex-col items-center justify-center text-slate-600 bg-slate-900 rounded-xl border-2 border-dashed border-slate-800 min-h-[400px]">
            <BarChart :size="64" class="mb-4 opacity-20" />
            <p class="font-medium">等待數據輸入...</p>
          </div>

          <template v-else-if="parsedData">

            <!-- AI Analysis -->
            <div v-if="aiAnalysis || isAiLoading || aiError"
              class="bg-gradient-to-br from-indigo-950 to-slate-900 p-6 rounded-xl shadow-lg border border-indigo-800/50 relative overflow-hidden">
              <div
                class="absolute top-0 right-0 w-64 h-64 bg-violet-600/5 rounded-full blur-3xl -mr-20 -mt-20 pointer-events-none">
              </div>
              <h3 class="text-violet-300 font-bold flex items-center gap-2 mb-4">
                <Bot :size="20" :class="{ 'animate-bounce': isAiLoading }" /> Gemini 智能分析報告
              </h3>

              <div v-if="isAiLoading" class="space-y-3 animate-pulse">
                <div class="h-2 bg-slate-700 rounded w-3/4"></div>
                <div class="h-2 bg-slate-700 rounded w-full"></div>
              </div>

              <div v-if="aiError" class="text-rose-400 text-sm bg-rose-950/30 p-3 rounded-lg border border-rose-900/50">
                {{ aiError }}</div>

              <div v-if="aiAnalysis" class="text-sm leading-relaxed text-slate-300 space-y-2">
                <div v-for="(line, i) in renderMarkdown(aiAnalysis)" :key="i">
                  <h4 v-if="line.startsWith('### ')" class="text-violet-300 font-bold text-lg mt-4 mb-2">{{
                    line.replace('### ', '') }}</h4>
                  <li v-else-if="line.startsWith('- ')" class="text-slate-300 ml-4 mb-1 list-disc">{{ line.replace('- ',
                    '').replace(/\*\*/g, '') }}</li>
                  <p v-else class="text-slate-400 mb-1 font-medium">{{ line.replace(/\*\*/g, '') }}</p>
                </div>
              </div>
            </div>

            <!-- Accuracy Card -->
            <div v-if="parsedData.accuracy !== null"
              class="bg-gradient-to-r from-slate-900 to-slate-800 p-6 rounded-xl shadow-lg border border-slate-800 flex items-center justify-between relative overflow-hidden">
              <div
                class="absolute top-0 right-0 w-32 h-32 bg-violet-600/10 rounded-full blur-3xl -mr-10 -mt-10 pointer-events-none">
              </div>
              <div class="z-10">
                <h3 class="text-slate-400 text-xs font-bold uppercase tracking-widest mb-1">整體準確率 (Accuracy)</h3>
                <div class="flex items-baseline gap-2">
                  <span class="text-4xl font-black text-white tracking-tighter">{{ tweenedAccuracy.toFixed(1) }}</span>
                  <span class="text-lg text-violet-400">%</span>
                </div>
              </div>
              <div
                class="h-20 w-20 rounded-full border-4 border-slate-700 flex items-center justify-center bg-slate-800 shadow-inner z-10 relative">
                <div class="absolute inset-0 rounded-full opacity-20"
                  :class="parsedData.accuracy > 0.8 ? 'bg-violet-500 animate-pulse' : 'bg-slate-500'"></div>
                <div class="text-2xl font-bold text-white relative z-10">{{ parsedData.accuracy.toFixed(2) }}</div>
              </div>
            </div>

            <!-- Main Metrics Table -->
            <div class="bg-slate-900 rounded-xl shadow-lg border border-slate-800 overflow-hidden">
              <div class="p-5 border-b border-slate-800 bg-slate-900/50 flex justify-between items-center">
                <h2 class="font-bold text-slate-200 flex items-center gap-2">
                  <span class="w-2 h-6 bg-violet-500 rounded-sm inline-block"></span>
                  詳細表現 (Class Metrics)
                </h2>
              </div>
              <div class="p-6">
                <!-- Headers -->
                <div class="grid grid-cols-12 gap-y-2 gap-x-4 items-center mb-4 text-center">
                  <div class="col-span-3"></div>
                  <div class="col-span-2 text-xs font-bold text-slate-500 uppercase">精確率</div>
                  <div class="col-span-2 text-xs font-bold text-slate-500 uppercase">召回率</div>
                  <div class="col-span-2 text-xs font-bold text-slate-500 uppercase">F1 分數</div>
                  <div class="col-span-3 text-xs font-bold text-slate-500 uppercase">樣本數</div>
                </div>

                <!-- Rows -->
                <div class="space-y-4">
                  <div v-for="(item, idx) in parsedData.classes" :key="idx"
                    class="grid grid-cols-12 gap-4 items-center group hover:bg-slate-800 p-3 rounded-xl transition-all border border-transparent hover:border-slate-700">
                    <div
                      class="col-span-3 font-bold text-slate-300 truncate pl-2 border-l-2 border-transparent group-hover:border-violet-500 transition-all"
                      :title="item.name">{{ item.name }}</div>

                    <div class="col-span-2 flex justify-center">
                      <div
                        class="flex flex-col items-center justify-center p-3 rounded-lg transition-all transform hover:scale-105"
                        :class="getColorClass(item.precision)">
                        <span class="text-lg tracking-tight font-bold">{{ item.precision.toFixed(2) }}</span>
                      </div>
                    </div>
                    <div class="col-span-2 flex justify-center">
                      <div
                        class="flex flex-col items-center justify-center p-3 rounded-lg transition-all transform hover:scale-105"
                        :class="getColorClass(item.recall)">
                        <span class="text-lg tracking-tight font-bold">{{ item.recall.toFixed(2) }}</span>
                      </div>
                    </div>
                    <div class="col-span-2 flex justify-center">
                      <div
                        class="flex flex-col items-center justify-center p-3 rounded-lg transition-all transform hover:scale-105"
                        :class="getColorClass(item['f1-score'])">
                        <span class="text-lg tracking-tight font-bold">{{ item['f1-score'].toFixed(2) }}</span>
                      </div>
                    </div>

                    <div class="col-span-3">
                      <div
                        class="w-full h-8 bg-slate-800 rounded-lg overflow-hidden relative flex items-center border border-slate-700">
                        <div
                          class="h-full bg-gradient-to-r from-blue-600 to-indigo-500 absolute left-0 top-0 transition-all duration-500"
                          :style="{ width: `${Math.min((item.support / maxSupport) * 100, 100)}%` }"></div>
                        <span class="z-10 w-full text-center text-sm font-bold text-slate-200 drop-shadow-md">{{
                          item.support }}</span>
                      </div>
                    </div>
                  </div>
                </div>

                <div class="my-8 border-t border-slate-800"></div>

                <!-- ROC Chart -->
                <div
                  class="bg-slate-800/50 p-6 rounded-xl border border-slate-700 relative overflow-hidden flex flex-col items-center">
                  <div class="absolute top-0 right-0 p-4 opacity-10">
                    <TrendingUp :size="120" class="text-violet-500" />
                  </div>
                  <div class="w-full flex justify-between items-end mb-4 z-10 relative">
                    <div>
                      <h3 class="text-slate-300 font-bold uppercase tracking-widest text-sm">ROC 曲線模擬圖</h3>
                      <p class="text-xs text-slate-500 mt-1">基於 AUC 分數的近似曲線</p>
                    </div>
                    <div class="text-right">
                      <div class="text-xs text-slate-400 font-bold uppercase mb-1">Area Under Curve</div>
                      <div class="text-3xl font-black text-violet-400 flex items-baseline justify-end gap-1">
                        <span class="text-lg text-slate-500 mr-1">AUC =</span>
                        <span>{{ tweenedAuc.toFixed(2) }}</span>
                      </div>
                    </div>
                  </div>
                  <div class="w-full h-64 relative z-10">
                    <svg viewBox="0 0 320 220" class="w-full h-full drop-shadow-xl">
                      <line x1="40" y1="20" x2="40" y2="180" stroke="#475569" stroke-width="2" />
                      <line x1="40" y1="180" x2="280" y2="180" stroke="#475569" stroke-width="2" />
                      <line x1="40" y1="180" x2="280" y2="20" stroke="#64748b" stroke-width="2"
                        stroke-dasharray="5,5" />
                      <text x="160" y="110" text-anchor="middle" fill="#64748b" font-size="10"
                        transform="rotate(-33 160,110)">Random Guess (0.5)</text>

                      <defs>
                        <linearGradient id="gradientRoc" x1="0" y1="0" x2="0" y2="1">
                          <stop offset="0%" stop-color="#8b5cf6" stop-opacity="0.8" />
                          <stop offset="100%" stop-color="#8b5cf6" stop-opacity="0" />
                        </linearGradient>
                      </defs>
                      <path :d="rocData.areaPath" fill="url(#gradientRoc)" opacity="0.4" />
                      <polyline :points="rocData.linePath" fill="none" stroke="#a78bfa" stroke-width="3"
                        stroke-linecap="round" stroke-linejoin="round" />

                      <text x="160" y="210" text-anchor="middle" fill="#94a3b8" font-size="12"
                        font-weight="bold">FPR</text>
                      <text x="15" y="100" text-anchor="middle" fill="#94a3b8" font-size="12" font-weight="bold"
                        transform="rotate(-90 15,100)">TPR</text>
                      <text x="35" y="185" text-anchor="end" fill="#64748b" font-size="10">0.0</text>
                      <text x="35" y="20" text-anchor="end" fill="#64748b" font-size="10">1.0</text>
                      <text x="40" y="195" text-anchor="middle" fill="#64748b" font-size="10">0.0</text>
                      <text x="280" y="195" text-anchor="middle" fill="#64748b" font-size="10">1.0</text>
                    </svg>
                  </div>
                </div>

              </div>
            </div>

            <!-- Confusion Matrix Card -->
            <div v-if="cmDisplayData"
              class="bg-slate-900 rounded-xl shadow-lg border border-slate-800 overflow-hidden flex flex-col">
              <div class="p-5 border-b border-slate-800 bg-slate-900/50 flex justify-between items-center">
                <h2 class="font-bold text-slate-200 flex items-center gap-2">
                  <Grid :size="18" class="text-violet-400" />
                  混淆矩陣 (Confusion Matrix)
                </h2>
              </div>
              <div class="p-6 overflow-x-auto">
                <div class="min-w-[300px] flex flex-col items-center">
                  <div class="text-slate-400 text-xs font-bold uppercase tracking-widest mb-2">Predicted Label</div>
                  <div class="flex">
                    <div class="flex items-center justify-center mr-2">
                      <span
                        class="text-slate-400 text-xs font-bold uppercase tracking-widest -rotate-90 whitespace-nowrap">True
                        Label</span>
                    </div>
                    <div class="flex flex-col gap-1">
                      <!-- Headers -->
                      <div class="flex gap-1 ml-8">
                        <div v-for="(label, i) in cmDisplayData.labels" :key="i"
                          class="w-16 text-center text-xs text-slate-400 font-mono truncate" :title="label">{{ label }}
                        </div>
                      </div>
                      <!-- Matrix -->
                      <div v-for="(row, i) in cmDisplayData.matrix" :key="i" class="flex gap-1">
                        <div class="w-8 flex items-center justify-end pr-2 text-xs text-slate-400 font-mono truncate"
                          :title="cmDisplayData.labels[i]">{{ cmDisplayData.labels[i] }}</div>

                        <div v-for="(val, j) in row" :key="j"
                          class="w-16 h-12 flex items-center justify-center rounded text-sm font-bold transition-transform hover:scale-110 relative group"
                          :class="i === j ? 'bg-violet-600 text-white' : (val === 0 ? 'bg-slate-800 text-slate-600' : 'bg-rose-600 text-white')"
                          :style="{
                            opacity: i === j ? Math.max(0.2, val / cmDisplayData.maxValue * 0.9 + 0.1) : (val === 0 ? 1 : Math.max(0.4, val / cmDisplayData.maxValue)),
                            border: i === j ? '1px solid rgba(139, 92, 246, 0.5)' : (val > 0 ? '1px solid rgba(225, 29, 72, 0.3)' : 'none')
                          }"
                          :title="`True: ${cmDisplayData.labels[i]}, Pred: ${cmDisplayData.labels[j]}, Count: ${val}`">
                          <span class="z-10 relative drop-shadow-md opacity-100">{{ val }}</span>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>

          </template>

        </div>
      </div>
    </div>
  </div>
</template>