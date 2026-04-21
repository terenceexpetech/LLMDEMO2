<script setup lang="ts">
import { ref, computed } from 'vue';
import { CreateMLCEngine, type MLCEngine } from "@mlc-ai/web-llm";
import { 
  NConfigProvider, 
  NGlobalStyle, 
  NScrollbar, 
  zhTW,
  dateZhTW,
  NResult
} from 'naive-ui';

// 引入 shadcn-vue 組件
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { ScrollArea } from '@/components/ui/scroll-area';

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
const statusText = ref("系統待命");
const progressValue = ref(0);
const webGPUAvailable = ref(true);
const userInput = ref(""); 
const tacticalSummary = ref(""); // 專用：存放核心分析報告
const chatMessages = ref<any[]>([]); // 專用：存放使用者主動發起的對話

if (typeof navigator !== "undefined" && !("gpu" in navigator)) {
  webGPUAvailable.value = false;
  statusText.value = "不支援_WEBGPU";
}

// 數據狀態
const stockId = ref("2330");
const priceHistory = ref<any[]>([]);
const indexHistory = ref<any[]>([]);
const newsList = ref<any[]>([]);
const latestPrice = ref<any>(null);

// 圖表數據 (工業風：2px 細線條)
const chartData = ref<any>({ labels: [], datasets: [] });
const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  interaction: { mode: 'index', intersect: false },
  scales: {
    y: { 
      display: true, 
      position: 'left', 
      grid: { color: '#f1f5f9' }, 
      ticks: { color: '#0f172a', font: { size: 10, family: 'monospace', weight: 'bold' } } 
    },
    y1: { 
      display: true, 
      position: 'right', 
      grid: { drawOnChartArea: false }, 
      ticks: { color: '#64748b', font: { size: 10, family: 'monospace' } } 
    },
    x: { 
      grid: { display: false }, 
      ticks: { color: '#0f172a', font: { size: 10, family: 'monospace', weight: 'bold' } } 
    }
  },
  plugins: {
    legend: { 
      position: 'top', 
      align: 'end', 
      labels: { boxWidth: 10, color: '#0f172a', font: { size: 11, weight: 'bold' } } 
    },
    tooltip: { 
      backgroundColor: '#ffffff', 
      titleColor: '#0f172a', 
      bodyColor: '#475569', 
      borderColor: '#0f172a', 
      borderWidth: 1,
      padding: 8,
      cornerRadius: 0,
      displayColors: false,
    }
  }
};

const systemMessage = { 
  role: "system", 
  content: "你是一位精煉的台灣股市分析師。請針對數據進行趨勢分析。嚴禁重複數據。直接指出走勢、量價配合與關鍵轉折。語氣需極度精簡。" 
};

// AI 背景記憶
const internalMessages = ref<any[]>([systemMessage]);

async function initializeModel() {
  if (isInitializing.value || isLoaded.value) return;
  isInitializing.value = true;
  try {
    engineInstance = await CreateMLCEngine(selectedModel, {
      initProgressCallback: (report) => {
        statusText.value = `核心部署_${Math.round(report.progress * 100)}%`;
        progressValue.value = Math.round(report.progress * 100);
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
  tacticalSummary.value = ""; // 清空舊報告內容
  
  try {
    statusText.value = `數據同步中...`;
    
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
            borderColor: '#0f172a',
            borderWidth: 2,
            backgroundColor: 'rgba(15, 23, 42, 0.03)',
            fill: true,
            tension: 0,
            yAxisID: 'y'
          },
          {
            label: '台指',
            data: plotIndex.map(d => d.close),
            borderColor: '#94a3b8',
            borderWidth: 1.5,
            fill: false,
            tension: 0,
            yAxisID: 'y1'
          }
        ]
      };
    } else {
      throw new Error("查無代號");
    }

    try {
      const newsRes = await fetch(`https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockNews&data_id=${stockId.value}&start_date=${startDate}`);
      const newsJson = await newsRes.json();
      if (newsJson.msg === 'success' && newsJson.data.length > 0) {
        newsList.value = newsJson.data.slice(-8).reverse().map((item: any) => ({
          date: item.date,
          title: item.title,
          link: item.link
        }));
      } else {
        newsList.value = [];
      }
    } catch (e) {
      console.warn(e);
    }

    const detailedPrice = priceHistory.value.slice(0, 5).map((d: any) => 
      `${d.date}: 收盤${d.close} (${d.spread})`
    ).join('\n');
    
    const userPrompt = `標的: ${stockId.value}\n[數據]\n${detailedPrice}\n[新聞]\n${newsList.value.map(n => n.title).join('\n')}`;
    
    // 背景記憶重置並加入當前標的數據
    internalMessages.value = [systemMessage, { role: "user", content: userPrompt }];

    statusText.value = "AI 分析中";
    const chunks = await engineInstance.chat.completions.create({
      messages: internalMessages.value as any,
      stream: true,
      temperature: 0.1,
    });

    for await (const chunk of chunks) {
      tacticalSummary.value += chunk.choices[0]?.delta?.content || "";
    }
    
    // 將分析結果同步到背景記憶
    internalMessages.value.push({ role: "assistant", content: tacticalSummary.value });
    statusText.value = "系統就緒";
  } catch (error: any) {
    statusText.value = "發生錯誤";
    tacticalSummary.value = `[錯誤] ${error.message}`;
  } finally {
    isWorking.value = false;
  }
}

async function sendMessage() {
  if (!userInput.value.trim() || isWorking.value || !engineInstance) return;
  
  const text = userInput.value;
  userInput.value = "";
  isWorking.value = true;
  statusText.value = "AI 正在回覆";

  try {
    // 1. 顯示在對話框
    chatMessages.value.push({ role: "user", content: text });
    // 2. 同步到背景記憶
    internalMessages.value.push({ role: "user", content: text });
    
    const assistantMsgIndex = chatMessages.value.length;
    chatMessages.value.push({ role: "assistant", content: "" });

    const chunks = await engineInstance.chat.completions.create({
      messages: internalMessages.value as any,
      stream: true,
      temperature: 0.3,
    });

    for await (const chunk of chunks) {
      const delta = chunk.choices[0]?.delta?.content || "";
      chatMessages.value[assistantMsgIndex].content += delta;
    }
    
    // 同步回覆到背景記憶
    internalMessages.value.push({ role: "assistant", content: chatMessages.value[assistantMsgIndex].content });
    statusText.value = "系統就緒";
  } catch (error: any) {
    statusText.value = "連線失敗";
    chatMessages.value.push({ role: "assistant", content: `[錯誤] ${error.message}` });
  } finally {
    isWorking.value = false;
  }
}

const themeOverrides = {
  common: { primaryColor: '#0f172a', borderRadius: '0px' },
  Card: { borderRadius: '0px' }
};
</script>

<template>
  <n-config-provider :locale="zhTW" :date-locale="dateZhTW" :theme-overrides="themeOverrides">
    <n-global-style />
    <div class="min-h-screen bg-[#fcfcfc] text-slate-900 font-sans selection:bg-slate-900 selection:text-white">
      
      <!-- 初始化全螢幕遮罩 -->
      <div v-if="!isLoaded" class="fixed inset-0 bg-white flex items-center justify-center z-[100] px-4">
        <div class="max-w-md w-full border-2 border-slate-900 bg-white p-8 space-y-8">
          <div class="space-y-2 border-b-2 border-slate-900 pb-6 text-center">
            <h1 class="text-3xl font-black tracking-tighter uppercase italic">Analytic_Core</h1>
            <p class="text-[9px] font-bold text-slate-500 tracking-[0.4em]">本地推理引擎 v1.5 / 繁體中文版</p>
          </div>
          
          <div class="space-y-6">
            <div class="space-y-2">
              <div class="flex justify-between text-[10px] font-black uppercase">
                <span>核心組件部署進度</span>
                <span class="font-mono">{{ progressValue }}%</span>
              </div>
              <div class="h-4 border-2 border-slate-900 p-0.5">
                <div class="h-full bg-slate-900 transition-all duration-300" :style="{ width: progressValue + '%' }"></div>
              </div>
            </div>
            
            <Button 
              size="lg" 
              class="w-full h-16 text-xl font-black uppercase rounded-none border-2 border-slate-900 bg-slate-900 text-white hover:bg-white hover:text-slate-900 transition-all active:translate-y-1"
              @click="initializeModel" 
              :disabled="isInitializing || !webGPUAvailable"
            >
              <span v-if="!isInitializing">進入分析中心</span>
              <span v-else class="animate-pulse">部署中...</span>
            </Button>
          </div>
        </div>
      </div>

      <!-- 主儀表板佈局 -->
      <div v-else class="max-w-[1800px] mx-auto p-3 md:p-4 space-y-4 animate-in fade-in duration-500">
        
        <!-- 頂部控制面板 -->
        <header class="flex flex-col md:flex-row items-stretch border-2 border-slate-900 bg-white shadow-sm">
          <div class="flex items-center gap-6 px-6 py-3 bg-slate-900 text-white">
            <div class="text-3xl font-black italic tracking-tighter">QA</div>
            <div class="h-8 w-0.5 bg-slate-700"></div>
            <div>
              <h1 class="text-lg font-black uppercase tracking-tight leading-none">戰情終端 A01</h1>
              <p class="text-[9px] font-bold text-slate-400 mt-1 uppercase tracking-widest">狀態: {{ statusText }}</p>
            </div>
          </div>
          
          <div class="flex flex-grow items-stretch">
            <input 
              v-model="stockId" 
              placeholder="輸入股票代號 (例: 2330)" 
              class="flex-grow px-6 py-3 bg-white border-l-2 border-slate-900 focus:outline-none font-bold text-xl placeholder:text-slate-200 uppercase"
              @keyup.enter="fetchDataAndSummarize"
            />
            <button 
              @click="fetchDataAndSummarize" 
              :disabled="isWorking"
              class="px-8 bg-slate-900 text-white font-black text-sm uppercase hover:bg-slate-800 transition-all active:bg-slate-700 border-l-2 border-slate-900 disabled:bg-slate-200"
            >
              <span v-if="!isWorking">執行數據分析</span>
              <span v-else class="animate-pulse">運算中...</span>
            </button>
          </div>
        </header>

        <!-- 三欄式網格佈局 -->
        <div class="grid grid-cols-1 lg:grid-cols-12 gap-4">
          
          <!-- 左側欄：即時市場情報 -->
          <aside class="lg:col-span-3 border-2 border-slate-900 bg-white flex flex-col shadow-sm">
            <div class="bg-slate-900 text-white py-2 px-4 flex justify-between items-center">
              <span class="text-[10px] font-black uppercase tracking-widest">即時市場情報</span>
              <div class="w-2 h-2 bg-emerald-500 animate-pulse"></div>
            </div>
            <ScrollArea class="flex-grow h-[750px]">
              <div class="p-4 space-y-5">
                <a 
                  v-for="n in newsList" 
                  :key="n.title" 
                  :href="n.link" 
                  target="_blank" 
                  class="block border-b border-slate-100 pb-4 group hover:border-slate-900 transition-colors"
                >
                  <div class="flex items-center gap-2 mb-1.5">
                    <span class="text-[9px] font-black font-mono text-slate-400">{{ n.date }}</span>
                    <Badge variant="outline" class="rounded-none border-slate-200 text-[8px] px-1 h-3.5 font-black">NEWS</Badge>
                  </div>
                  <h3 class="text-xs font-black text-slate-800 leading-snug group-hover:text-slate-900 group-hover:underline">
                    {{ n.title }}
                  </h3>
                </a>
                <div v-if="newsList.length === 0" class="text-center py-24 border-2 border-dashed border-slate-50">
                  <p class="text-[10px] font-black text-slate-300 uppercase tracking-widest">等待指令載入</p>
                </div>
              </div>
            </ScrollArea>
          </aside>

          <!-- 中間欄：圖表與核心數據 -->
          <main class="lg:col-span-6 space-y-4">
            
            <Card class="border-2 border-slate-900 bg-white rounded-none shadow-none">
              <CardHeader class="border-b-2 border-slate-900 py-2.5 px-6 flex flex-row items-center justify-between">
                <CardTitle class="text-[10px] font-black uppercase tracking-[0.2em]">價格走勢與大盤比較</CardTitle>
                <div class="flex gap-1">
                  <div v-for="i in 3" :key="i" class="w-1 h-4 bg-slate-900"></div>
                </div>
              </CardHeader>
              <CardContent class="p-4">
                <div class="h-[400px]">
                  <Line v-if="chartData.labels.length > 0" :data="chartData" :options="chartOptions" />
                  <div v-else class="h-full flex flex-col items-center justify-center border-2 border-dashed border-slate-50">
                    <p class="font-black uppercase tracking-[0.4em] text-[9px] text-slate-200">Waiting_For_Target_Input</p>
                  </div>
                </div>
              </CardContent>
            </Card>

            <div class="grid grid-cols-2 md:grid-cols-4 gap-0 border-t-2 border-l-2 border-slate-900 bg-white">
              <div v-for="kpi in [
                { label: '當前收盤', value: latestPrice?.close || '0.00', color: 'text-slate-900' },
                { label: '今日漲跌', value: latestPrice ? (latestPrice.spread > 0 ? '+' : '') + latestPrice.spread : '0.00', color: latestPrice?.spread > 0 ? 'text-white bg-rose-600' : 'text-white bg-emerald-600' },
                { label: '成交金額(M)', value: latestPrice?.Trading_Volume ? (latestPrice.Trading_Volume / 1000000).toFixed(2) : '0.00', color: 'text-slate-900' },
                { label: '近五日均價', value: priceHistory.length > 0 ? (priceHistory.reduce((a, b) => a + b.close, 0) / priceHistory.length).toFixed(2) : '0.00', color: 'text-slate-900' }
              ]" :key="kpi.label" class="border-b-2 border-r-2 border-slate-900 p-4">
                <p class="text-[9px] font-black text-slate-400 uppercase tracking-widest mb-1.5">{{ kpi.label }}</p>
                <p class="text-2xl font-black tracking-tighter font-mono" :class="kpi.color">{{ kpi.value }}</p>
              </div>
            </div>

            <div class="border-2 border-slate-900 bg-white overflow-hidden">
              <div class="bg-slate-100 border-b-2 border-slate-900 py-1.5 px-4">
                <span class="text-[9px] font-black uppercase tracking-widest">近五日交易數據快照</span>
              </div>
              <table class="w-full text-[10px] font-mono border-collapse">
                <tbody>
                  <tr v-for="d in priceHistory.slice(0, 5)" :key="d.date" class="border-b border-slate-100 hover:bg-slate-50 transition-colors">
                    <td class="py-2.5 px-6 font-black text-slate-400">{{ d.date }}</td>
                    <td class="py-2.5 px-6 text-right font-black text-slate-900">收盤: {{ d.close }}</td>
                    <td class="py-2.5 px-6 text-right font-black" :class="d.spread > 0 ? 'text-rose-600' : 'text-emerald-600'">
                      漲跌: {{ d.spread > 0 ? '▲' : '▼' }}{{ Math.abs(d.spread) }}
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </main>

          <!-- 右側欄：分析與對話 -->
          <section class="lg:col-span-3 flex flex-col gap-4">
            
            <!-- 上半部：戰術摘要 (固定顯示主報告) -->
            <div class="border-2 border-slate-900 bg-white flex flex-col h-[300px] shadow-sm">
              <div class="bg-slate-900 text-white py-2 px-4 flex justify-between items-center shrink-0">
                <span class="text-[10px] font-black tracking-widest uppercase">戰術分析報告</span>
                <Badge v-if="isWorking && tacticalSummary === ''" class="bg-blue-600 border-none text-[8px] animate-pulse">ANALYZING</Badge>
              </div>
              <ScrollArea class="flex-grow">
                <div class="p-4">
                  <p v-if="tacticalSummary" class="text-xs font-black leading-relaxed text-slate-900 uppercase whitespace-pre-wrap">
                    {{ tacticalSummary }}
                  </p>
                  <div v-else-if="isWorking" class="flex flex-col items-center justify-center py-12 text-slate-300">
                    <div class="w-8 h-0.5 bg-slate-200 animate-pulse mb-2"></div>
                    <p class="text-[8px] font-black uppercase tracking-widest">正在解析市場數據...</p>
                  </div>
                  <div v-else class="text-center py-20 text-slate-200">
                    <p class="text-[9px] font-black uppercase tracking-widest">等待標的導入</p>
                  </div>
                </div>
              </ScrollArea>
            </div>

            <!-- 下半部：通訊日誌 (互動對話框) -->
            <div class="border-2 border-slate-900 bg-white flex-grow flex flex-col shadow-[6px_6px_0px_rgba(15,23,42,1)]">
              <div class="bg-slate-100 border-b-2 border-slate-900 py-2 px-4 shrink-0">
                <span class="text-[9px] font-black tracking-widest uppercase text-slate-600">指令互動日誌</span>
              </div>
              
              <div class="flex-grow overflow-hidden">
                <ScrollArea class="h-[350px] lg:h-[400px]">
                  <div class="p-4 space-y-6">
                    <template v-for="(msg, i) in chatMessages" :key="i">
                      <div class="animate-in slide-in-from-top-1 duration-200">
                        <div :class="msg.role === 'user' ? 'border-r-4 pr-3 text-right border-slate-200' : 'border-l-4 border-slate-900 pl-3 text-left'">
                          <p class="text-[7px] font-black text-slate-400 uppercase mb-1 tracking-tighter">
                            {{ msg.role === 'user' ? 'CMD_INPUT' : 'CORE_REPLY' }}
                          </p>
                          <p class="text-[11px] font-black leading-tight text-slate-900 uppercase">
                            {{ msg.content }}
                          </p>
                        </div>
                      </div>
                    </template>
                    <div v-if="chatMessages.length === 0" class="text-center py-20 text-slate-100 italic">
                      <p class="text-[8px] font-black uppercase tracking-widest">Awaiting Command...</p>
                    </div>
                  </div>
                </ScrollArea>
              </div>

              <div class="border-t-2 border-slate-900 p-3 bg-slate-50 space-y-2 shrink-0">
                <div class="flex gap-2">
                  <input 
                    v-model="userInput" 
                    placeholder="輸入追蹤指令..." 
                    class="flex-grow bg-white border-2 border-slate-900 p-2 text-[11px] font-black focus:outline-none placeholder:text-slate-200 uppercase"
                    @keyup.enter="sendMessage"
                  />
                  <button 
                    @click="sendMessage"
                    :disabled="isWorking || !isLoaded"
                    class="bg-slate-900 text-white px-3 font-black text-[10px] uppercase hover:bg-slate-800 disabled:bg-slate-200"
                  >
                    CMD
                  </button>
                </div>
                <div class="flex justify-between items-center text-[7px] font-black text-slate-400 tracking-tighter uppercase">
                  <span>QWEN_V2.5_0.5B</span>
                  <span class="flex items-center gap-1"><span class="w-1 h-1 bg-emerald-500"></span> SECURE_LINK</span>
                </div>
              </div>
            </div>
          </section>

        </div>
      </div>
    </div>
  </n-config-provider>
</template>

<style>
  @import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@800&display=swap');
  body { background-color: #ffffff; font-family: 'PingFang TC', 'Microsoft JhengHei', sans-serif; }
  .font-mono { font-family: 'JetBrains Mono', monospace !important; }
  .n-scrollbar-rail { --n-scrollbar-width: 2px; }
  .chart-container canvas { image-rendering: crisp-edges; }
  input:focus { box-shadow: inset 0 0 0 2px #0f172a; }
</style>
