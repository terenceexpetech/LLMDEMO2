# Gemini Project Context: AI Investment Dashboard (Frontend)

這是一個純前端的投資分析工具，結合了本地運行的大型語言模型 (WebLLM) 與金融數據 API (FinMind)，為使用者提供即時的個股分析與投資建議。

## 專案概況 (Project Overview)
- **核心目標**：在瀏覽器中實現「端對端」的 AI 股市分析，不依賴後端伺服器進行模型推理。
- **技術棧**：
  - **框架**：Vue 3 (Composition API, TypeScript)
  - **構建工具**：Vite
  - **UI 元件庫**：Naive UI (主要使用) 與 shadcn-vue (配置中)
  - **樣式**：Tailwind CSS v4
  - **AI 引擎**：`@mlc-ai/web-llm` (使用 WebGPU 執行模型，預設模型：`Qwen2.5-0.5B-Instruct`)
  - **圖表**：Chart.js (透過 `vue-chartjs`)
  - **數據來源**：FinMind API (台股價格、大盤指數)、Google News (經由 rss2json 轉換)

## 核心架構 (Architecture)
- **本地 AI 推理**：透過 WebLLM 在使用者瀏覽器中加載並運行小參數模型 (如 Qwen 0.5B)，保護隱私且節省伺服器成本。
- **數據流**：
  1. 使用者輸入股票代號。
  2. 同時發起 FinMind API (價格) 與 News API (新聞) 請求。
  3. 將獲取的結構化數據 (近五日股價、新聞標題) 組合為 Prompt。
  4. WebLLM 根據數據生成精簡的分析建議。

## 指令與開發 (Commands & Development)
### 常用指令
- **啟動開發伺服器**：`npm run dev`
- **專案構建**：`npm run build` (會執行 `vue-tsc` 進行型別檢查)
- **預覽產出**：`npm run preview`

### 開發慣例
- **型別安全**：嚴格使用 TypeScript，定義 API 回傳值與狀態變數的型別。
- **UI 風格**：目前 `App.vue` 主要採用 Naive UI，樣式調整則配合 Tailwind CSS。
- **API 金鑰**：FinMind API 雖有免費配額，但生產環境建議使用 Token (參閱 `finmindAPI.md`)。
- **瀏覽器要求**：由於使用 WebLLM，使用者瀏覽器必須支援 **WebGPU** (Chrome 113+ 或 Edge)。

## 關鍵檔案
- `src/App.vue`：主要的應用程式邏輯、UI 佈局、API 調用與 AI 推理流程。
- `vite.config.ts`：Vite 設定，包含路徑別名 `@` 與 Tailwind 插件。
- `finmindAPI.md`：FinMind API 的詳細文件與使用說明。
- `components.json`：shadcn-vue 的配置資訊。

## 注意事項 (TODO/Notes)
- [ ] 考慮將 `App.vue` 中的龐大邏輯拆分為 Composable 函數 (如 `useWebLLM`, `useStockData`)。
- [ ] 完善錯誤處理機制，特別是當 WebGPU 不可用或 API 請求失敗時的引導。
- [ ] 增加更多技術指標 (如 MA, RSI) 以供 AI 參考。
