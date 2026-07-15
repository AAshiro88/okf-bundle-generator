# okf-bundle-generator

A Claude Skill that converts user-supplied source material (text files, Word documents, PDFs, code, chat logs, screenshots, or any other input) into a markdown knowledge bundle conforming to the [Open Knowledge Format (OKF) v0.1 spec](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md), and writes it directly to the user's local filesystem.

## What this Skill does

- Splits source content into concept-level markdown files, each with OKF-compliant YAML frontmatter and a structured body.
- Auto-generates a category folder structure plus `index.md` files at every level.
- Applies a personal, non-OKF extension for image handling: images referenced by a concept file are moved into a category-level `img/` folder, and the original filenames are recorded in the file's frontmatter.
- Supports two additional workflows via `prompts/`: generating OKF documents from structured metadata (table schemas, API specs), and enriching an existing bundle by crawling web pages.

## Design principle: MCP-agnostic

The Skill is written against five abstract filesystem capabilities — **read file, write file, create directory, list directory, move file** — rather than any specific MCP server's tool names. The mapping from these abstractions to concrete tool names for the environment currently in use lives in a single table inside `SKILL.md`. Moving to a different filesystem MCP server only requires updating that one table; nothing else in the Skill needs to change.

If no filesystem MCP is connected, the Skill falls back to Claude's own sandbox tools (`create_file`, `bash_tool`), produces the same structure under `/mnt/user-data/outputs`, and delivers it via `present_files` — the user must then download the result and place it in the intended local folder themselves.

## Repository structure

```
okf-bundle-generator/
├── SKILL.md                          # Entry point: workflow, capability mapping, conventions
├── references/
│   ├── okf-spec-summary.md           # Condensed OKF v0.1 spec (frontmatter, body, index/log rules)
│   ├── image-handling.md             # Personal image-archiving convention (not part of OKF itself)
│   └── advanced-conventions.md       # Optional: references/, metrics/joins reference docs
├── skills/
│   ├── kb-search/REFERENCE.md        # Optional helper: search/read an existing local knowledge base
│   └── fileset-source/REFERENCE.md   # Optional helper: access a markdown fileset via an MCP server
└── prompts/
    ├── enrich-from-metadata.md       # Generate OKF docs from structured metadata (schemas, API specs)
    └── ingest-from-web.md            # Enrich an existing bundle by crawling and summarizing web pages
```

## Core workflow (see `SKILL.md` for full detail)

1. Confirm the output location and bundle name (`<output-location>\<YYYYMMDD>-OKF`, or update an existing bundle in place if one is found).
2. Read all source files and decide on category subfolders.
3. Write one concept `.md` file per knowledge point, following the frontmatter/body rules in `references/okf-spec-summary.md`.
4. Move any images into the relevant category's `img/` folder and record them in frontmatter (`references/image-handling.md`).
5. Generate `index.md` at the bundle root and at every populated subfolder.
6. (Optional) Append a dated entry to `log.md` when updating an existing bundle.
7. Self-check the resulting tree and flag anything ambiguous (unclear `type`, unclassified images, etc.) instead of reporting unconditional success.

## Known gaps / things to be aware of

- The fallback (no filesystem MCP) sandbox path cannot copy a file within the user's own machine — only move. If the user needs the original file to stay in place, this must be handled manually.
- `web_fetch` (used by `ingest-from-web.md`) can only fetch URLs that have already appeared in the conversation; it has no independent crawling capability of its own, and there is no built-in page-count or host-allowlist enforcement — those limits must be tracked manually during the workflow.
- The optional helper skills (`kb-search`, `fileset-source`) assume specific MCP servers (`fileskb`, `md-fileset`); when those aren't connected, the fallback path degrades to listing every file and manually scanning for keywords, which does not scale well to a large knowledge base.

____

# okf-bundle-generator

這是一個 Claude Skill，將使用者提供的來源素材（文字檔、Word、PDF、程式碼、聊天紀錄、螢幕截圖或其他任何素材）轉換成符合 [Open Knowledge Format（OKF）v0.1 規範](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) 的 markdown 知識包，並直接寫入使用者本機的檔案系統。

## 這個 Skill 做什麼

- 將來源內容拆分成一份份 concept 層級的 markdown 文件，每份都帶有符合 OKF 規範的 YAML frontmatter 與結構化 body。
- 自動建立分類資料夾結構，並在每一層產生對應的 `index.md`。
- 套用一條非 OKF 官方規範、屬於個人擴充的圖片處理慣例：concept 文件引用到的圖片會被搬移到該分類底下的 `img/` 資料夾，原始檔名記錄在該文件的 frontmatter 中。
- 透過 `prompts/` 目錄支援另外兩種工作流程：從結構化 metadata（資料表 schema、API 規格）產生 OKF 文件，以及透過爬取網頁擴充既有 bundle。

## 設計原則：MCP-agnostic（不綁定特定 MCP）

此 Skill 是依照五種抽象檔案系統能力撰寫的——**讀檔、寫檔、建立目錄、列出目錄、搬移檔案**——而不是綁定任何特定 MCP server 的工具名稱。這些抽象能力與目前環境實際連接工具名稱之間的對應，集中放在 `SKILL.md` 裡的一張表格。換到別的檔案系統 MCP server 時，只需要更新這一張表，Skill 其餘部分完全不用修改。

若沒有連接任何檔案系統 MCP，此 Skill 會改用 Claude 自己的沙盒工具（`create_file`、`bash_tool`），在 `/mnt/user-data/outputs` 產生相同結構，再透過 `present_files` 交付——使用者需要自行下載結果並放到本機預期的資料夾。

## 專案結構

```
okf-bundle-generator/
├── SKILL.md                          # 入口文件：工作流程、抽象能力對應表、規範慣例
├── references/
│   ├── okf-spec-summary.md           # OKF v0.1 規範摘要（frontmatter、body、index/log 規則）
│   ├── image-handling.md             # 個人圖片歸檔慣例（非 OKF 官方規範的一部分）
│   └── advanced-conventions.md       # 選用：references/ 子目錄、metrics/joins 參考文件慣例
├── skills/
│   ├── kb-search/REFERENCE.md        # 選用輔助技能：搜尋、讀取既有本機知識庫
│   └── fileset-source/REFERENCE.md   # 選用輔助技能：透過 MCP server 存取 markdown 文件目錄
└── prompts/
    ├── enrich-from-metadata.md       # 從結構化 metadata（schema、API 規格）產生 OKF 文件
    └── ingest-from-web.md            # 透過爬取網頁擴充既有 bundle
```

## 核心工作流程（完整內容見 `SKILL.md`）

1. 確認輸出位置與 bundle 名稱（`<輸出位置>\<YYYYMMDD>-OKF`，若既有 bundle 已存在則改為就地更新）。
2. 讀取所有來源檔案，決定分類子目錄。
3. 依照 `references/okf-spec-summary.md` 的 frontmatter／body 規則，為每個知識點寫一份 concept 文件。
4. 將圖片搬移到對應分類底下的 `img/` 資料夾，並記錄到 frontmatter（見 `references/image-handling.md`）。
5. 在 bundle 根目錄與每個有內容的子目錄產生 `index.md`。
6. （選用）更新既有 bundle 時，在 `log.md` 補上帶日期的異動紀錄。
7. 檢查產生的目錄結構，客觀指出分類方式或省略內容裡可能存在的疑點（例如 `type` 判斷不明確、圖片找不到明確歸屬分類），不要無條件回報「已完美產生」。

## 已知的限制與需要注意的地方

- 沒有連接檔案系統 MCP 時的沙盒備援路徑，沒有辦法在使用者自己的電腦上做「複製」，只能搬移。若使用者需要保留原始檔案，這件事需要自己手動處理。
- `ingest-from-web.md` 使用的 `web_fetch` 只能抓取已經出現在對話中的網址，本身沒有獨立的爬取能力，也沒有內建的頁數上限或網域白名單機制，這些限制需要在工作流程中自行計數與遵守。
- 選用的輔助技能（`kb-search`、`fileset-source`）預設會連接特定 MCP server（`fileskb`、`md-fileset`）；沒有連接時，備援方式會退化成列出全部檔案再逐一手動比對關鍵字，面對大型知識庫時擴展性不佳。
