<script setup lang="ts">
import { ref, computed } from 'vue';
import { CreateMLCEngine, type MLCEngine } from "@mlc-ai/web-llm";
import { 
  NConfigProvider, 
  NGlobalStyle, 
  NCard, 
  NInput, 
  NButton, 
  NProgress, 
  NScrollbar, 
  NBadge, 
  NGrid, 
  NGridItem, 
  NSpace, 
  NStatistic,
  NText,
  zhTW,
  dateZhTW,
  NResult
} from 'naive-ui';

import { Line } from 'vue-chartjs'
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler
} from 'chart.js'

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler
)

// 狀態變數
const selectedModel = "Qwen2.5-0.5B-Instruct-q4f16_1-MLC"; 

let engineInstance: MLCEngine | null = null;
const isLoaded = ref(false);
const isInitializing = ref(false);
const isWorking = ref(false);
const statusText = ref("待命");
const progress = ref(0);
const webGPUAvailable = ref(true);

if (typeof navigator !== "undefined" && !("gpu" in navigator)) {
  webGPUAvailable.value = false;
  statusText.value = "不支援 WebGPU";
}

// 數據狀態
const stockId = ref("2330");
const priceHistory = ref<any[]>([]);
const indexHistory = ref<any[]>([]);
const newsList = ref<any[]>([]);
const latestPrice = ref<any>(null);

// 圖表數據
const chartData = ref<any>({ labels: [], datasets: [] });
const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  interaction: { mode: 'index', intersect: false },
  scales: {
    y: { display: true, position: 'left', grid: { color: 'rgba(0,0,0,0.03)' }, ticks: { font: { size: 9 } } },
    y1: { display: true, position: 'right', grid: { drawOnChartArea: false }, ticks: { font: { size: 9 } } },
    x: { grid: { display: false }, ticks: { font: { size: 9 } } }
  },
  plugins: {
    legend: { position: 'top', align: 'end', labels: { boxWidth: 6, font: { size: 9 } } },
    tooltip: { backgroundColor: 'rgba(255,255,255,0.9)', titleColor: '#000', bodyColor: '#666', borderColor: '#eee', borderWidth: 1 }
  }
};

const systemMessage = { 
  role: "system", 
  content: "你是一位精煉的台灣股市分析師。請針對數據進行趨勢分析。嚴禁重複數據。直接指出走勢、量價配合與關鍵轉折。語氣需極度精簡。" 
};

const messages = ref<any[]>([systemMessage]);

async function initializeModel() {
  if (isInitializing.value || isLoaded.value) return;
  isInitializing.value = true;
  try {
    engineInstance = await CreateMLCEngine(selectedModel, {
      initProgressCallback: (report) => {
        statusText.value = report.text;
        progress.value = report.progress;
      }
    });
    isLoaded.value = true;
    statusText.value = "系統就緒";
  } catch (error: any) {
    console.error(error);
    statusText.value = "載入失敗";
    engineInstance = null;
  } finally {
    isInitializing.value = false;
  }
}

async function fetchDataAndSummarize() {
  if (!stockId.value.trim() || isWorking.value || !engineInstance) return;
  isWorking.value = true;

  try {
    messages.value = [systemMessage];
    statusText.value = `同步中...`;
    
    const pastDate = new Date();
    pastDate.setDate(pastDate.getDate() - 35);
    const startDate = pastDate.toISOString().split('T')[0];

    const [priceRes, indexRes] = await Promise.all([
      fetch(`https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockPrice&data_id=${stockId.value}&start_date=${startDate}`),
      fetch(`https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockPrice&data_id=TAIEX&start_date=${startDate}`)
    ]);

    const priceData = await priceRes.json();
    const indexData = await indexRes.json();

    if (priceData.msg === "success" && priceData.data.length > 0) {
      priceHistory.value = priceData.data.slice(-20).reverse();
      latestPrice.value = priceHistory.value[0];
      
      const plotPrice = [...priceData.data.slice(-20)];
      const plotIndex = indexData.msg === "success" ? indexData.data.slice(-20) : [];

      chartData.value = {
        labels: plotPrice.map(d => d.date.slice(5)),
        datasets: [
          {
            label: `個股 ${stockId.value}`,
            data: plotPrice.map(d => d.close),
            borderColor: '#334155',
            backgroundColor: 'rgba(51, 65, 85, 0.05)',
            fill: true,
            tension: 0.3,
            yAxisID: 'y'
          },
          {
            label: '台指',
            data: plotIndex.map(d => d.close),
            borderColor: '#94a3b8',
            borderDash: [4, 4],
            fill: false,
            tension: 0.3,
            yAxisID: 'y1'
          }
        ]
      };
    } else {
      throw new Error("查無代號");
    }

    try {
      const q = encodeURIComponent(`${stockId.value} 台灣 股票`);
      const newsRes = await fetch(`https://api.rss2json.com/v1/api.json?rss_url=https%3A%2F%2Fnews.google.com%2Frss%2Fsearch%3Fq%3D${q}%26hl%3Dzh-TW%26gl%3DTW%26ceid%3DTW%253Azh-Hant&api_key=oyw4f9v3j5o6p8q7a9v0q5u2b1r7z6m5n4l3k2j1`);
      const newsJson = await newsRes.json();
      if (newsJson.status === 'ok') {
        newsList.value = newsJson.items.slice(0, 6).map((item: any) => ({
          date: item.pubDate.split(' ')[0].slice(5),
          title: item.title.split(' - ')[0]
        }));
      }
    } catch (e) {
      console.warn(e);
    }

    const detailedPrice = priceHistory.value.slice(0, 5).map((d: any) => 
      `${d.date}: 收盤${d.close}, 漲跌${d.spread}`
    ).join('\n');
    
    const userPrompt = `標的: ${stockId.value}\n【近五日】\n${detailedPrice}\n【情報】\n${newsList.value.map(n => n.title).join('\n')}`;
    messages.value.push({ role: "user" as const, content: userPrompt });

    statusText.value = "AI 分析中";
    const assistantMsgIndex = messages.value.length;
    messages.value.push({ role: "assistant" as const, content: "" });

    const chunks = await engineInstance.chat.completions.create({
      messages: messages.value.slice(0, assistantMsgIndex) as any,
      stream: true,
      temperature: 0.2,
    });

    for await (const chunk of chunks) {
      messages.value[assistantMsgIndex].content += chunk.choices[0]?.delta?.content || "";
    }
    statusText.value = "系統就緒";
  } catch (error: any) {
    statusText.value = "錯誤";
    messages.value.push({ role: "assistant" as const, content: `[錯誤] ${error.message}` });
  } finally {
    isWorking.value = false;
  }
}

const themeOverrides = {
  common: { primaryColor: '#334155', borderRadius: '12px' },
  Card: { borderRadius: '16px', paddingMedium: '16px' }
};
</script>

<template>
  <n-config-provider :locale="zhTW" :date-locale="dateZhTW" :theme-overrides="themeOverrides">
    <n-global-style />
    <div class="main-container">
      
      <!-- 初始化全屏遮罩 -->
      <div v-if="!isLoaded" class="init-overlay">
        <n-card class="init-card" :bordered="false">
          <h2 class="init-title">戰情室系統</h2>
          <div class="progress-section">
            <div class="progress-info">
              <span>核心載入</span>
              <span class="percentage">{{ Math.round(progress * 100) }}%</span>
            </div>
            <n-progress type="line" :percentage="Math.round(progress * 100)" :show-indicator="false" processing color="#334155" rail-color="#e2e8f0" :height="12" />
          </div>
          <n-button type="primary" block size="large" @click="initializeModel" :disabled="isInitializing || !webGPUAvailable" :loading="isInitializing" class="init-btn">啟動戰情系統</n-button>
          <p v-if="isInitializing" class="download-hint">首次部署約 400MB，請保持網路連線</p>
        </n-card>
      </div>

      <!-- 主戰情室介面 (RWD 優化) -->
      <div v-else class="dashboard-layout">
        
        <!-- Top Bar (RWD) -->
        <div class="top-nav">
          <div class="nav-brand">
            <h1 class="logo font-black">AI DASHBOARD <span class="version text-xs opacity-30">v1.5</span></h1>
          </div>
          <div class="nav-controls">
            <n-input v-model:value="stockId" placeholder="代號" size="large" class="stock-input" @keyup.enter="fetchDataAndSummarize" />
            <n-button type="primary" size="large" @click="fetchDataAndSummarize" :disabled="isWorking" :loading="isWorking" class="exec-btn">分析</n-button>
          </div>
        </div>

        <!-- Main Content (RWD Grid) -->
        <div class="content-wrapper">
          <div class="grid-main">
            
            <!-- Left Side -->
            <div class="column-left">
              <n-space vertical :size="16">
                
                <!-- 圖表 -->
                <n-card title="走勢對比 (個股 vs 台指)" class="chart-card">
                  <div class="chart-container">
                    <Line v-if="chartData.labels.length > 0" :data="chartData" :options="chartOptions" />
                    <n-result v-else status="404" title="等待輸入" size="small" />
                  </div>
                </n-card>

                <!-- KPI (手機端 2x2) -->
                <div class="kpi-grid">
                  <div v-for="kpi in [
                    { label: '收盤價', value: latestPrice?.close || '---', color: '#1e293b' },
                    { label: '漲跌', value: latestPrice ? (latestPrice.spread > 0 ? '+' : '') + latestPrice.spread : '---', color: latestPrice?.spread > 0 ? '#ef4444' : '#10b981' },
                    { label: '成交量(M)', value: latestPrice?.Trading_Volume ? (latestPrice.Trading_Volume / 1000000).toFixed(2) : '---', color: '#64748b' },
                    { label: '狀態', value: isWorking ? '運算中' : '就緒', color: isWorking ? '#f59e0b' : '#10b981' }
                  ]" :key="kpi.label" class="kpi-box">
                    <span class="kpi-label">{{ kpi.label }}</span>
                    <span class="kpi-value" :style="{ color: kpi.color }">{{ kpi.value }}</span>
                  </div>
                </div>

                <!-- 雙面板 (手機端堆疊) -->
                <div class="sub-panels">
                  <n-card title="市場情報" class="side-panel">
                    <n-scrollbar style="max-height: 250px">
                      <div class="news-list">
                        <div v-for="n in newsList" :key="n.title" class="news-item">
                          <span class="news-date">{{ n.date }}</span>
                          <p class="news-title">{{ n.title }}</p>
                        </div>
                      </div>
                    </n-scrollbar>
                  </n-card>
                  <n-card title="交易紀錄" class="side-panel">
                    <n-scrollbar style="max-height: 250px">
                      <table class="history-table">
                        <tr v-for="d in priceHistory.slice(0, 10)" :key="d.date">
                          <td>{{ d.date.slice(5) }}</td>
                          <td align="right">{{ d.close }}</td>
                          <td align="right" :class="d.spread > 0 ? 'up' : 'down'">{{ d.spread > 0 ? '▲' : '▼' }}{{ Math.abs(d.spread) }}</td>
                        </tr>
                      </table>
                    </n-scrollbar>
                  </n-card>
                </div>

              </n-space>
            </div>

            <!-- Right Side (AI) -->
            <div class="column-right">
              <n-card title="AI 專家分析結果" class="ai-card" header-style="font-weight: bold; background: #f8fafc">
                <n-scrollbar style="max-height: 500px">
                  <div class="ai-chat-body">
                    <template v-for="(msg, i) in messages" :key="i">
                      <div v-if="msg.role === 'assistant'" class="ai-msg">
                        {{ msg.content }}
                      </div>
                    </template>
                    <div v-if="isWorking && messages.length <= 1" class="ai-loading">核心運算分析中...</div>
                    <div v-if="!latestPrice && !isWorking" class="ai-placeholder text-center text-gray-300 py-10">
                      請在上方輸入代號並執行分析
                    </div>
                  </div>
                </n-scrollbar>
              </n-card>
            </div>

          </div>
        </div>
      </div>
    </div>
  </n-config-provider>
</template>

<style>
body { margin: 0; font-family: 'PingFang TC', sans-serif; background-color: #f1f5f9; }
.main-container { min-height: 100vh; padding: 12px; }

/* 初始化 */
.init-overlay { position: fixed; inset: 0; background: #f1f5f9; display: flex; align-items: center; justify-content: center; z-index: 1000; padding: 20px; }
.init-card { max-width: 450px; width: 100%; text-align: center; border-radius: 24px !important; }
.init-title { font-size: 2rem; font-weight: 900; margin-bottom: 24px; }
.progress-section { margin-bottom: 32px; }
.progress-info { display: flex; justify-content: space-between; font-weight: 700; color: #94a3b8; font-size: 12px; margin-bottom: 8px; }
.init-btn { height: 56px !important; font-size: 1.1rem !important; font-weight: 900 !important; }

/* 頂部欄 */
.top-nav { background: white; padding: 12px 16px; border-radius: 16px; display: flex; flex-direction: column; gap: 12px; margin-bottom: 16px; border: 1px solid #e2e8f0; }
@media (min-width: 640px) { .top-nav { flex-direction: row; justify-content: space-between; padding: 16px 24px; } }
.nav-brand .logo { font-size: 1.2rem; color: #334155; margin: 0; letter-spacing: 1px; }
.nav-controls { display: flex; gap: 8px; width: 100%; }
@media (min-width: 640px) { .nav-controls { width: auto; } }
.stock-input { flex-grow: 1; min-width: 100px; }

/* 主內容區 */
.content-wrapper { height: auto; }
.grid-main { display: grid; grid-template-cols: 1fr; gap: 16px; }
@media (min-width: 1024px) { .grid-main { grid-template-cols: 8fr 4fr; } }

/* 走勢圖 */
.chart-card { border-radius: 20px !important; box-shadow: 0 2px 8px rgba(0,0,0,0.02) !important; }
.chart-container { height: 250px; }
@media (min-width: 640px) { .chart-container { height: 320px; } }

/* KPI 卡片 */
.kpi-grid { display: grid; grid-template-cols: repeat(2, 1fr); gap: 12px; }
@media (min-width: 768px) { .kpi-grid { grid-template-cols: repeat(4, 1fr); } }
.kpi-box { background: white; padding: 12px; border-radius: 16px; border: 1px solid #e2e8f0; display: flex; flex-direction: column; gap: 4px; }
.kpi-label { font-size: 11px; font-weight: 700; color: #94a3b8; }
.kpi-value { font-size: 1.5rem; font-weight: 900; letter-spacing: -0.5px; }

/* 下方雙面板 */
.sub-panels { display: grid; grid-template-cols: 1fr; gap: 16px; }
@media (min-width: 768px) { .sub-panels { grid-template-cols: 1fr 1fr; } }
.side-panel { border-radius: 20px !important; height: 100%; }

/* 新聞與表格 */
.news-item { padding: 12px 0; border-bottom: 1px solid #f1f5f9; }
.news-date { font-size: 10px; font-weight: 800; color: #cbd5e1; }
.news-title { margin: 2px 0 0 0; font-size: 0.95rem; font-weight: 600; color: #334155; line-height: 1.5; }
.history-table { width: 100%; font-size: 0.9rem; border-collapse: collapse; }
.history-table td { padding: 10px 0; border-bottom: 1px solid #f1f5f9; font-weight: 600; color: #475569; }
.up { color: #ef4444; }
.down { color: #10b981; }

/* AI 分析區 (淺色優化) */
.column-right { height: 100%; }
.ai-card { border-radius: 24px !important; height: 100%; border: 1px solid #e2e8f0 !important; box-shadow: 0 4px 20px rgba(0,0,0,0.03) !important; }
.ai-chat-body { padding: 8px; }
.ai-msg { background: white; border: 1px solid #f1f5f9; padding: 24px; border-radius: 20px; font-size: 1.1rem; line-height: 1.8; color: #1e293b; font-weight: 500; margin-bottom: 16px; box-shadow: 0 2px 10px rgba(0,0,0,0.01); }
.ai-loading { text-align: center; color: #94a3b8; padding: 20px; font-style: italic; font-size: 0.9rem; }
.ai-footer { display: flex; justify-content: space-between; align-items: center; font-size: 10px; font-weight: 800; color: #cbd5e1; padding: 4px 0; }

.pulse-dot { width: 6px; height: 6px; background: #ef4444; border-radius: 50%; margin-right: 6px; display: inline-block; }

.scrollbar-hide::-webkit-scrollbar { display: none; }
</style>
