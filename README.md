# AskgoogleGemini

讓你的 AI agent 透過**你本人已登入的 Chrome**，向 Google 搜尋的 **AI 模式（`udm=50`）** 大量查詢並擷取答案——**不必串 Google API（不計費）、不用無頭瀏覽器（會被 Google 擋）**。

特別適合「**一次要查幾十上百筆的清單**」：逐一判斷冷僻項目、查年份/來源、比對是否為歷屆/系列⋯⋯。實戰用此法跑完 553 筆查證，全清、零漏。

> 這是一個 **Skill（給 AI 看的操作說明書）**，不是能獨立執行的程式。它教一個「已經能操控 Chrome」的 AI *怎麼* 高效查 Google。

---

## ⚠️ 前置需求（沒有這些就跑不起來，務必先看）

1. **你的 AI 必須能操控 Chrome**（最關鍵）：
   - **Claude Code / Cowork**：安裝並配對 **Claude in Chrome** 擴充功能。
   - **其他 agent（Codex / Hermes 等）**：接一個**瀏覽器 MCP**（[chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp) 或 Playwright MCP），且掛在「已登入 Google 的那個 Chrome profile」，不要開全新無痕/無頭實例。
   - 若你的 AI 只是純聊天、沒有瀏覽器控制工具 → 這個 skill 無法執行。
2. **Chrome 開著且已登入 Google**（AI 不能代你登入）。
3. **模型用 Sonnet 級以上**。實測小模型（Haiku 級）能做機械操作，但多分頁收割會整批失敗；**不要用 Fable**（週額度小、消耗兇）。

驗收：叫你的 AI 導航到 `https://www.google.com/search?udm=50&q=test`，能讀到 AI 回答就代表環境 OK。

---

## 安裝（依你的工具擇一）

- **Claude Code**：把 `ask-gemini/` 整個資料夾放到 `~/.claude/skills/ask-gemini/`。
- **Cowork**：用 `ask-gemini.skill` 檔，從 **設定 > Capabilities** 匯入。
- **其他 agent**：把 `ask-gemini/SKILL.md`（或 `reference-guide.md`）的內容/路徑放進該 agent 的常駐 instructions（如 `AGENTS.md`、專案設定），註明「使用者明確要求用 Gemini/Google AI 查時，依此執行」。

---

## 怎麼用（核心兩種，schedule 只是選配）

觸發：**明確**叫你的 AI 用 Gemini/Google AI 查，例如：
- 「幫我用 Google AI 查這個清單，判斷每個是不是歷屆比賽，附來源」
- 「這 30 個名稱逐一去 Google 查有沒有其他年份，回傳表格」

兩種預設用法：
1. **給一份清單** → skill 逐筆組短 query、開分頁、收割、回傳結構化結果到你的對話框。
2. **被另一個 agent 呼叫** → 在流程中把單筆/多筆 query 交給它查、拿回答案再往下走。

> **schedule 不是預設用法**。它只是「清單超大 / 要整夜無人值守 / 要閃額度尖峰」時的**選配外掛**（每 N 分鐘接手一批 + 鎖檔防撞）。一般用途當場跑完即可，不需要 schedule。

---

## 它怎麼做到省 token（重點優化）

- **問題編成短句、直接 URL 編碼進 `q=` 自動送出**（免打字，數字也不會被吃）。
- **收割用一支 JS** 一次抓「年份＋結論＋真來源網址」，取代大量 `find`/`read_page` 呼叫；末尾用 `history.replaceState` 洗掉 Google 自加的 `mstk` 追蹤碼，縮短後續回音。
- **成本公式**：開著分頁數 × 網址長度 × 分頁開著期間呼叫次數 → 對策：問題短、猛併批、收割完秒關分頁、回填放最後。
- 每批 ≤ 10 分頁；被限流/reCAPTCHA 的筆標記重試、不硬刷；避開尖峰時段。

完整方法與 JS 範本見 [`ask-gemini/SKILL.md`](ask-gemini/SKILL.md)，原理與踩坑全紀錄見 [`reference-guide.md`](reference-guide.md)。

---

## 隱私與安全

- 提問內容會送到 Google：含私人檔案/金鑰/個資時先徵得同意或去識別化。
- **絕不代輸入帳號密碼。**
- 這是自動化操作消費者網頁：偶爾查沒問題，大量高頻請務必遵守上面的限流退避；真正超大規模仍建議評估官方 API。

---

## 授權
MIT（可自行調整）
