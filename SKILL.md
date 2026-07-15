---
name: okf-bundle-generator
description: 當使用者提供文件（文字檔、Word、PDF、程式碼、聊天紀錄、螢幕截圖或任何素材）並要求整理成知識庫、產生 Open Knowledge Format（OKF）bundle、建立知識目錄結構、或提到「OKF」「knowledge bundle」「runbook 整理」「IT support 文件庫」等字眼時使用此技能。即使使用者只是說「幫我把這些文件整理成一個目錄」「幫我把這份 SOP 拆成好幾份文件並建立索引」，只要目標是產生一個可被人或 agent 讀取的 markdown 知識庫，就應觸發此技能。此技能會依照 OKF 規範，將輸入內容轉換成帶 YAML frontmatter 的 markdown 概念文件，自動建立分類資料夾與 index.md，並在來源包含圖片時，把圖片記錄檔名後搬移到對應分類的 img 資料夾下。
---

# OKF Bundle Generator

將使用者提供的文件（或聊天中討論出的知識）轉換成一個符合 Open Knowledge Format（OKF）規範的 markdown 知識包（bundle），並寫入使用者電腦的檔案系統。

這個技能包含以下部分：

1. **OKF 官方規範**（純 markdown + YAML frontmatter 的目錄結構）— 詳見 `references/okf-spec-summary.md`。
2. **個人擴充規則：圖片歸檔**（OKF 規範本身沒有定義圖片處理方式，這是使用者自訂的個人慣例）— 詳見 `references/image-handling.md`。
3. **[選用] 進階慣例**（references/ 子目錄、metrics/joins 參考文件、跨文件共用定義）— 詳見 `references/advanced-conventions.md`。
4. **[選用] 輔助技能** — 用於在產生 bundle 時查閱既有知識庫，詳見 `skills/kb-search/REFERENCE.md` 與 `skills/fileset-source/REFERENCE.md`。
5. **[選用] 進階指令** — 從結構化 metadata 產生 OKF 文件（`prompts/enrich-from-metadata.md`）或從網頁抓取擴充 bundle（`prompts/ingest-from-web.md`）。

## 前置條件

此技能只需要一組能操作使用者本機檔案系統的工具，抽象上共五種能力，後續工作流程一律用這五個抽象能力描述，不綁定特定 MCP server：

| 抽象能力 | 說明 |
|---|---|
| 讀檔 | 讀取單一檔案完整內容 |
| 寫檔 | 建立新檔案或覆寫既有檔案內容 |
| 建立目錄 | 建立資料夾（含巢狀路徑），資料夾已存在時視為成功而非錯誤 |
| 列出目錄 | 列出指定路徑下的檔案與子目錄 |
| 搬移檔案 | 將檔案從一個路徑移動到另一個路徑 |

> **以下對應表是此環境目前的具體工具名稱，屬於這個使用者專屬連接的 Filesystem MCP，換到別的環境／別的檔案系統 MCP server 時，只需要把右欄換成對應工具即可，左欄的抽象能力與本文件其餘部分完全不用改。**
>
> | 抽象能力 | 此環境目前連接的具體工具 |
> |---|---|
> | 讀檔 | `Filesystem:read_text_file` |
> | 寫檔 | `Filesystem:write_file` |
> | 建立目錄 | `Filesystem:create_directory` |
> | 列出目錄 | `Filesystem:list_directory` |
> | 搬移檔案 | `Filesystem:move_file` |

若沒有連接任何檔案系統 MCP，改用 Claude 自己的沙盒工具（`create_file`、`bash_tool`）在 `/mnt/user-data/outputs` 產生同樣結構，再用 `present_files` 交付，並提醒使用者這是產生在 Claude 的暫存環境，需要自行下載後放到本機目錄。

沙盒模式下，「搬移檔案」這個抽象能力沒有現成工具，改用 `bash_tool` 執行 `mv <來源路徑> <目的路徑>`；若來源圖片本來就在使用者本機（不在 Claude 沙盒裡），要先用 `copy_file_user_to_claude` 把圖片複製進沙盒，才能用 `mv` 搬到 bundle 對應的 `img` 資料夾底下。

## 工作流程

### 步驟 1：確認輸出位置與 bundle 名稱

Bundle 根目錄命名慣例為：

```
<輸出位置>\<YYYYMMDD>-OKF
```

`YYYYMMDD` 為產生當天日期（例如今天是 2026-07-09，資料夾就叫 `20260709-OKF`）。

- 若使用者已指定輸出路徑，直接在該路徑下建立 `<日期>-OKF` 資料夾。
- 若沒有指定，先用「列出目錄」確認使用者常用的工作區（例如 `Desktop`）是否存在，並詢問使用者要放在哪裡，而不是自行假設一個路徑。
- 若使用者是要「更新」既有的 OKF bundle（路徑底下已經有 `index.md`），不要重新建立新的日期資料夾，而是在既有 bundle 內新增或修改概念文件，並在 `log.md` 補上更新紀錄（見 §6）。

### 步驟 2：分析來源內容，決定分類（子目錄）

讀取使用者提供的所有來源檔案（文字、Word、PDF、程式碼、既有的 troubleshooting 紀錄等），依主題切成幾個分類，每個分類對應一個子目錄，例如 `it-support`、`network`、`database` 等。

分類判斷原則：

- 一個子目錄大約對應「一個可以獨立瀏覽的知識領域」，不要切得太細（避免只有一份文件的子目錄），也不要把不相關主題塞進同一個子目錄。
- 子目錄名稱使用小寫、連字號分隔的英文（例如 `it-support`），除非使用者要求用中文或其他命名方式。
- 每個分類底下的每一個具體知識點（一支 runbook、一個 API、一個 troubleshooting 步驟、一個資料表）各自對應一份 concept 文件（`.md`）。

### 步驟 3：撰寫 concept 文件

依照 `references/okf-spec-summary.md` 的 frontmatter 與 body 規則，為每個知識點寫一份 markdown：

- frontmatter 必填 `type`；建議依序補上 `title`、`description`、`resource`（若有對應的系統/文件連結）、`tags`、`timestamp`。
- body 用結構化 markdown（標題、清單、表格、程式碼區塊），避免整段散文。
- 若內容有慣例段落（`# Schema`、`# Examples`、`# Citations`），依規範使用。
- 文件之間的關聯用 markdown link 表示，bundle 內部連結一律用「以 `/` 開頭的 bundle 相對路徑」（例如 `[隨附文件](/it-support/some-doc.md)`），不要用絕對磁碟路徑。
- 每份文件寫入前先用「建立目錄」確保上層資料夾存在，再用「寫檔」寫入。

### 步驟 4：處理圖片（個人擴充規則）

依照 `references/image-handling.md` 的規則執行，摘要如下：

1. 找出這次來源裡的所有圖片檔案（截圖、附圖、掃描檔等），記下每一張的原始檔名。
2. 對於圖片所屬的分類子目錄，若還沒有 `img` 資料夾，先用「建立目錄」建立 `<bundle 根目錄>\<分類>\img`。
3. 用「搬移檔案」把圖片從原始位置搬移到 `<分類>\img\<原始檔名>`（保留原始檔名，不要重新命名，除非檔名重複需要加序號避免覆蓋）。
4. 在引用該圖片的 concept 文件裡：
   - body 內用 `![說明文字](img/<原始檔名>)` 的相對路徑插入圖片。
   - frontmatter 補上一個擴充欄位 `images: [<原始檔名>, ...]`，記錄這份文件用到的所有圖片原始檔名（OKF 規範允許 producer 自行擴充欄位，consumer 必須容忍未知欄位）。

### 步驟 5：產生 index.md

依照 OKF §6 規則，在 bundle 根目錄與每一個有內容的子目錄底下各寫一份 `index.md`：

- `index.md` 不能有 frontmatter（唯一例外：bundle 根目錄的 `index.md` 可以有一個只包含 `okf_version: "0.1"` 的 frontmatter，用來宣告版本）。
- body 用一個或多個章節，每個章節底下用項目符號列出該層的概念文件或子目錄，並附上該文件 frontmatter 裡的 `description`：

  ```markdown
  # IT Support

  * [某份 runbook 標題](some-doc.md) - 對應 description
  * [子目錄](sub-category/) - 子目錄簡述
  ```

### 步驟 6（選用）：記錄 log.md

若使用者這次是在既有 bundle 上新增/修改內容，於對應層級的 `log.md`（不存在就建立）追加一段，格式依照 OKF §7：

```markdown
## 2026-07-09
* **Creation**: 新增 [某份文件標題](/it-support/some-doc.md)。
```

### 步驟 7：完成後自我檢查

寫完後，用「列出目錄」逐層檢查產生的目錄結構，並用「讀檔」抽查幾份文件，確認：

- 每一份非保留檔名的 `.md` 都有合法 YAML frontmatter，且 `type` 欄位非空（對應 OKF §9 conformance 規則）。
- 每個有圖片的子目錄底下確實有 `img` 資料夾，且圖片檔案確實已經搬移過去（不是留在原地又漏搬）。
- index.md 裡列出的連結，對應的檔案確實存在。

檢查完後，用一段簡短文字告訴使用者最終產生的目錄樹（不需要每份文件內容都貼出來），並客觀指出這次分類方式或省略掉的資訊裡，使用者可能沒注意到的風險（例如：某份來源文件內容不足以判斷 `type`、某張圖片找不到明確歸屬的分類等），不要迎合式地說「已完美產生」。

## 範例目錄結構

```
20260708-OKF/
├── index.md
└── it-support/
    ├── index.md
    ├── vpn-troubleshooting.md
    └── img/
        └── vpn-error-screenshot.png
```

## 參考文件

### 規範與規則
- `references/okf-spec-summary.md` — OKF v0.1 規範摘要（frontmatter 欄位、body 慣例段落、連結規則、index.md / log.md 格式、conformance 規則）。
- `references/image-handling.md` — 圖片歸檔的個人擴充規則細節與邊界情況處理。
- `references/advanced-conventions.md` — 進階慣例：references/ 子目錄、metrics/joins 參考文件、共用定義提取。對應 `knowledge-catalog` 專案中 `reference_agent` 的實作經驗。

### 輔助技能（選用）
在產生 bundle 時，若需要查閱既有知識庫或文件目錄作為參考素材，可載入以下技能：
- `skills/kb-search/REFERENCE.md` — 從本機 markdown 知識庫中列出、搜尋、讀取文件。
- `skills/fileset-source/REFERENCE.md` — 使用 fileset MCP server 存取 markdown 文件目錄。

### 進階指令（選用）
當使用情境超出「從使用者提供的原始檔案產生 bundle」時，可參考以下指令：
- `prompts/enrich-from-metadata.md` — 從結構化 metadata（資料表 schema、API 規格）產出 OKF concept 文件，含 cross-linking 與 body 段落慣例。工具呼叫已改成本文件前置條件定義的抽象能力，不綁定特定 MCP、也不再依賴 reference_agent 的 Python 工具。
- `prompts/ingest-from-web.md` — 從網頁抓取內容擴充既有 OKF bundle，含 metrics/dimensions/joins 結構化擷取規則。已改用 Claude 內建的 `web_fetch`（這是 Claude 本身就有的工具，不是使用者連接的 MCP，所以不需要通用化），取代原本 `reference_agent` 的 `fetch_url`，需注意 `web_fetch` 只能讀已出現在對話中的網址。
