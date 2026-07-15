# Enrich from Metadata — 從結構化 metadata 產生 OKF 文件

當你需要的不是從使用者提供的原始檔案建立 bundle，而是從**結構化 metadata**（資料表 schema、API 規格、資料集描述等）產出 OKF concept 文件時，遵循此指令。

> **與原始 reference_agent 的差異**：這份指令原本對應 `knowledge-catalog` 專案中 `reference_agent` 的 Python 工具介面（`read_existing_doc`、`read_concept_raw`、`sample_rows`、`list_concepts`、`write_concept_doc`），那些工具跑在獨立的 BigQuery + Gemini 環境裡，這個環境沒有同名工具可以呼叫。以下步驟改用「讀檔」「寫檔」「建立目錄」「列出目錄」這幾個抽象能力描述，具體對應到哪個工具，見 `SKILL.md` 前置條件的對應表（那張表列的是此環境目前連接的 Filesystem MCP 專屬工具名稱，換到別的 MCP 只需要改那張表，這份指令不用跟著改）。

每次處理針對**一個** concept，完成後寫入該 concept 對應的 `.md` 檔一次即可。

## 工作流程

1. **讀取既有文件（若有）**：用「讀檔」讀取 concept 對應的路徑（例如 concept_id 是 `tables/orders`，路徑就是 `<bundle 根目錄>\tables\orders.md`）。
   - 檔案不存在時會得到錯誤，這代表這是新建的 concept，不是異常，直接當作沒有既有內容繼續即可。
   - 有既有內容時，以它為基礎精煉而非整份重寫（同 §「擴充規則」）。
2. **取得結構化 metadata**：reference_agent 原本用 `read_concept_raw(concept_id)` 直接連線 BigQuery 抓 schema。這個環境沒有對應的資料源連線工具，metadata 來源只會是下列兩種之一：
   - 使用者在對話中直接貼上或描述的 schema / API 規格 / 資料集說明。
   - 使用者提供的檔案（例如 schema 匯出檔、OpenAPI yaml/json），用「讀檔」讀取。
   - 若 metadata 不足以支撐文件內容，直接向使用者確認缺少的欄位，不要用猜測或既有經驗補齊。
3. **沒有資料取樣工具**：reference_agent 原本可選用 `sample_rows(concept_id, n=3)` 連線資料源取樣。這個環境沒有對應工具，只能使用使用者已經提供在對話中或檔案裡的範例資料列；沒有提供時，文件裡不要生造範例資料。
4. **了解 bundle 中其他 concept 以便 cross-link**：reference_agent 原本用 `list_concepts()` 向資料源查詢完整清單。這裡改用「列出目錄」（需要看多層就逐層呼叫）列出 bundle 根目錄底下的檔案樹，把每個非保留檔名（`index.md`、`log.md` 以外）的 `.md` 檔路徑去掉副檔名，當作可連結的 concept_id 清單。
5. **組合 OKF 文件並寫入**：
   - 組出 frontmatter（YAML）與 body 字串。
   - 用「建立目錄」確保上層資料夾存在。
   - 用「寫檔」寫入完整內容（frontmatter + `---` + body）。原始工具 `write_concept_doc` 內建了 frontmatter 驗證與「不可縮減既有 schema/citations」的防呆檢查，這裡沒有對應的自動機制，改成人工檢查（見下方「寫入前自我檢查」）。

## 寫入前自我檢查（取代原本 write_concept_doc 的自動防呆）

原本的 `write_concept_doc` 會在寫入前自動擋下兩種情況：frontmatter 缺少必填欄位、或（web enrich 情境下）新 body 的 `# Schema` 欄位或 `# Citations` 筆數比既有文件少。這裡沒有工具幫忙擋，寫入前自行確認：

- `type` 欄位存在且非空字串。
- 若步驟 1 有讀到既有內容，比對舊 body 的 `# Schema` 欄位名稱（反引號包住的欄位）與新 body 是否有遺漏；有遺漏就先把遺漏欄位補回去再寫入，不要直接覆蓋掉。
- 同樣比對 `# Citations` 底下的條目數，新版不應該比舊版少。
- 若上述任一項不滿足，先在回覆中告知使用者差異，不要靜默覆蓋。

## Frontmatter 規則

- `type`：概念類型，例如 `BigQuery Table`、`BigQuery Dataset`、`API Endpoint`、`Playbook`。
- `title`：簡短的人類可讀顯示名稱。
- `description`：**一句話**解釋這個概念，會用在自動產生的 `index.md` 中。
- `timestamp`：留空則手動填入目前的 ISO 8601 時間字串（原本工具會自動帶入，這裡要自己補上，例如 `datetime.now(timezone.utc)` 格式的字串）。
- `resource`（建議）：底層資產的 URI。
- `tags`（建議）：從 metadata 推斷的搜尋標籤 YAML 列表。

## Body 段落順序

1. **簡短描述**（1–3 段）— 說明這個概念是什麼、代表什麼、如何使用。對 table 來說，描述資料粒度（one row per X）、時間範圍、遮罩或抽樣注意事項。
2. **`# Schema`** — 扁平化、可讀的欄位摘要。巢狀 RECORD 欄位用縮排或表格呈現子欄位。明確標註 repeated record。
3. **`# Common query patterns`** — 1 到 3 個簡短的 SQL 範例，用 ` ```sql ` 圍起來，展示這個資產的真實使用方式。只根據使用者提供的 metadata 或已知的實際查詢方式撰寫，不要憑空生造從未見過的查詢。
4. **`# Citations`** — 使用 OKF 引用格式。第一個條目（若存在）放這個 concept 的 `resource` 值，後面跟著任何協助描述的外部 URL。

## Cross-linking 規則

- 使用**相對於當前文件目錄的路徑**（不是以 `/` 開頭的 bundle 根路徑），讓連結在 GitHub 等純檔案瀏覽時也能正確解析。
- 範例（從 `tables/<this_table>.md` 出發）：
  - 同層 table：`[users](users.md)`
  - 上層 dataset：`[dataset](../datasets/<slug>.md)`
  - 參考文件：`[event parameters](../references/event_parameters.md)`
- 只連結步驟 4 列出的 concept_id，不要憑空創造連結目標。
- 不要從標題、程式碼區塊或 schema 欄位名稱列表內連結。

## 風格要求

- 具體。優先使用具體的欄位名稱、枚舉值和範例查詢。
- 不要發明不在提供的 metadata 中的欄位、分區或分片計數。
- body 中不要包含前言、道歉或推理敘述。body 必須是可直接消費的有效 markdown。
