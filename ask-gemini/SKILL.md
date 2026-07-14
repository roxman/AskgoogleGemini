---
name: ask-gemini
description: 讓 AI agent 透過使用者「本人已登入」的 Chrome，向 Google 搜尋 AI 模式（udm=50）或 Gemini 提問並擷取答案——不串 Google API（不計費）、不用無頭瀏覽器（會被 Google 擋）。特別適合「大量、逐筆的查證清單」（如判斷一堆比賽是否為歷屆／系列、逐一查冷僻項目的年份與來源）。只在使用者明確要求用 Gemini／Google AI 查、或明講「不想串 API 想用網頁版大量查」時使用；一般問題用 agent 自己的知識或內建搜尋。
---

# Ask Gemini — 用登入版 Chrome 大量查 Google AI（免 API）

## 這是什麼、為什麼
操控使用者本人已登入的 Chrome，向 Google 搜尋的 **AI 模式**（`udm=50`）提問並收割答案。**不用 API**（避免計費）、**不用 Playwright 無頭**（Google 登入會被機器人偵測擋）。消耗的是使用者已付的訂閱額度。核心價值：**要一次查幾百筆、又不想串 Google API 付費**時，這是最省的路。

## 前置條件
1. **Chrome 已登入 Google**（AI 模式要登入；agent 不能代登入）。
2. **agent 能操控該 Chrome**——任何能「導航到 URL」「在頁面上執行 JavaScript」「開／關分頁」的瀏覽器工具皆可。常見搭配見下方映射表。須附掛已登入 Google 的 profile，不要開全新無痕實例。
3. **模型要求（重要）**：跑批次收割＋判讀的 agent 要 **Sonnet 級以上**。小模型（Haiku 級）只能做機械操作，多分頁收割會整批失敗。**不要用 Fable**（週額度小、消耗兇）。

驗收：導航到 `https://www.google.com/search?udm=50&q=test`，能讀到 AI 回答即可用。

## 瀏覽器工具映射

本 skill 用**動作**描述操作，請對應你手上的工具：

| 動作 | Claude in Chrome | chrome-devtools-mcp | Playwright MCP | Hermes 內建 browser |
|------|-----------------|--------------------|-----------------|--------------------|
| 導航到 URL | `navigate` | `navigate_page` | `navigate` | `navigate` |
| 在頁面執行 JS | `javascript_tool` | `evaluate_script` | `evaluate` | `evaluate` |
| 開新分頁 | `tabs_create_mcp` | `new_page` | `new_page` | 視實作 |
| 關閉分頁 | `tabs_close_mcp` | `close_page` | `close` | `close_tab` |
| 列出所有分頁 | `tabs_context_mcp` | `list_pages` | — | — |

找不到完全對應的工具時，用你的瀏覽器工具中功能最接近的即可。

## 觸發
只在使用者「明確」要用 Gemini／Google AI 查、或明講「不想串 API、用網頁版大量查」時使用。未點名時用自己的知識或內建搜尋。

---

## 單發查詢（快速模式）
1. 問題 URL 編碼，直接導航：`https://www.google.com/search?udm=50&q=<編碼後問題>`。**自動送出、免打字**（所以數字不會被吃）。
2. 等約 8–12 秒生成。
3. **收割用 JS（在頁面上執行 JavaScript 擷取），絕不用純文字擷取**（AI 模式頁面用 innerText 類方法常抓不到內容）、**也不要整頁讀取**（單頁約 21k 字元會爆量）。

---

## 大量批次查證（正式主作法・實戰驗證 553 筆全清）

**原理**：問題編成短句、直接進網址自動送出（免打字→可真並行），收割用一支 JS 一次抓齊。

**問題模板**（先清洗名稱、去掉原本的年份／屆數，只留純名稱；太籠統時補主辦關鍵字）：
```
『<清洗後名稱>』是歷屆比賽嗎？它有其他年份屆次嗎？列出各屆年份與來源網址
```
（此為「判斷是否歷屆/系列」的範例；換成你的查證問題即可。數字放網址內，URL 編碼不吃數字。）

**流程（一批 N 筆，N ≤ 10）**：
1. **開 N 個分頁**，每個分頁導航到 `…/search?udm=50&q=<編碼(短句)>`。若工具支援批次操作，盡量一次送出。等約 12 秒讓 AI 生成。
2. **逐分頁執行收割 JS**（見下），一次拿到「年份＋結論句＋真來源 href」，並在最後 `replaceState` 洗掉網址的 mstk 追蹤碼。若工具支援批次執行，盡量合併。
3. **收割完立刻關閉全部分頁**（網址一關就不再被回音）。
4. 判讀、回填放最後做（不碰瀏覽器）。

**收割 JS 範本**（回傳精簡 JSON；末行 replaceState 縮短後續分頁清單回音）：
```js
(() => {
  const seen=new Set(), links=[];
  document.querySelectorAll('a[href^="http"]').forEach(a=>{
    const h=a.href;
    if(/google\.|gstatic|schema\.org|policies|support\.|accounts\./.test(h))return;
    if(seen.has(h))return; seen.add(h);
    const t=(a.innerText||a.textContent||'').trim().replace(/\s+/g,' ').slice(0,45);
    if(t) links.push(t+' | '+h.split('?')[0].slice(0,110));
  });
  const bt=(document.body.innerText||'').replace(/\s+/g,' ');
  const years=[...new Set((bt.match(/20\d{2}/g)||[]))].sort();
  const sents=bt.split(/[。\n]/).map(s=>s.trim()).filter(s=>s.length>6);
  const concl=sents.filter(s=>/歷屆|系列|每年|一次性|僅有|沒有其他|其他年份|多個年份|屆/.test(s)).slice(0,4);
  history.replaceState({}, '', '/search?udm=50&q=x');   // 洗掉 mstk，縮短後續回音（實測有效）
  return JSON.stringify({years, concl, links: links.slice(0,20)});
})()
```
> 實測：一支 JS 取代了多次頁面文字擷取與連結抽取，token 大降、且直接拿到可點網址。版面若改版導致 JS 抓不到，退回逐段文字搜尋（先找結論＋年份、再找來源連結）。

## 省 token 心法（根因）
成本 ≈ **開著分頁數 × 每條網址長度 × 分頁開著期間的工具呼叫次數**（每次工具回傳都會把所有開著分頁的網址印一遍）。四對策：**問題短、猛併批、收割完秒關分頁、回填放最後**。長網址元兇是 Google 自加的 ~200 字元 `mstk`，用上面 JS 的 replaceState 洗掉。

## 規模化實務
- 每批 **≤ 10 分頁**（>10 易觸發 Google 限流）。
- 被 sorry／reCAPTCHA／限流的筆 → 標記待重試、下輪重撿，**不硬刷**；一次多筆被擋就縮到 5 分頁。
- **避開尖峰時段**（消耗較快）。
- 大量作業用**排程每 N 分鐘接手一批 + 鎖檔防重疊 + 逐列「已完成就跳過」防覆蓋**；設自動停止時間。要**無人值守跨數小時自動跑完**，見 `reference-guide.md` 進階章節「**心跳輪詢 ＋ FREE 旗標鎖**」（`*/2` 密集心跳 ＋ 檔案互斥鎖 ＋ 7 分租期防當機死鎖，實測比 `*/10` 快約 3 倍）。
- **單題 vs 多題（預設單題）**：**預設一律一題一問**（廣度、最快）。只有使用者**明確說有連續多題**、或 agent **開跑前**判定需分層時，才切「**多題 funnel 模式**」（同分頁逐層追問＋逐層關閉、脈絡連續；收割抓頁面尾端；鎖租期調大）。細節見 `reference-guide.md` 進階章節「多題 funnel 模式」。

## 消化與回覆
1. 不原文照貼；先用自己的知識檢查合理性，並列不同觀點。
2. **有真實可點來源＝有搜尋佐證**；只有文字宣稱來源、點不了 → 打折並標「未經佐證」。無可點來源不給最高信心。
3. 標明哪些來自 Gemini/AI 模式，附來源連結。

## 隱私與安全
- 提問會送到 Google：含私人檔案／金鑰／個資時先徵得同意或去識別化。
- 絕不代輸入帳密。
- 大量高頻可能觸發 Google 異常偵測；務必遵守上面的限流退避。
