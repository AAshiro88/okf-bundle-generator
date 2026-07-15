# OKF Bundle Generator

An AI skill that converts documents (text files, Word, PDF, source code, chat logs, screenshots, or any material) into [Open Knowledge Format (OKF)](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) knowledge bundles — structured, hierarchical markdown knowledge bases that can be read by both humans and AI agents.

## What It Does

Given raw documents or knowledge discussed in a conversation, this skill:

1. Analyzes content and determines logical categories (subdirectories)
2. Converts each knowledge point into an OKF-compliant concept file (markdown with YAML frontmatter)
3. Moves images into categorized `img/` folders with proper references
4. Generates `index.md` files at each directory level
5. Optionally logs changes in `log.md`

## Output Structure

```
20260709-OKF/
├── index.md                    # Bundle root index
├── it-support/
│   ├── index.md                # Category index
│   ├── vpn-troubleshooting.md  # Concept file
│   └── img/
│       └── vpn-error-screenshot.png
└── network/
    ├── index.md
    └── firewall-rules.md
```

## Project Structure

```
okf-bundle-generator/
├── SKILL.md                          # Main skill definition & workflow
├── prompts/
│   ├── enrich-from-metadata.md       # Generate concept files from structured metadata
│   └── ingest-from-web.md            # Expand bundle by scraping web content
├── references/
│   ├── okf-spec-summary.md           # OKF v0.1 spec summary (frontmatter, body, index, log rules)
│   ├── image-handling.md             # Custom image archiving rules
│   └── advanced-conventions.md       # Advanced conventions (references/, metrics, joins)
└── skills/
    ├── kb-search/
    │   └── REFERENCE.md              # Search existing local markdown knowledge bases
    └── fileset-source/
        └── REFERENCE.md              # Access markdown file directories via fileset MCP
```

## Key Concepts

| Term | Description |
|---|---|
| **Knowledge Bundle** | A self-contained, hierarchical collection of markdown knowledge files |
| **Concept** | A single knowledge unit (one `.md` file) — can be a concrete asset (table, API) or abstract concept (SOP, playbook) |
| **Concept ID** | The file path within the bundle without `.md` extension (e.g., `it-support/vpn-troubleshooting`) |
| **Frontmatter** | YAML block at the top of each concept file (`type` is required) |

## OKF Concept File Format

```yaml
---
type: Troubleshooting Guide    # Required
title: VPN Connection Issues   # Recommended
description: Steps to diagnose VPN failures.  # Recommended
resource: https://...          # Optional - URI of the actual asset
tags: [it-support, vpn]        # Optional
timestamp: 2026-07-09T10:00:00Z  # Optional
images: [error-screenshot.png] # Custom extension for image tracking
---
```

## Prerequisites

The skill requires filesystem tools with these five abstract capabilities:

| Capability | Purpose |
|---|---|
| Read file | Read full content of a single file |
| Write file | Create new or overwrite existing files |
| Create directory | Create folders (including nested paths) |
| List directory | List files and subdirectories at a given path |
| Move file | Move a file from one path to another |

These map to any filesystem MCP server (e.g., `Filesystem:read_text_file`, `Filesystem:write_file`, etc.) or Claude's sandbox tools.

## Workflow

1. **Confirm output location** — Ask user where to place the bundle (default: `<Desktop>/<YYYYMMDD>-OKF/`)
2. **Analyze source content** — Categorize by topic into subdirectories (e.g., `it-support`, `network`, `database`)
3. **Write concept files** — Create one `.md` per knowledge point following OKF frontmatter/body rules
4. **Handle images** — Move images to `<category>/img/`, embed with relative paths, record in frontmatter `images` field
5. **Generate index.md** — Create at bundle root and each category directory
6. **Optional: Log changes** — Append to `log.md` if updating an existing bundle
7. **Self-verify** — Check directory structure, validate frontmatter, confirm image placement, verify links

## Advanced Features

### Enrich from Metadata

Generate concept files from structured data (database schemas, API specs, dataset descriptions). See `prompts/enrich-from-metadata.md`.

### Ingest from Web

Expand an existing bundle by scraping authoritative documentation websites. Supports metrics, dimensions, and join path extraction. See `prompts/ingest-from-web.md`.

### Advanced Conventions

Optional organizational patterns beyond OKF v0.1 spec:
- `references/` subdirectory for shared definitions
- `references/metrics/<slug>.md` for metric definitions with SQL
- `references/joins/<a>__<b>.md` for table join paths
- `# Metrics`, `# Joins`, `# Dimensions` sections in concept files

See `references/advanced-conventions.md`.

## OKF Conformance (v0.1)

A bundle is conformant when:

1. Every non-reserved `.md` file contains parseable YAML frontmatter with a non-empty `type` field
2. Reserved filenames (`index.md`, `log.md`) follow their respective format rules
3. Consumer must tolerate: missing optional frontmatter fields, unknown `type` values, extra frontmatter fields, broken internal links, missing `index.md`

---

# OKF Bundle Generator

一個 AI 技能，將使用者提供的文件（文字檔、Word、PDF、程式碼、聊天紀錄、螢幕截圖或任何素材）轉換成 [Open Knowledge Format (OKF)](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) 知識包 — 結構化、具層級的 markdown 知識庫，可被人類與 AI agent 讀取。

## 功能說明

給定原始文件或對話中討論出的知識，此技能會：

1. 分析內容並決定邏輯分類（子目錄）
2. 將每個知識點轉換為符合 OKF 規範的概念文件（帶 YAML frontmatter 的 markdown）
3. 將圖片移至分類的 `img/` 資料夾並建立正確引用
4. 在每個目錄層級產生 `index.md`
5. 選用：在 `log.md` 輸出異動紀錄

## 輸出結構

```
20260709-OKF/
├── index.md                    # Bundle 根目錄索引
├── it-support/
│   ├── index.md                # 分類索引
│   ├── vpn-troubleshooting.md  # 概念文件
│   └── img/
│       └── vpn-error-screenshot.png
└── network/
    ├── index.md
    └── firewall-rules.md
```

## 專案結構

```
okf-bundle-generator/
├── SKILL.md                          # 主技能定義與工作流程
├── prompts/
│   ├── enrich-from-metadata.md       # 從結構化 metadata 產生概念文件
│   └── ingest-from-web.md            # 從網頁抓取內容擴充 bundle
├── references/
│   ├── okf-spec-summary.md           # OKF v0.1 規範摘要（frontmatter、body、index、log 規則）
│   ├── image-handling.md             # 自訂圖片歸檔規則
│   └── advanced-conventions.md       # 進階慣例（references/、metrics、joins）
└── skills/
    ├── kb-search/
    │   └── REFERENCE.md              # 從本機 markdown 知識庫搜尋文件
    └── fileset-source/
        └── REFERENCE.md              # 透過 fileset MCP 存取 markdown 文件目錄
```

## 關鍵名詞

| 名詞 | 說明 |
|---|---|
| **Knowledge Bundle** | 自成一體、有階層的 markdown 知識文件集合，是分發的單位 |
| **Concept** | bundle 內的一個知識單元，對應一份 `.md` 檔案 — 可以是具體資產（表格、API）或抽象概念（SOP、playbook） |
| **Concept ID** | 該文件在 bundle 內的路徑去掉 `.md` 副檔名（例如 `it-support/vpn-troubleshooting`） |
| **Frontmatter** | 每份概念檔案開頭的 YAML 區塊（`type` 為必填欄位） |

## OKF 概念檔案格式

```yaml
---
type: Troubleshooting Guide    # 必填
title: VPN 連線疑難排解       # 建議
description: VPN 連線失敗時的排查步驟。  # 建議
resource: https://...          # 選填 - 對應資產的 URI
tags: [it-support, vpn]        # 選填
timestamp: 2026-07-09T10:00:00Z  # 選填
images: [error-screenshot.png] # 自訂擴充欄位，記錄圖片檔名
---
```

## 前置條件

此技能需要具備以下五種抽象能力的檔案系統工具：

| 抽象能力 | 說明 |
|---|---|
| 讀檔 | 讀取單一檔案完整內容 |
| 寫檔 | 建立新檔案或覆寫既有檔案 |
| 建立目錄 | 建立資料夾（含巢狀路徑） |
| 列出目錄 | 列出指定路徑下的檔案與子目錄 |
| 搬移檔案 | 將檔案從一個路徑移動到另一個路徑 |

對應到任何檔案系統 MCP server（例如 `Filesystem:read_text_file`、`Filesystem:write_file` 等）或 Claude 的沙盒工具。

## 工作流程

1. **確認輸出位置** — 詢問使用者 bundle 放置位置（預設：`<Desktop>/<YYYYMMDD>-OKF/`）
2. **分析來源內容** — 依主題分類成子目錄（例如 `it-support`、`network`、`database`）
3. **撰寫概念檔案** — 每個知識點建立一份 `.md`，遵循 OKF frontmatter/body 規則
4. **處理圖片** — 將圖片搬移至 `<分類>/img/`，用相對路徑嵌入，在 frontmatter `images` 欄位記錄
5. **產生 index.md** — 在 bundle 根目錄與每個分類目錄建立
6. **選用：記錄異動** — 若為更新既有 bundle，在 `log.md` 追加紀錄
7. **自我驗證** — 檢查目錄結構、驗證 frontmatter、確認圖片放置、檢查連結

## 進階功能

### 從 Metadata 擴充

從結構化資料（資料庫 schema、API 規格、資料集描述）產生概念文件。詳見 `prompts/enrich-from-metadata.md`。

### 從網頁擷取

從權威文件網站抓取內容擴充既有 bundle。支援指標、維度與 join 路徑的結構化擷取。詳見 `prompts/ingest-from-web.md`。

### 進階慣例

超出 OKF v0.1 規範的選用組織方式：
- `references/` 子目錄放置共用定義
- `references/metrics/<slug>.md` 放置含 SQL 的指標定義
- `references/joins/<a>__<b>.md` 放置表格 join 路徑
- 概念文件中使用 `# Metrics`、`# Joins`、`# Dimensions` 章節

詳見 `references/advanced-conventions.md`。

## OKF 合規規則（v0.1）

一個 bundle 符合規範的條件：

1. 每個非保留檔名的 `.md` 檔都含有可解析的 YAML frontmatter，且 `type` 欄位非空
2. 保留檔名（`index.md`、`log.md`）符合各自格式規則
3. Consumer 必須容忍：選填欄位缺漏、未知 `type` 值、未知額外欄位、失效內部連結、缺少 `index.md`
