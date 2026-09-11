# CLAUDE.md — product-keyword-cloud（跨境電商關鍵字文字雲產生器）

單檔前端工具，輸入商品名稱（＋選填類別/受眾/賣點），呼叫 BYOK AI 一次產生 40 組跨境電商關鍵字（英文＋繁中對照、1-100 熱度分、四分類），用 `wordcloud2.js` 畫成文字雲，可下載 PNG／CSV／TXT 或複製清單。與 `product-title-generator`（規則式詞庫組標題）、`traffic-rank-estimator`（AI-only 流量估算）是同分類姊妹專案，但本工具**不加序號授權**（純公開免費工具）。

## 定位（與姊妹專案的差異）

- **100% 依賴 AI，無規則式離線備援**：比照 `traffic-rank-estimator`，因為任意商品名稱很難用固定詞庫規則產生有意義的關鍵字。無金鑰時「產生」按鈕仍可點擊，點擊後直接顯示明確錯誤訊息並中止（不是 disabled 到使用者搞不清楚）。
- **不分平台**：不像 `product-title-generator` 分 Amazon／Alibaba 用詞規則，統一輸出一組關鍵字。
- **不加序號授權**：比照 `mandala-thinking`／`scamper-thinking-generator`／`where-what-how-strategy-studio` 的「無授權、無exe」定位，無 `Code.gs`、無 `#licenseGate`。
- **語言切換是純前端行為，不重新呼叫 AI**：`buildKeywordPrompt()` 固定要求 AI 同時回傳 `keyword`（英文）與 `keyword_zh`（繁中）兩個欄位，`STATE.displayLang`（`both`／`en`／`zh`）只控制 `displayText(k)` 這個顯示層函式怎麼組字串，不會觸發新的 API 請求——切換語言時只是把既有的 `STATE.keywords` 重新丟給 `renderWordCloud()`／`renderKeywordTable()`。

## 架構

單一 `index.html`：CSS/JS 全內嵌、無外部資源（除 wordcloud2.js CDN）、無建置步驟。視覺主題是**青綠色系（Teal/Turquoise，呼應「跨境/海洋」意象）**，工作區目前唯一使用這個色系的工具。

- **狀態**（`localStorage` key `pkcState`）：`{fields:{productName,category,targetAudience,sellingPoints}, displayLang, keywords:[], summary, generatedAt}`。`keywords` 陣列本身也存進 state，重新整理頁面後仍看得到已產生的文字雲（已用 Playwright 驗證：reload 後 canvas／table 直接還原，不需重打 AI）。
- **AI 串接**：`callLLM()`/`AI_PROVIDERS` 逐字複製自 `product-title-generator/index.html`（約1042-1141行），**省略了圖片分析用的 `imageDataUrl` 參數**（本工具沒有圖片分析功能，純文字 prompt），四家服務商（Claude/OpenAI/Gemini/OpenRouter）皆為純文字呼叫。金鑰存 `localStorage` key `pkcApiConfig`。
- **Prompt**：`buildKeywordPrompt(fields)` 要求 AI 恰好回傳 40 組 `{keyword, keyword_zh, weight(1-100), category(core/longtail/scenario/painpoint), reason}` ＋一段 60-120 字的 `summary`，並明確告知 weight 是主觀估計非真實搜尋數據（沿用 `traffic-rank-estimator` 的免責語氣句式）。
- **解析與容錯**：`extractJsonObject()` 先剝除 markdown code fence 直接 `JSON.parse`，失敗才退而用正則抓第一個 `{...}` 區塊再 parse（比 `product-title-generator` 的 `parseJsonLoose()` 多一層正則備援，比照 `mandala-thinking` 的 `extractJson()` 手法）。`validateAiKeywords()` 逐筆檢查：`keyword` 非空字串、`weight` 為有限數字（**用 `Math.max(1,Math.min(100,...))` 夾範圍，不是丟棄整筆**）、`category` 落在四合法值內（不合法自動退回 `core`，不丟棄）——只有 `keyword` 缺漏或 `weight` 非數字才會整筆略過（`skipped` 計數）。已用 Playwright 灌入一筆 `weight:999, category:'not-a-real-category'` 的邊界案例驗證：確實被修正成 `weight:100, category:'core'` 而非被丟棄或讓整批失敗。
- **文字雲渲染**：`wordcloud2.js`（CDN `cdn.jsdelivr.net/npm/wordcloud@1.2.3/src/wordcloud2.js`，**npm 套件名稱是 `wordcloud` 不是 `wordcloud2`**，容易誤植）。`renderWordCloud()` 把 `sortedKeywords()`（依 weight 降冪排序，僅影響渲染順序，不 mutate `STATE.keywords` 本身）轉成 `list:[[displayText,weight],...]` 陣列傳給 `WordCloud()`——**務必走 `list` 參數而非丟一段文字自動斷詞**，否則含空格的英文片語／中文詞會被錯誤拆字。`weightFactor` 把 1-100 線性映射到 `[14px,88px]*devicePixelRatio` 而非直接當 px 用（否則最大字會過巨）。顏色用 `color()` callback 依 `category` 對應到固定色票（`CATEGORY_META`），達成「大小=熱度、顏色=詞類」雙重編碼。
- **匯出**：PNG 用 `canvas.toBlob()` 直接匯出（不需要 html2canvas，canvas 本身就是最終畫面）；CSV 沿用 `product-title-generator` 的 `csvCell()`+UTF-8 BOM 寫法；TXT 依目前 `displayLang` 一行一詞；複製清單用 `navigator.clipboard.writeText()`。
- **關鍵字表格**：`renderKeywordTable()` 是**唯讀**表格（依 weight 降冪排序顯示），刻意不加編輯/排序互動——使用者已明確表示不要「AI 產生後手動編輯關鍵字」這個額外功能，保持 MVP 簡單。

## 共用模組（直接複製既有程式碼）

- **跑馬燈**：`#marqueeBar` 逐字複製 `mandala-thinking/index.html` 的 IIFE，共用同一個 Google Apps Script 端點與 Sheet（`localStorage` key 改為 `pkcMarquee`）。
- **PWA**：`manifest.json`／`service-worker.js`／`#installBtn` 邏輯比照既有已加裝 PWA 的專案（network-first + 同源快取備援）。圖示（`icons/`）用 Python PIL 現畫：深色圓角背景＋青色雲朵造型＋四個對應分類色的小圓點（藍/琥珀/珊瑚/青），呼應「彩色加權標籤雲」的工具本質。
- **訪客計數器**：`visitor-badge.laobi.icu`，`page_id=m255525.product-keyword-cloud`。
- **創作者資訊／使用警語**：`manual.html`／`index.html` footer 逐字複製既有共用內容，第一/二點警語措辭改成貼合「AI-only、無離線模式」與「文字雲熱度非真實搜尋數據」。

## 測試方式

沒有真實 API 金鑰可測，用 Playwright 攔截 `window.fetch`（判斷偽造 Claude 回應格式 `{content:[{type:'text',text:JSON.stringify({keywords:[...],summary:'...'})}]}`）已驗證過整條管線：`callLLM → extractJsonObject → validateAiKeywords → renderWordCloud/renderKeywordTable`，含一筆刻意灌入的邊界案例（`weight:999`＋不合法 `category`）確認只修正該筆不影響其他 39 筆。**測完務必把 `window.fetch` 還原**（`window.__origFetch`），因為後續要測匯出功能時若忘記還原，`fetch(blobUrl)` 會被誤導向假 AI 回應——本次實測就踩到這個坑（第一次匯出測試因為忘記還原 fetch，CSV/TXT 內容變成整包 AI JSON payload，重新還原 fetch 後才拿到正確的 blob 內容）。`navigator.clipboard.readText()` 在此沙箱環境會卡在權限彈窗不resolve，測複製功能時改用 stub 掉 `navigator.clipboard.writeText` 記錄呼叫參數，不要嘗試 `readText()` 驗證。

canvas 渲染驗證：`getImageData()` 抽樣比對背景色（`#123640`→`rgb(18,54,64)`），確認有足量像素偏離背景色即代表文字雲確實畫出內容。

無建置/測試指令。修改 `index.html` 後直接用瀏覽器開啟驗證，或暫起 `python -m http.server 8815` 測完關閉。

## Port

**8815**（工作區 8765-8814 已全數占用，本專案是目前最新的空號，已登記進 `.claude/launch.json`）。

## 序號授權（鎖定整個工具，12 個月，2026-09-11 新增，取代原本「不套用序號授權」的決定）

使用者第一版明確要求不加序號授權，但後續改變主意要求加上，比照 `traffic-rank-estimator`／`amazon-listing-generator` 的「鎖整個工具、全螢幕遮罩」模式（非 `member-license-gate` skill 資源檔預設的「僅鎖單一功能的 license-bar」樣式——這個工具核心價值就是單一個 AI 生成動作，沒有值得留白的獨立免費功能，鎖整個工具更貼合實際使用情境）：`#licenseGate` 全螢幕遮罩預設鎖定，驗證通過才加 `.hidden`；載入時一律對後端即時重驗，背景每 20 分鐘重驗一次；`localStorage` key：`pkcLicenseSerial`；剩餘天數徽章 `#licenseBadge` 放在 `.topbar nav` 內。

- **綁定的 Google Sheet**：使用者指定的新試算表「跨境電商工具」（<https://docs.google.com/spreadsheets/d/1sUrMVQ7T3_JOG_vMY1peGWxxxbw8PTxbbXBhj2IPM6s/edit>），固定操作獨立分頁「文字雲」（`SHEET_NAME` 常數）——與同一試算表內「工作表1」（該表既有的任務追蹤/測試序號，跟本工具無關）互不干擾。分頁不存在時 `getLicenseSheet_()` 會在第一次真正執行 `doPost`/`doGet` 時自動 `insertSheet()` 並寫入表頭（序號／開始日期／結束日期）。
- **部署方式**：`clasp create --parentId <SheetID>`（不加 `--type`，且 `clasp create` 會把 `appsscript.json` 重置成預設內容，記得推送前手動改回 `timeZone:"Asia/Taipei"` + `webapp:{executeAs:"USER_DEPLOYING",access:"ANYONE_ANONYMOUS"}`）→ 複製 `Code.gs` → `clasp push --force` → `clasp deploy`，全程在 `.gas-deploy/`（已加入 `.gitignore`，不進版控）內操作，一次成功、未卡複製貼上壞掉的坑。已部署：`LICENSE_CHECK_URL = https://script.google.com/macros/s/AKfycbygIVBDSwx9rnuRyTQe4by7W3f6dMJCjTjMPaLEkkWozMIyFYEOyhRPTQThxYVF75Nf/exec`，Apps Script 編輯器：<https://script.google.com/d/1RR_VbeizI8s5v3drmU95hg4WsFzr0UXVNJuyhFeuOCB0mVxrg9YMuc4U/edit>。
- **⚠️ 尚待使用者完成一次性 OAuth 授權**：部署後尚未經過首次同意流程（用 `curl -sL` 打 `/exec` 網址回傳的是 Google Drive「需要存取權」頁面而非 JSON），前端 `licenseGate` 目前會顯示「無法連線授權伺服器」（已用 Playwright/claude-in-chrome 實測確認 fail-closed 行為正確——不是放行，是明確擋下並顯示原因）。需使用者親自用瀏覽器（登入 tsaimark@gmail.com）開啟上面的 Apps Script 編輯器並執行一次 `doGet` 完成同意畫面，之後才能正式驗證序號；同意完成後，「文字雲」分頁應該會自動出現（含表頭），屆時需在該分頁新增至少一列測試序號（序號欄填值，開始/結束日期留空）才能做端對端驗證。
- 這支後端只做序號驗證，不代理任何付費 API（LLM 串接仍是 BYOK），也不處理跑馬燈（跑馬燈內容抓自工作區既有共用授權伺服器，是另一個不相干的系統）。

## 本次未做（後續視需要再處理）

- 桌面版 exe 未打包。
- 是否推公開 GitHub Pages 部署，依工作區「實驗性新工具部署前先確認」慣例，本次未執行，留待使用者確認。
- 未實測真實 AI 金鑰的端對端呼叫（金鑰驗證邏輯與 UI 骨架已用假回應測過，真實金鑰測試留給使用者自行操作，目前卡在序號授權閘門需先完成 OAuth 授權才能進入工具本體）。
- 序號授權後端的一次性 OAuth 授權尚未完成，「文字雲」分頁尚未實際建立、尚無可測試的真實序號。
