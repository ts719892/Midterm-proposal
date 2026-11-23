# TRA Insight — StackBlitz / Vue 3 + D3.js + Leaflet

這是一份可以直接放進 StackBlitz（Vue + Vite 專案）的起始範例，針對「臺鐵 2025 年事故與異常事件視覺化分析」製作。

## 如何在 StackBlitz 開啟

1. 開啟 https://vite.new/vue ，建立 Vue + Vite 專案。
2. 解壓縮本專案 ZIP。把本資料夾內的 **所有檔案和資料夾**（`package.json`、`index.html`、`vite.config.js`、`src` 等）拖曳到 StackBlitz 左邊的檔案區，覆蓋預設檔案。**不要只上傳 ZIP 而不解壓縮**。
3. 如果依賴沒有自動安裝，在 StackBlitz Terminal 執行 `npm install`，再執行 `npm run dev`。
4. 等 Preview 顯示「TRA INSIGHT 臺鐵事故與異常事件分析總覽」。
5. 點選月度長條圖的月份、試試月份下拉選單；地圖上點擊車站或搜尋站名。
6. 可將專案儲存並上傳至 GitHub，再使用 StackBlitz 或 Vercel / Cloudflare Pages 部署。

StackBlitz 官方匯入教學：https://developer.stackblitz.com/guides/user-guide/importing-projects

## 使用方式

- **月份篩選**：長條圖顯示全年月份；點某月份後，指標卡及類型圓環更新，長條圖強調選取月份。
- **地圖**：顯示 **246 筆車站位置**；搜尋名稱、代碼或地址；點站點查看詳細資料。
- **透明說明**：地圖只表示車站位置，不是事故熱點，月份篩選不會假裝影響車站位置。

## 資料來源與限制

- `src/data/monthly.json`：根據使用者提供之臺鐵 114 年（2025 年）12 份行車事故（件）月報表彙整，合計 **861 件**：重大 **2 件**、一般 **39 件**、異常 **820 件**。月度依序 78、60、75、56、67、68、99、85、70、72、68、63 件。
- `src/data/stations.json`：根據使用者提供之《車站基本資料集.json》縮減，只保留可展示的車站名稱、代碼、座標與地址，**246 筆**。
- 暫時**沒有**逐筆事故位置資料，所以 **不能宣稱是事故熱點圖**。亦不能依據本資料推論哪個車站事故風險最高。
- 車站位置資料不一定是最新，可能涉及增廢站、改名，正式研究應標示版本並再與官方站點清單核對。
- 月度事件數沒有按列車班次、列車公里或站點客流量校正，僅為次數分析，不等同風險率。
- 地圖底圖使用 OpenStreetMap 圖磚，必須連網才能載入；地圖上保留 OSM attribution，公開網站應遵守 OSM Tile Usage Policy。

## 下一階段：逐筆事故事件

待處理 12 份 PDF 後，可新增 `src/data/events.json`，欄位建議如下：

```json
[
  {
    "event_id": "2025-001",
    "date": "2025-01-01",
    "time": "09:30",
    "event_level": "行車異常事件",
    "location_raw": "原始月報發生地點原文",
    "station_code": null,
    "section_from": null,
    "section_to": null,
    "cause_raw": "",
    "source_month": 1,
    "source_page": null,
    "location_confidence": "unmatched"
  }
]
```

上面只示範 **JSON 格式，不是真實事故紀錄**。轉檔後需人工驗證筆數及位置匹配。站間事件應以區間表示，不可直接填入某一車站當成精確事故位置。

## 技術

- Vue 3（元件、狀態管理）
- D3.js（SVG 月度堆疊長條圖）
- Leaflet（地圖及車站標記）
- Vite（本地及線上開發伺服器）
- OpenStreetMap（底圖圖磚）

專案不需要後端、不需要 API 金鑰、不需要資料庫。`npm run build` 產生可部署的 `dist/`。
