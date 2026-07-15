# OKF v0.1 規範摘要

來源：`knowledge-catalog` 專案 `okf/SPEC.md`。此檔案只摘錄產生 bundle 時實際用得到的規則，完整定義以原始 SPEC.md 為準。

## 名詞

- **Knowledge Bundle**：一個自成一體、有階層的 markdown 知識文件集合，是分發的單位。
- **Concept**：bundle 內的一個知識單元，對應一份 markdown 文件。可以是具體資產（表格、API），也可以是抽象概念（指標、流程、SOP）。
- **Concept ID**：該文件在 bundle 內的路徑，去掉 `.md` 副檔名。例如 `it-support/vpn-troubleshooting.md` 的 concept ID 是 `it-support/vpn-troubleshooting`。
- **Frontmatter**：檔案開頭用 `---` 包起來的 YAML 區塊。
- **Body**：frontmatter 之後的所有內容。

## 保留檔名

以下檔名在任何層級都有固定意義，不能拿來當作一般 concept 文件的檔名：

| 檔名 | 用途 |
|---|---|
| `index.md` | 目錄列表（見下方 Index 規則） |
| `log.md` | 異動歷史紀錄（見下方 Log 規則） |

## Concept 文件格式

每份 concept 文件是 UTF-8 markdown，分成 frontmatter 與 body 兩部分。

### Frontmatter

```yaml
---
type: <類型名稱>                    # 必填
title: <顯示名稱>                   # 建議
description: <一句話摘要>           # 建議
resource: <對應資產的 URI>          # 選填，抽象概念可省略
tags: [<標籤>, <標籤>, ...]         # 選填
timestamp: <ISO 8601 日期時間>      # 選填，最後修改時間
# ... producer 可自行擴充其他 key/value
---
```

- `type` 是唯一必填欄位，用來讓 consumer 做路由、篩選、呈現。值不用註冊，可以自由訂，例如 `BigQuery Table`、`Playbook`、`Reference`、`Troubleshooting Guide`。consumer 必須容忍未知的 `type` 值。
- `resource` 只有在這個 concept 對應到一個實際系統/資產時才填，純粹的流程、知識、SOP 類文件通常不需要。
- producer 可以自行加任何額外欄位（例如本技能用到的 `images` 欄位），consumer 必須保留並容忍未知欄位。

### Body 慣例段落

body 是自由格式 markdown，優先用標題、清單、表格、程式碼區塊而不是整段散文。以下標題有慣例意義，符合情境時建議使用：

| 標題 | 用途 |
|---|---|
| `# Schema` | 資產欄位/欄位結構的結構化描述 |
| `# Examples` | 具體使用範例，通常是程式碼區塊 |
| `# Citations` | 支撐 body 內容的外部來源引用 |

沒有規定 body 一定要有哪些段落，依內容需要決定。

## 連結規則

- **Bundle 相對路徑（推薦）**：以 `/` 開頭，相對於 bundle 根目錄，例如 `[customers](/tables/customers.md)`。這種寫法在文件被搬到別的子目錄時仍然穩定，優先使用。
- **一般相對路徑**：標準 markdown 相對路徑，例如 `[neighboring](./other.md)`。
- 連結代表「兩個概念之間有關係」，關係的種類（parent/child、references、joins-with...）由連結周圍的文字表達，不是由連結本身表達。
- consumer 必須容忍失效連結（連結目標不存在不代表文件有問題，可能只是還沒寫）。

## Index 檔案（`index.md`）

- 可以出現在任何層級（含 bundle 根目錄），用來列出該層有哪些東西，支援漸進式揭露。
- **沒有 frontmatter**，唯一例外：bundle 根目錄的 `index.md` 可以有一個只包含 `okf_version: "0.1"` 的 frontmatter，用來宣告這個 bundle 遵循的 OKF 版本。
- body 用一或多個章節，每個章節底下列出連結加簡述：

```markdown
# Section / Group Heading

* [Title 1](relative-url-1) - short description of item 1
* [Title 2](relative-url-2) - short description of item 2

# Another Section

* [Subdirectory](subdir/) - short description of the subdirectory
```

- 條目應該附上被連結文件 frontmatter 裡的 `description`。

## Log 檔案（`log.md`，選用）

- 可以出現在任何層級，記錄該層級的異動歷史。
- 格式是依日期分組、由新到舊的清單，日期用 ISO 8601 的 `YYYY-MM-DD`：

```markdown
# Directory Update Log

## 2026-05-22
* **Update**: Added new BigQuery table reference for [Customer Metrics](/tables/customer-metrics.md).
* **Creation**: Established the [Dataplex Playbook](/playbooks/dataplex.md).

## 2026-05-15
* **Initialization**: Created foundational directory structure.
```

- 開頭的粗體字（`**Update**`、`**Creation**`、`**Deprecation**`）是慣例寫法，不是硬性規定。

## Citations（引用）

當 body 內容有引用外部來源支撐論述時，放在文件最底部的 `# Citations` 段落，用編號列表：

```markdown
# Citations

[1] [BigQuery public dataset announcement](https://cloud.google.com/blog/...)
[2] [Internal data quality runbook](https://wiki.acme.internal/data/quality)
```

## Conformance（合規規則，§9）

一個 bundle 符合 OKF v0.1 規範需要滿足：

1. 樹狀結構裡每一個非保留檔名的 `.md` 檔都含有可解析的 YAML frontmatter。
2. 每個 frontmatter 都有非空的 `type` 欄位。
3. 每個出現的保留檔名（`index.md`、`log.md`）都符合上述格式。

以下狀況**不能**成為拒絕 bundle 的理由（consumer 必須容忍）：

- 選填 frontmatter 欄位缺漏。
- 未知的 `type` 值。
- 未知的額外 frontmatter 欄位。
- 失效的內部連結。
- 缺少 `index.md`。

## 完整範例（來自 SPEC.md Appendix A）

```
my_bundle/
├── index.md
├── datasets/
│   ├── index.md
│   └── sales.md
└── tables/
    ├── index.md
    ├── orders.md
    └── customers.md
```

`tables/orders.md`：

```markdown
---
type: BigQuery Table
title: Orders
description: One row per completed customer order.
resource: https://console.cloud.google.com/bigquery?p=acme&d=sales&t=orders
tags: [sales, orders]
timestamp: 2026-05-28T00:00:00Z
---

# Schema

| Column        | Type      | Description                  |
|---------------|-----------|-------------------------------|
| `order_id`    | STRING    | Unique order identifier.     |
| `customer_id` | STRING    | FK to [customers](/tables/customers.md). |
| `total_usd`   | NUMERIC   | Order total in USD.          |

Part of the [sales dataset](/datasets/sales.md).
```
