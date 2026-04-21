<script setup lang="ts">
import { ref, computed, watch, nextTick } from 'vue';
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
const tacticalSummary = ref(""); 
const chatMessages = ref<any[]>([]); 
const chatScrollRef = ref<any>(null);
const summaryScrollRef = ref<any>(null);

if (typeof navigator !== "undefined" && !("gpu" in navigator)) {
  webGPUAvailable.value = false;
  statusText.value = "不支援_WEBGPU";
}

// 數據狀態
const stockId = ref("2330");
const priceHistory = ref<any[]>([]);
const newsList = ref<any[]>([]);
const latestPrice = ref<any>(null);

// 圖表數據 (精密風)
const chartData = ref<any>({ labels: [], datasets: [] });
const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  interaction: { mode: 'index', intersect: false },
  scales: {
    y: { display: true, grid: { color: '#f1f5f9' }, ticks: { color: '#0f172a', font: { size: 9, family: 'monospace' } } },
    y1: { display: true, grid: { drawOnChartArea: false }, ticks: { color: '#94a3b8', font: { size: 9, family: 'monospace' } } },
    x: { grid: { display: false }, ticks: { color: '#0f172a', font: { size: 9, family: 'monospace' } } }
  },
  plugins: {
    legend: { position: 'top', align: 'end', labels: { boxWidth: 8, color: '#0f172a', font: { size: 10, weight: 'bold' } } },
    tooltip: { backgroundColor: '#ffffff', borderColor: '#0f172a', borderWidth: 1, padding: 6, cornerRadius: 0 }
  }
};

// 結合數據與新聞的精確提示詞 (提速優化版)
const systemMessage = { 
  role: "system", 
  content: "你是一位精煉的台灣股市分析師。請結合「數據趨勢」與「最新新聞」進行綜合解讀。直接點出走勢關鍵、新聞對情緒的影響及對策。嚴禁 Markdown 與星號，僅輸出繁體中文純文字。" 
};

const internalMessages = ref<any[]>([systemMessage]);

// 自動捲動
watch([chatMessages, tacticalSummary], () => {
  nextTick(() => {
    if (chatScrollRef.value) chatScrollRef.value.scrollTo({ top: 9999, behavior: 'smooth' });
    if (summaryScrollRef.value) summaryScrollRef.value.scrollTo({ top: 9999, behavior: 'smooth' });
  });
}, { deep: true });

// 文字後處理
function cleanText(text: string) {
  return text
    .replace(/[\*#]/g, "")
    .replace(/信息/g, "資訊")
    .replace(/支持/g, "支撐")
    .replace(/走势/g, "走勢");
}

async function initializeModel() {
  if (isInitializing.value || isLoaded.value) return;
  isInitializing.value = true;
  try {
    engineInstance = await CreateMLCEngine(selectedModel, {
      initProgressCallback: (report) => {
        statusText.value = `核心載入_${Math.round(report.progress * 100)}%`;
        progressValue.value = Math.round(report.progress * 100);
      }
    });
    isLoaded.value = true;
    statusText.value = "系統就緒";
  } catch (error: any) {
    statusText.value = "載入失敗";
    engineInstance = null;
  } finally {
    isInitializing.value = false;
  }
}

async function fetchDataAndSummarize() {
  if (!stockId.value.trim() || isWorking.value || !engineInstance) return;
  isWorking.value = true;
  tacticalSummary.value = ""; 
  
  try {
    statusText.value = `同步數據中...`;
    
    const dateList = [];
    for (let i = 0; i < 10; i++) {
      const d = new Date();
      d.setDate(d.getDate() - i);
      dateList.push(d.toISOString().split('T')[0]);
    }
    const chartDate = new Date();
    chartDate.setDate(chartDate.getDate() - 90);
    const chartStartDate = chartDate.toISOString().split('T')[0];

    const [priceRes, indexRes] = await Promise.all([
      fetch(`https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockPrice&data_id=${stockId.value}&start_date=${chartStartDate}`),
      fetch(`https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockPrice&data_id=TAIEX&start_date=${chartStartDate}`)
    ]);

    const priceData = await priceRes.json();
    const indexData = await indexRes.json();

    if (priceData.msg === "success" && priceData.data.length > 0) {
      priceHistory.value = priceData.data.slice(-15).reverse();
      latestPrice.value = priceHistory.value[0];
      
      const plotPrice = priceData.data;
      const plotIndex = indexData.msg === "success" ? indexData.data : [];

      chartData.value = {
        labels: plotPrice.map(d => d.date.slice(5)),
        datasets: [
          { label: `個股 ${stockId.value}`, data: plotPrice.map(d => d.close), borderColor: '#0f172a', borderWidth: 2, backgroundColor: 'rgba(15, 23, 42, 0.03)', fill: true, tension: 0, yAxisID: 'y', pointRadius: 0 },
          { label: '台指', data: plotIndex.map(d => d.close), borderColor: '#94a3b8', borderWidth: 1.5, fill: false, tension: 0, yAxisID: 'y1', pointRadius: 0 }
        ]
      };
    } else {
      throw new Error("查無代號");
    }

    try {
      const newsPromises = dateList.slice(0, 7).map(date => 
        fetch(`https://api.finmindtrade.com/api/v4/data?dataset=TaiwanStockNews&data_id=${stockId.value}&start_date=${date}`).then(res => res.json())
      );
      const newsResults = await Promise.all(newsPromises);
      const allNews = newsResults
        .filter(res => res.msg === 'success' && res.data)
        .flatMap(res => res.data)
        .sort((a, b) => new Date(b.date).getTime() - new Date(a.date).getTime());
      
      newsList.value = allNews.slice(0, 30).map((item: any) => ({
        date: item.date, title: item.title, link: item.link
      }));
    } catch (e) {
      console.warn("新聞抓取失敗", e);
    }

    const detailedPrice = priceHistory.value.slice(0, 5).map((d: any) => `${d.date}: ${d.close} (${d.spread})`).join('\n');
    
    // 組合更明確的 Input 引導 AI 讀取新聞
    const userPrompt = `【關鍵新聞標題】\n${newsList.value.slice(0, 8).map(n => n.title).join('\n')}\n\n【近五日量價走勢】\n${detailedPrice}\n\n請結合上述新聞與數據對 ${stockId.value} 進行分析：`;
    
    internalMessages.value = [systemMessage, { role: "user", content: userPrompt }];
    statusText.value = "AI 分析中";
    const chunks = await engineInstance.chat.completions.create({ messages: internalMessages.value as any, stream: true, temperature: 0.1 });

    for await (const chunk of chunks) { 
      const content = chunk.choices[0]?.delta?.content || "";
      tacticalSummary.value += cleanText(content); 
    }
    internalMessages.value.push({ role: "assistant", content: tacticalSummary.value });
    statusText.value = "系統就緒";
  } catch (error: any) {
    statusText.value = "錯誤";
    tacticalSummary.value = `[系統失敗] ${error.message}`;
  } finally {
    isWorking.value = false;
  }
}

async function sendMessage() {
  if (!userInput.value.trim() || isWorking.value || !engineInstance) return;
  const text = userInput.value;
  userInput.value = "";
  isWorking.value = true;
  statusText.value = "AI 回覆中";

  try {
    chatMessages.value.push({ role: "user", content: text });
    internalMessages.value.push({ role: "user", content: text });
    const assistantMsgIndex = chatMessages.value.length;
    chatMessages.value.push({ role: "assistant", content: "" });

    const chunks = await engineInstance.chat.completions.create({ messages: internalMessages.value as any, stream: true, temperature: 0.3 });
    for await (const chunk of chunks) { 
      const delta = chunk.choices[0]?.delta?.content || "";
      chatMessages.value[assistantMsgIndex].content += cleanText(delta); 
    }
    internalMessages.value.push({ role: "assistant", content: chatMessages.value[assistantMsgIndex].content });
    statusText.value = "系統就緒";
  } catch (error: any) {
    chatMessages.value.push({ role: "assistant", content: `[錯誤] ${error.message}` });
  } finally {
    isWorking.value = false;
  }
}

const themeOverrides = { common: { primaryColor: '#0f172a', borderRadius: '0px' }, Card: { borderRadius: '0px' } };
</script>

<template>
  <n-config-provider :locale="zhTW" :date-locale="dateZhTW" :theme-overrides="themeOverrides">
    <n-global-style />
    <div class="h-screen w-full bg-[#fcfcfc] text-slate-900 font-sans selection:bg-slate-900 selection:text-white overflow-hidden text-sm">
      
      <div v-if="!isLoaded" class="fixed inset-0 bg-white flex items-center justify-center z-[100] px-4 text-slate-900">
        <div class="max-w-md w-full border-2 border-slate-900 bg-white p-8 space-y-8 text-slate-900">
          <div class="space-y-2 border-b-2 border-slate-900 pb-6 text-center text-slate-900">
            <h1 class="text-3xl font-black tracking-tighter uppercase italic text-slate-900">Analytic_Core</h1>
            <p class="text-[9px] font-bold text-slate-500 tracking-[0.4em]">本地推理引擎 v1.5 / 繁體中文版</p>
          </div>
          <div class="space-y-6">
            <div class="space-y-2">
              <div class="flex justify-between text-[10px] font-black uppercase text-slate-900"><span>部署進度</span><span>{{ progressValue }}%</span></div>
              <div class="h-4 border-2 border-slate-900 p-0.5"><div class="h-full bg-slate-900 transition-all duration-300" :style="{ width: progressValue + '%' }"></div></div>
            </div>
            <Button size="lg" class="w-full h-16 text-xl font-black uppercase rounded-none border-2 border-slate-900 bg-slate-900 text-white hover:bg-white hover:text-slate-900 transition-all active:translate-y-1" @click="initializeModel" :disabled="isInitializing || !webGPUAvailable">
              <span v-if="!isInitializing">進入分析中心</span><span v-else class="animate-pulse">BOOTING...</span>
            </Button>
          </div>
        </div>
      </div>

      <div v-else class="h-full flex flex-col p-2 gap-2 animate-in fade-in duration-500">
        <header class="flex-none flex items-stretch border-2 border-slate-900 bg-white shadow-sm">
          <div class="flex items-center gap-3 px-4 py-1.5 bg-slate-900 text-white text-slate-900">
            <div class="text-xl font-black italic tracking-tighter uppercase text-white">QA_TERMINAL</div>
            <div class="h-5 w-0.5 bg-slate-700"></div>
            <div><p class="text-[8px] font-bold text-slate-400 uppercase tracking-widest text-white">{{ statusText }}</p></div>
          </div>
          <div class="flex-grow flex items-stretch text-slate-900">
            <input v-model="stockId" placeholder="輸入台股代號" class="flex-grow px-4 bg-white border-l-2 border-slate-900 focus:outline-none font-bold text-lg placeholder:text-slate-200 uppercase text-slate-900" @keyup.enter="fetchDataAndSummarize" />
            <button @click="fetchDataAndSummarize" :disabled="isWorking" class="px-6 bg-slate-900 text-white font-black text-xs uppercase hover:bg-slate-800 border-l-2 border-slate-900 disabled:bg-slate-200 transition-colors">
              <span v-if="!isWorking">執行分析</span><span v-else class="animate-pulse">RUNNING</span>
            </button>
          </div>
        </header>

        <div class="flex-grow grid grid-cols-12 gap-2 min-h-0 text-slate-900">
          <aside class="col-span-3 border-2 border-slate-900 bg-white flex flex-col shadow-sm min-h-0 text-slate-900">
            <div class="bg-slate-900 text-white py-1 px-4 flex justify-between items-center shrink-0"><span class="text-[9px] font-black uppercase tracking-widest">市場情報快訊庫</span><div class="w-1.5 h-1.5 bg-emerald-500 animate-pulse"></div></div>
            <n-scrollbar class="flex-grow">
              <div class="p-3 space-y-3">
                <a v-for="n in newsList" :key="n.title + n.date" :href="n.link" target="_blank" class="block border-b border-slate-100 pb-3 group hover:border-slate-900 transition-colors">
                  <div class="flex items-center gap-2 mb-1"><span class="text-[8px] font-black font-mono text-slate-400">{{ n.date }}</span><Badge variant="outline" class="rounded-none border-slate-200 text-[7px] px-1 h-3.5 font-black text-slate-400">NEWS</Badge></div>
                  <h3 class="text-[11px] font-black text-slate-800 leading-tight group-hover:text-slate-900 group-hover:underline uppercase text-slate-900">{{ n.title }}</h3>
                </a>
              </div>
            </n-scrollbar>
          </aside>

          <main class="col-span-6 flex flex-col gap-2 min-h-0">
            <Card class="flex-none border-2 border-slate-900 bg-white rounded-none shadow-none overflow-hidden text-slate-900">
              <CardHeader class="border-b-2 border-slate-900 py-1 px-4 flex flex-row items-center justify-between shrink-0 text-slate-900"><CardTitle class="text-[9px] font-black uppercase tracking-[0.2em] text-slate-900">趨勢比較 / 價格與大盤</CardTitle><div class="w-1 h-3 bg-slate-900"></div></CardHeader>
              <CardContent class="p-2">
                <div class="h-[300px]"><Line v-if="chartData.labels.length > 0" :data="chartData" :options="chartOptions" /><div v-else class="h-full flex flex-col items-center justify-center border border-dashed border-slate-50"><p class="font-black uppercase tracking-[0.4em] text-[8px] text-slate-200">AWAITING_INPUT</p></div></div>
              </CardContent>
            </Card>

            <div class="flex-none grid grid-cols-4 gap-0 border-t-2 border-l-2 border-slate-900 bg-white">
              <div v-for="kpi in [{ label: '現價', value: latestPrice?.close || '0.00' }, { label: '漲跌幅度', value: latestPrice ? (latestPrice.spread > 0 ? '+' : '') + latestPrice.spread : '0.00', color: latestPrice?.spread > 0 ? 'bg-rose-600 text-white' : 'bg-emerald-600 text-white' }, { label: '成交金額(M)', value: latestPrice?.Trading_Volume ? (latestPrice.Trading_Volume / 1000000).toFixed(2) : '0.00' }, { label: '十日均價', value: priceHistory.length > 0 ? (priceHistory.reduce((a, b) => a + b.close, 0) / priceHistory.length).toFixed(2) : '0.00' }]" :key="kpi.label" class="border-b-2 border-r-2 border-slate-900 p-2 text-slate-900 text-slate-900">
                <p class="text-[8px] font-black text-slate-400 uppercase tracking-widest mb-0.5 text-slate-400 font-bold">{{ kpi.label }}</p>
                <p class="text-2xl font-black tracking-tighter font-mono" :class="kpi.color || 'text-slate-900'">{{ kpi.value }}</p>
              </div>
            </div>

            <div class="flex-grow border-2 border-slate-900 bg-white overflow-hidden flex flex-col min-h-0 shadow-sm">
              <div class="bg-slate-100 border-b border-slate-900 py-1 px-4 flex-none"><span class="text-[8px] font-black uppercase tracking-widest text-slate-500 font-bold">歷史成交對照索引庫</span></div>
              <n-scrollbar class="flex-grow">
                <table class="w-full text-[9px] font-mono border-collapse text-slate-900">
                  <tbody>
                    <tr v-for="d in priceHistory.slice(0, 15)" :key="d.date" class="border-b border-slate-50 hover:bg-slate-50 transition-colors">
                      <td class="py-1.5 px-4 font-black text-slate-400 font-bold">{{ d.date }}</td>
                      <td class="py-1.5 px-4 text-right font-black text-slate-900">收盤: {{ d.close }}</td>
                      <td class="py-1.5 px-4 text-right font-black" :class="d.spread > 0 ? 'text-rose-600' : 'text-emerald-600'">漲跌: {{ d.spread > 0 ? '▲' : '▼' }}{{ Math.abs(d.spread) }}</td>
                    </tr>
                  </tbody>
                </table>
              </n-scrollbar>
            </div>
          </main>

          <section class="col-span-3 flex flex-col gap-2 min-h-0">
            <div class="flex-[2] border-2 border-slate-900 bg-white flex flex-col shadow-sm min-h-0 overflow-hidden">
              <div class="bg-slate-900 text-white py-1.5 px-4 flex justify-between items-center shrink-0">
                <span class="text-[9px] font-black tracking-widest uppercase italic">AI_TACTICAL_SUMMARY</span>
                <Badge v-if="isWorking && tacticalSummary === ''" class="bg-blue-600 text-[7px] animate-pulse border-none rounded-none text-white">ANALYZING</Badge>
              </div>
              <n-scrollbar ref="summaryScrollRef" class="flex-grow">
                <div class="p-5 text-[15px] font-black leading-[1.6] text-slate-900 uppercase whitespace-pre-wrap text-slate-900">
                  {{ tacticalSummary || '等待指令載入分析結果...' }}
                </div>
              </n-scrollbar>
            </div>

            <div class="flex-[3] border-2 border-slate-900 bg-white flex flex-col shadow-[6px_6px_0px_rgba(15,23,42,1)] min-h-0 overflow-hidden">
              <div class="bg-slate-100 border-b-2 border-slate-900 py-1.5 px-4 shrink-0 text-slate-900">
                <span class="text-[9px] font-black tracking-widest uppercase text-slate-600 font-bold">指令互動對策日誌</span>
              </div>
              <n-scrollbar ref="chatScrollRef" class="flex-grow">
                <div class="p-5 space-y-6">
                  <div v-for="(msg, i) in chatMessages" :key="i" class="animate-in slide-in-from-top-1 duration-200">
                    <div :class="msg.role === 'user' ? 'border-r-4 pr-3 text-right border-slate-200' : 'border-l-4 border-slate-900 pl-3 text-left'">
                      <p class="text-[7px] font-black text-slate-400 uppercase mb-1 tracking-tighter">{{ msg.role === 'user' ? 'USER_CMD' : 'CORE_REPLY' }}</p>
                      <p class="text-[15px] font-black leading-[1.5] text-slate-900 uppercase">{{ msg.content }}</p>
                    </div>
                  </div>
                </div>
              </n-scrollbar>
              <div class="flex-none border-t-2 border-slate-900 p-4 bg-slate-50 space-y-2">
                <div class="flex gap-2">
                  <input v-model="userInput" placeholder="請輸入對策指令..." class="flex-grow bg-white border-2 border-slate-900 p-3 text-sm font-black focus:outline-none placeholder:text-slate-200 uppercase text-slate-900" @keyup.enter="sendMessage" />
                  <button @click="sendMessage" :disabled="isWorking || !isLoaded" class="bg-slate-900 text-white px-4 font-black text-[10px] uppercase hover:bg-slate-800 active:bg-slate-700 transition-colors disabled:bg-slate-200 border-none rounded-none text-white">CMD</button>
                </div>
                <div class="flex justify-between items-center text-[7px] font-black text-slate-400 tracking-tighter uppercase">
                  <span>QWEN_LOCAL_CORE</span>
                  <span class="flex items-center gap-1"><span class="w-1 h-1 bg-emerald-500 rounded-full animate-pulse"></span> SECURE_CONNECTION</span>
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
  body { margin: 0; padding: 0; background-color: #ffffff; font-family: 'PingFang TC', 'Microsoft JhengHei', sans-serif; overflow: hidden; color: #0f172a; }
  .font-mono { font-family: 'JetBrains Mono', monospace !important; }
  .n-scrollbar-rail { --n-scrollbar-width: 2px; }
  .chart-container canvas { image-rendering: crisp-edges; }
  input:focus { box-shadow: inset 0 0 0 2px #0f172a; }
  .h-screen { height: 100vh; height: 100dvh; }
  * { color-scheme: light; }
</style>
