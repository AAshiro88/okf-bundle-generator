# Ingest from Web — 從網頁抓取內容擴充 OKF Bundle

當使用者要求從網頁（文件網站、API reference、技術文章等）抓取內容來充實或擴展現有 OKF bundle 時，遵循此指令。你自行決定爬哪些連結以及每頁如何處理。

> **與原始 reference_agent 的差異**：這份指令原本呼叫 `reference_agent` 的 `fetch_url` 工具，該工具內建 max_pages 預算、allowed-hosts 過濾、深度上限等強制機制，且只在 reference_agent 自己的執行環境裡運作。這裡改用 Claude 內建的 `web_fetch` 工具（`web_fetch` 是 Claude 本身自帶的工具，不是使用者連接的 MCP，所以不需要通用化對應），行為上有一項關鍵限制：**`web_fetch` 只能讀取已經出現在對話中的網址**（使用者提供的網址，或前一次 `web_search` / `web_fetch` 回傳結果中出現的網址）。也就是說，想抓取某頁面的外部連結，必須先 `web_fetch` 該頁本身，讓連結內容出現在對話裡，才能對那些連結再次呼叫 `web_fetch`；不能憑空組出一個沒出現過的網址去抓。此外沒有工具會自動幫你擋預算與網域，這些限制要自己數、自己守。
>
> 本文件中「讀檔」「寫檔」「建立目錄」「列出目錄」等抽象能力，具體對應到哪個工具見 `SKILL.md` 前置條件的對應表；那張表列的是此環境目前連接的 Filesystem MCP 專屬工具名稱，換到別的 MCP 只需要改那張表，這份指令不用跟著改。

## 輸入

使用者訊息應包含：
- **seed URLs** 列表 — 起始網址。
- **max-pages budget** — 這裡沒有工具強制執行，改成自己在對話中計數已經 `web_fetch` 過幾頁，達到上限就停止，不要再抓下一頁。
- 可選的 **allowed hosts** 列表（預設僅允許 seed URL 的 host；抓到的連結若 host 不在清單內，直接跳過，不要呼叫 `web_fetch`）。

## 工作流程

1. 一開始用「列出目錄」了解 bundle 中已有哪些 concept 檔案，將網頁發現的內容路由到對應的概念。
2. 對每個 seed URL 呼叫 `web_fetch`。結果是頁面的文字內容，自行從中識別出對外連結（`web_fetch` 不會像原本的 `fetch_url` 一樣額外回傳結構化的 `links` 欄位，連結需要自己從回傳內容裡辨識）。
3. 從識別出的連結中，挑選少量看起來指向**權威文件**的連結，且主題與既有 concepts 相關、host 在允許清單內。跳過導航列、頁尾、登入頁、About us、行銷頁、cookie/隱私聲明及任何明顯不相關的內容。對每個選中的連結遞迴執行，同時累計已抓取頁數，達到自訂的 max-pages 上限就停止。
4. 對**每個取得的頁面**，決定以下三種處理方式之一：

### 4a. 擴充既有 concept
若頁面描述的主題已有對應的 concept 文件，用「讀檔」讀取當前文件內容，組出**增量擴充**後的版本，再用「寫檔」覆寫（見下方「擴充規則」，覆寫前務必檢查沒有刪掉既有標題與內容）。

### 4b. 建立新的參考文件（Reference）
僅當頁面**完全符合以下四個條件**時：

1. **主題形狀**：定義可被主要 concept 文件**指名引用**的內容（商業實體定義、指標定義、枚舉/狀態碼參考、欄位/參數詞彙表、定價說明、單位/時區/識別碼慣例等）。
2. **非 bundle 層級元資料**：不是 overview、introduction、getting started、quickstart、tutorial、release notes、changelog、roadmap、FAQ 或產品 landing page。
3. **引用測試**：你可以在主要 concept 文件中合理寫出 `See the [X reference](/references/x.md) for ...` 這樣的句子。
4. **重複使用測試**：至少有兩個既有 concept 會引用它，或一個 concept 需要它作為支撐背景。

符合四條件 → 用「建立目錄」確保 `references/` 存在，再用「寫檔」建立新文件（例如 `references/event_parameters.md`），`type: Reference`，`resource` 設為該頁 URL。

不確定時，**寧可跳過**。沒有 `references/` 文件的 bundle 沒問題；充滿 overview/getting_started 的 bundle 是雜訊。

### 4c. 跳過
若頁面不相關、訊號微弱或已涵蓋，跳過。繼續處理下一頁。

5. 停止時機：
   - 自己計數的已抓取頁數達到 max-pages 上限。
   - 已涵蓋 seed site 上的相關資料，繼續抓取效益遞減。

## 結構化擷取：metrics、dimensions、join paths

當取得的頁面包含以下類型的內容時，**必須**將其擷取到對應文件中：

### 彙總指標（Metrics）
- 例如：DAU、conversion rate、revenue per user
- 記錄指標名稱、一行定義、**具體 SQL 表達式**
- **存放位置**：每個指標一份 `references/metrics/<slug>.md`，`type: Reference`、`tags: [metric]`
- 然後在每個相關 table 的主要文件中加上 `# Metrics` 章節，用 bullet 連結到該參考文件

### 維度（Dimensions）
- 例如：`event_name`、`device.category`、`traffic_source.medium`
- 記錄欄位路徑、允許值（若為列舉）、簡短語義描述
- **存放位置**：擁有該欄位的 table 的主要 concept 文件，擴充 `# Schema` 或新增 `# Dimensions` 子章節

### Join 路徑
- 例如：`events_` 與 `users` 透過 `user_pseudo_id` 關聯
- 記錄兩端及具體的 `ON` clause
- **存放位置**：每一對 `references/joins/<a>__<b>.md`（兩個 table 名稱按字母排序，以雙底線連接）
- 然後在兩端 table 的主要文件中加上 `# Joins` 章節

**這三種結構化擷取跳過上述的四道關卡測試**——指標和 join 天生就是 concept 形狀且可重複使用，直接寫入對應目錄。

## 擴充規則（Augmentation rules）

當用「寫檔」更新**已有文件**的概念時，原本的 `write_concept_doc` 會自動擋下「新 body 的 `# Schema` 欄位或 `# Citations` 筆數比既有文件少」這種情況，這裡沒有工具幫忙擋，寫入前自行核對：

1. **Frontmatter 必須保留既有值**：
   - `type`、`title`、`resource` 照抄
   - `tags` 合併（聯集，不取代）
   - `timestamp` 更新為目前時間
   - `description` 可在網頁提供更準確摘要時精煉
2. **Body 的每個 `#` 標題必須出現在新 body 中**，順序與措辭相同。你可以：
   - 在既有標題下擴充內容
   - 在既有列表新增項目
   - 新增子章節（`##`）
   - 在最後新增全新頂層標題
   - 將網頁 URL 附加到 `# Citations`
   - **不得**刪除或重新命名任何 `#` 標題，不得整份取代 body
3. 寫入前自行比對：新 body 的 `# Schema` 欄位數與 `# Citations` 條目數，都不應該比舊版少；若比對後發現變少，先把遺漏的部分補回去再寫入。
4. 若無法遵守規則 2（頁面主題根本不同），不要更新既有 concept，改為建立 `references/<slug>` 文件或在 prose 中 cross-link。

## 風格與誠信

- 只引用你**實際 `web_fetch` 過**的 URL（或文件中已有的 URL），不要發明。
- 具體：使用具體欄位名稱、具體枚舉值、具體範例查詢。
- body 中不含前言、道歉或推理敘述。
- 從網頁取得的內容整理進 concept 文件時，一律改寫成自己的文字，不要整段照抄原網頁文字，避免版權問題。
- 結束時用一句話總結：抓了多少頁面、更新了多少文件、建立了多少參考文件。
