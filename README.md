# okf-bundle-generator

A Claude Skill that converts user-supplied source material (text files, Word documents, PDFs, code, chat logs, screenshots, or any other input) into a markdown knowledge bundle conforming to the [Open Knowledge Format (OKF) v0.1 spec](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md), and writes it directly to the user's local filesystem.

This README covers three layers: (1) the OKF spec itself, since the Skill's output is only useful if it actually conforms to it; (2) the personal conventions this Skill layers on top of OKF, which are **not** part of the official spec; (3) how the Skill and its files are organized.

## 1. OKF v0.1 essentials (from `SPEC.md`)

OKF is deliberately minimal: a directory of markdown files with YAML frontmatter, no required tooling. The terms below are used throughout this document and the Skill's own files:

- **Knowledge Bundle** — the whole directory tree; the unit of distribution.
- **Concept** — a single knowledge unit, i.e. one markdown file. Can describe a concrete asset (a table, an API) or an abstract one (a metric, a playbook).
- **Concept ID** — the concept's path inside the bundle with `.md` stripped, e.g. `it-support/vpn-troubleshooting.md` → `it-support/vpn-troubleshooting`.
- **Reserved filenames** — `index.md` and `log.md` have fixed meanings at every level and can never be used as a concept's filename.

### Frontmatter

The exact structure defined in `SPEC.md` §4.1:

```yaml
---
type: <Type name>                  # REQUIRED
title: <Optional display name>
description: <Optional one-line summary>
resource: <Optional canonical URI for the underlying asset>
tags: [<tag>, <tag>, …]            # Optional
timestamp: <ISO 8601 datetime>     # Optional last-modified time
# … other producer-defined key/value pairs
---
```

Only `type` is required. Everything else (`title`, `description`, `resource`, `tags`, `timestamp`, and any producer-defined extra key) is optional, and unknown keys/types **must** be tolerated by consumers rather than rejected. `resource` is only filled in when the concept maps to an actual asset/system — pure process or knowledge documents usually omit it.

### Body

Free-form markdown, but structure (headings/lists/tables/code blocks) is preferred over prose. Three heading names carry conventional meaning when used:

| Heading | Purpose |
|---|---|
| `# Schema` | Structured description of an asset's columns/fields. |
| `# Examples` | Concrete usage examples, often as fenced code blocks. |
| `# Citations` | External sources backing claims in the body. |

None of these headings are mandatory — only used when they fit the content. A full concept example bound to a resource, straight from `SPEC.md` §4.3:

```markdown
---
type: BigQuery Table
title: Customer Orders
description: One row per completed customer order across all channels.
resource: https://console.cloud.google.com/bigquery?p=acme&d=sales&t=orders
tags: [sales, orders, revenue]
timestamp: 2026-05-28T14:30:00Z
---

# Schema

| Column        | Type      | Description                              |
|---------------|-----------|-------------------------------------------|
| `order_id`    | STRING    | Globally unique order identifier.        |
| `customer_id` | STRING    | Foreign key into [customers](/tables/customers.md). |
| `total_usd`   | NUMERIC   | Order total in US dollars.               |
| `placed_at`   | TIMESTAMP | When the customer submitted the order.   |

# Joins

Joined with [customers](/tables/customers.md) on `customer_id`.

# Citations

[1] [BigQuery table schema](https://console.cloud.google.com/bigquery?p=acme&d=sales&t=orders)
```

And a concept with no underlying resource (§4.4), showing `resource` simply omitted:

```markdown
---
type: Playbook
title: Incident response — data freshness alert
description: Steps to triage a freshness alert on the orders pipeline.
tags: [oncall, incident]
timestamp: 2026-04-12T09:00:00Z
---

# Trigger

A freshness alert fires when `orders` lags more than 30 minutes behind
its expected SLA. See the [orders table](/tables/orders.md).

# Steps

1. Check the [ingestion job dashboard](https://example.com/dash).
2. …
```

### Links

Two forms: bundle-root-relative (`/tables/customers.md`, recommended because it's stable when files move within the same subdirectory) and plain relative (`./other.md`). A link only asserts "these two concepts are related" — the *kind* of relationship is carried by the surrounding prose, not the link syntax. Broken links are explicitly allowed and must not be treated as a defect.

### `index.md`

May appear at any level, including the bundle root. Has **no frontmatter**, with exactly one exception: the bundle-root `index.md` may carry a frontmatter block containing only `okf_version: "0.1"`, to declare which OKF version the bundle targets. Body is one or more headed sections, each a bullet list of links with the linked concept's `description` alongside (`SPEC.md` §6):

```markdown
# Section / Group Heading

* [Title 1](relative-url-1) - short description of item 1
* [Title 2](relative-url-2) - short description of item 2

# Another Section

* [Subdirectory](subdir/) - short description of the subdirectory
```

### `log.md` (optional)

May appear at any level. Date-grouped (`## YYYY-MM-DD`), newest date first, bullet entries (`SPEC.md` §7):

```markdown
# Directory Update Log

## 2026-05-22
* **Update**: Added new BigQuery table reference for [Customer Metrics](/tables/customer-metrics.md).
* **Creation**: Established the [Dataplex Playbook](/playbooks/dataplex.md).

## 2026-05-15
* **Initialization**: Created foundational directory structure.
* **Update**: Added progressive-disclosure guidelines to the root [index](/index.md).
```

The leading bold word (`**Update**`, `**Creation**`, `**Deprecation**`) is convention, not a hard rule.

### Conformance (§9) — what actually makes a bundle valid

A bundle is OKF-conformant if, and only if:
1. Every non-reserved `.md` file has parseable YAML frontmatter.
2. Every frontmatter block has a non-empty `type`.
3. Every `index.md`/`log.md` present follows the structure above.

Everything else is explicitly **soft guidance** — a bundle is still conformant even with missing optional fields, unknown `type` values, unknown extra frontmatter keys, broken links, or no `index.md` at all. This matters practically: the Skill's own self-check step (see §5 below) only needs to hard-enforce these three rules; everything else is a quality judgment call, not a pass/fail check.

## 2. Personal extensions on top of OKF (not in the official spec)

OKF itself defines no attachment/image handling at all. This Skill layers its own convention on top, documented in `references/image-handling.md`:

- Every image referenced by a concept is **moved** (not copied — the environment's filesystem tools generally don't offer a same-machine copy operation) into `<category>/img/<original-filename>`, keeping the original filename unless it collides, in which case a `-2`, `-3`, … suffix is appended and the user is told which files were renamed.
- The image's original filename is also recorded in the concept's frontmatter under an extension key: `images: [<filename>, ...]`. This is a producer-defined extra key exactly in the sense OKF §4.1 allows.
- An image is filed under whichever category *first* references it. If a second concept in a different category also needs it, that second document links back with a relative path (e.g. `../other-category/img/xxx.png`) instead of duplicating the file, and the user is told this cross-category reference happened.
- A source that is itself just an image with no text still gets its own concept file (frontmatter + a short description) — an image dropped straight into `img/` with nothing pointing to it becomes an untraceable "orphan" and must be avoided.
- If the correct category for an image can't be determined, the Skill either asks the user or picks the closest existing category and explicitly flags the guess in the final summary — it does not silently force a category.
- Non-image attachments (logs, config exports, etc.) are explicitly out of scope for this convention; extending the same pattern to them requires confirming with the user first rather than assuming `img/` doubles as a general attachments folder.

`references/advanced-conventions.md` documents a second, independent set of optional conventions (also not part of OKF itself, carried over from the `knowledge-catalog` project's `reference_agent`), for when a bundle needs shared cross-references:

- `references/<slug>.md` — general reference concepts (glossaries, enum/status-code tables, shared entity definitions).
- `references/metrics/<slug>.md` — one file per aggregate metric, holding the single source-of-truth SQL for it, linked from contributing tables via a `# Metrics` section.
- `references/joins/<a>__<b>.md` — one file per foreign-key relationship between two tables (filename = both table names, alphabetized, joined by `__`), linked from both tables via a `# Joins` section.
- A `# Dimensions` section (either under `# Schema` or standalone) documents field paths, allowed enum values, and their meaning.

These are only used when the source material actually has this shape (shared metrics, explicit join keys, dimension enums) — most bundles produced from plain documents won't need any of them.

## 3. Repository / Skill structure

```
okf-bundle-generator/
├── SKILL.md                          # Entry point: workflow, capability mapping, conventions
├── SPEC.md                           # The official OKF v0.1 spec, copied in verbatim for reference
├── references/
│   ├── okf-spec-summary.md           # Condensed OKF v0.1 spec used operationally by SKILL.md
│   ├── image-handling.md             # Personal image-archiving convention (§2 above)
│   └── advanced-conventions.md       # Optional references/metrics/joins convention (§2 above)
├── skills/
│   ├── kb-search/REFERENCE.md        # Optional helper: search/read an existing local KB via `fileskb` MCP
│   └── fileset-source/REFERENCE.md   # Optional helper: access a markdown fileset via `md-fileset` MCP
└── prompts/
    ├── enrich-from-metadata.md       # Generate OKF docs from structured metadata (schemas, API specs)
    └── ingest-from-web.md            # Enrich an existing bundle by crawling and summarizing web pages
```

`kb-search` and `fileset-source` look similar but target two different, specific MCP servers (`fileskb` vs `md-fileset`); neither is required — either or both can be entirely absent, in which case the Skill falls back to plain "list directory" + "read file" and loses keyword-search capability.

## 4. Design principle: MCP-agnostic

The Skill is written against five abstract filesystem capabilities — **read file, write file, create directory, list directory, move file** — rather than any specific MCP server's tool names. The mapping from these abstractions to concrete tool names for the environment currently in use lives in a single table inside `SKILL.md`. Moving to a different filesystem MCP server only requires updating that one table; nothing else in the Skill needs to change.

If no filesystem MCP is connected, the Skill falls back to Claude's own sandbox tools (`create_file`, `bash_tool`), produces the same structure under `/mnt/user-data/outputs`, and delivers it via `present_files` — the user must then download the result and place it in the intended local folder themselves. In this fallback mode "move file" has no direct tool equivalent either; it's approximated with `bash_tool`'s `mv`, and if the source image is on the user's own machine rather than already in the sandbox, it first has to be pulled in with `copy_file_user_to_claude`.

## 5. Core workflow (`SKILL.md`, full detail there)

1. **Locate/name the bundle.** New bundle: `<output-location>\<YYYYMMDD>-OKF`. If the target path already has an `index.md`, treat it as an existing bundle to update in place rather than creating a new dated folder.
2. **Read all sources and decide categories.** One category ≈ one independently browsable knowledge area; avoid one-document categories and avoid mixing unrelated topics into one category. Category folder names are lowercase, hyphen-separated English unless the user asks otherwise.
3. **Write one concept file per knowledge point**, following the frontmatter/body rules above; internal links use bundle-root-relative paths (`/category/doc.md`).
4. **Handle images** per §2 above.
5. **Generate `index.md`** at the bundle root and every populated subfolder.
6. **(Optional) Append to `log.md`** with a dated entry when updating an existing bundle, not when creating a brand-new one.
7. **Self-check**: re-list the tree, spot-read a few files, confirm every non-reserved `.md` has valid frontmatter with a non-empty `type` (the actual §9 conformance rules — see §1 above), confirm every category with images has an `img/` folder and nothing was left un-moved, confirm `index.md` links resolve. Report the resulting tree plus any judgment calls the user might disagree with (ambiguous `type`, guessed image category) — not an unconditional "done perfectly."

## 6. Optional prompts — when the main workflow doesn't apply

### `prompts/enrich-from-metadata.md`

For generating OKF docs from structured metadata (table schemas, API specs) rather than from raw user-supplied files. Differs from the original `reference_agent` Python tools it's adapted from in three concrete ways:
- No live data-source connection — metadata only comes from what the user pastes/describes or from a file the Skill reads; missing fields are asked about, never guessed.
- No row-sampling tool — example data in the doc only comes from rows the user already provided; none are invented.
- No automatic frontmatter/regression guard on write — the original `write_concept_doc` blocked incomplete frontmatter or a shrinking `# Schema`/`# Citations` on overwrite automatically; here that check is manual (spelled out in the file's "寫入前自我檢查" section) and must be done by hand before every write.

### `prompts/ingest-from-web.md`

For enriching an existing bundle from web pages. Built on Claude's built-in `web_fetch`, which has one hard constraint worth calling out explicitly: **it can only fetch a URL that has already appeared somewhere in the conversation** (either given by the user, or returned by a prior `web_search`/`web_fetch`). To follow a link found on a page, that page must be fetched first so the link text appears in-context — a URL can't be constructed from scratch and fetched directly. There is also no automatic page-count budget or host-allowlist enforcement (unlike the original `fetch_url` tool it replaces); both must be tracked by hand during the crawl. Metrics, dimensions, and join paths found on a page are extracted per the §2 advanced-conventions rules above; anything else follows a four-condition test in the file itself before being turned into a new `references/` document (skip when in doubt).

## 7. Known gaps / things to be aware of

- No same-machine file **copy** in either the filesystem-MCP path or the sandbox fallback — only move. If the user needs the original file to stay in place, that has to be handled manually.
- `web_fetch` cannot crawl independently and has no built-in budget/allowlist enforcement; both are self-tracked, which is easy to lose count of in a long crawl.
- The optional helper skills assume specific MCP servers (`fileskb`, `md-fileset`); without them, search degrades to listing every file and manually scanning for keywords, which doesn't scale to a large knowledge base.
- OKF conformance (§9) is intentionally loose — the Skill's self-check only hard-enforces frontmatter/`type`/reserved-file structure. Category boundaries, `description` quality, and whether something should be a `references/` doc are all judgment calls the Skill makes and states explicitly rather than something it can mechanically verify.

____

# okf-bundle-generator

這是一個 Claude Skill，將使用者提供的來源素材（文字檔、Word、PDF、程式碼、聊天紀錄、螢幕截圖或其他任何素材）轉換成符合 [Open Knowledge Format（OKF）v0.1 規範](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) 的 markdown 知識包，並直接寫入使用者本機的檔案系統。

這份 README 分三層說明：（1）OKF 規範本身——因為這個 Skill 產出的東西是否有用，取決於它是否真的符合規範；（2）這個 Skill 疊加在 OKF 之上、**不屬於**官方規範的個人慣例；（3）Skill 本身與其檔案的組織方式。

## 1. OKF v0.1 重點摘要（來自 `SPEC.md`）

OKF 刻意設計得很極簡：一個由 markdown 檔案加 YAML frontmatter 組成的目錄，不需要任何強制工具。以下名詞會貫穿本文件與 Skill 自身的檔案：

- **Knowledge Bundle（知識包）**——整個目錄樹，是分發的單位。
- **Concept（概念）**——一個知識單元，對應一份 markdown 檔案。可以描述具體資產（一張表、一個 API），也可以描述抽象概念（一個指標、一份 playbook）。
- **Concept ID**——概念在 bundle 內的路徑，去掉 `.md`，例如 `it-support/vpn-troubleshooting.md` 對應 `it-support/vpn-troubleshooting`。
- **保留檔名**——`index.md` 與 `log.md` 在任何層級都有固定意義，永遠不能拿來當概念文件的檔名。

### Frontmatter

`SPEC.md` §4.1 定義的確切結構：

```yaml
---
type: <Type name>                  # REQUIRED
title: <Optional display name>
description: <Optional one-line summary>
resource: <Optional canonical URI for the underlying asset>
tags: [<tag>, <tag>, …]            # Optional
timestamp: <ISO 8601 datetime>     # Optional last-modified time
# … other producer-defined key/value pairs
---
```

只有 `type` 是必填。其餘（`title`、`description`、`resource`、`tags`、`timestamp`，以及任何 producer 自訂的額外欄位）都是選填，而且未知欄位／未知 `type` 值**必須**被 consumer 容忍，不能因此拒收文件。`resource` 只有在概念對應到實際資產／系統時才填，純粹的流程或知識文件通常不填。

### Body

自由格式 markdown，但優先用結構（標題／清單／表格／程式碼區塊）而不是整段散文。三個標題名稱有慣例意義：

| 標題 | 用途 |
|---|---|
| `# Schema` | 資產欄位/欄位結構的結構化描述 |
| `# Examples` | 具體使用範例，通常是程式碼區塊 |
| `# Citations` | 支撐 body 內容的外部來源引用 |

這些標題都不是強制的——只在內容真的符合時才使用。以下是 `SPEC.md` §4.3 對應到實際資產的完整概念範例：

```markdown
---
type: BigQuery Table
title: Customer Orders
description: One row per completed customer order across all channels.
resource: https://console.cloud.google.com/bigquery?p=acme&d=sales&t=orders
tags: [sales, orders, revenue]
timestamp: 2026-05-28T14:30:00Z
---

# Schema

| Column        | Type      | Description                              |
|---------------|-----------|-------------------------------------------|
| `order_id`    | STRING    | Globally unique order identifier.        |
| `customer_id` | STRING    | Foreign key into [customers](/tables/customers.md). |
| `total_usd`   | NUMERIC   | Order total in US dollars.               |
| `placed_at`   | TIMESTAMP | When the customer submitted the order.   |

# Joins

Joined with [customers](/tables/customers.md) on `customer_id`.

# Citations

[1] [BigQuery table schema](https://console.cloud.google.com/bigquery?p=acme&d=sales&t=orders)
```

以及一份沒有對應實際資產的概念（§4.4），可以看到 `resource` 直接省略：

```markdown
---
type: Playbook
title: Incident response — data freshness alert
description: Steps to triage a freshness alert on the orders pipeline.
tags: [oncall, incident]
timestamp: 2026-04-12T09:00:00Z
---

# Trigger

A freshness alert fires when `orders` lags more than 30 minutes behind
its expected SLA. See the [orders table](/tables/orders.md).

# Steps

1. Check the [ingestion job dashboard](https://example.com/dash).
2. …
```

### 連結

兩種形式：以 bundle 根目錄為基準的絕對路徑（`/tables/customers.md`，推薦，因為文件在同一子目錄內移動時仍然穩定）與一般相對路徑（`./other.md`）。連結只表示「這兩個概念有關聯」——關聯的**種類**由連結周圍的文字表達，不是由連結語法本身表達。失效連結是明確被允許的，不能當成缺陷處理。

### `index.md`

可以出現在任何層級，包含 bundle 根目錄。**沒有 frontmatter**，唯一例外：bundle 根目錄的 `index.md` 可以有一個只包含 `okf_version: "0.1"` 的 frontmatter，用來宣告這個 bundle 遵循的 OKF 版本。Body 是一或多個帶標題的段落，每段底下用項目符號列出連結，並附上被連結概念的 `description`（`SPEC.md` §6）：

```markdown
# Section / Group Heading

* [Title 1](relative-url-1) - short description of item 1
* [Title 2](relative-url-2) - short description of item 2

# Another Section

* [Subdirectory](subdir/) - short description of the subdirectory
```

### `log.md`（選用）

可以出現在任何層級。按日期分組（`## YYYY-MM-DD`），由新到舊，項目符號條列（`SPEC.md` §7）：

```markdown
# Directory Update Log

## 2026-05-22
* **Update**: Added new BigQuery table reference for [Customer Metrics](/tables/customer-metrics.md).
* **Creation**: Established the [Dataplex Playbook](/playbooks/dataplex.md).

## 2026-05-15
* **Initialization**: Created foundational directory structure.
* **Update**: Added progressive-disclosure guidelines to the root [index](/index.md).
```

開頭的粗體字（`**Update**`、`**Creation**`、`**Deprecation**`）是慣例，不是硬性規定。

### 合規規則（§9）——真正決定 bundle 是否有效的條件

一個 bundle 符合 OKF 規範，若且唯若：
1. 樹狀結構中每一個非保留檔名的 `.md` 檔都含有可解析的 YAML frontmatter。
2. 每個 frontmatter 都有非空的 `type` 欄位。
3. 出現的每一個 `index.md`／`log.md` 都符合上述格式。

除此之外的一切都明確屬於**軟性建議**——即使選填欄位缺漏、`type` 值未知、有未知的額外 frontmatter 欄位、連結失效、或完全沒有 `index.md`，bundle 仍然是合規的。這件事在實務上很重要：Skill 自己的檢查步驟（見下方第 5 節）只需要對這三條規則做硬性檢查；其餘都是品質判斷，不是通過／不通過的檢查項。

## 2. 疊加在 OKF 之上的個人慣例（非官方規範）

OKF 本身完全沒有定義任何附件／圖片處理方式。這個 Skill 在其上疊加了自己的慣例，記錄在 `references/image-handling.md`：

- 每張被概念文件引用的圖片都會被**搬移**（不是複製——這個環境的檔案系統工具通常沒有提供同機複製的操作）到 `<分類>\img\<原始檔名>`，保留原始檔名，除非撞名，撞名時加上 `-2`、`-3`……後綴，並在完成後的摘要裡明確告知使用者哪些檔名被改過。
- 圖片的原始檔名同時記錄在該概念文件 frontmatter 的擴充欄位 `images: [<檔名>, ...]` 裡，這正是 OKF §4.1 允許 producer 自訂額外欄位的用法。
- 圖片歸屬到**第一份**引用它的概念所在分類。如果另一個分類的第二份概念也需要它，那份文件改用相對路徑往回連結（例如 `../other-category/img/xxx.png`），不重複存放檔案，並且會告知使用者發生了這種跨分類引用。
- 若來源本身就是一張沒有任何文字說明的圖片，仍然要幫它建一份概念文件（frontmatter + 簡短描述）——直接把圖片丟進 `img/` 卻沒有任何文件指向它，會變成無法追溯的「孤兒圖片」，必須避免。
- 若無法判斷圖片該歸哪個分類，會先詢問使用者，或歸到最接近的既有分類並在最終摘要裡明確標註這是推測，不會悄悄地強行歸類。
- 非圖片附件（log 檔、設定檔匯出等）明確不在這個慣例的涵蓋範圍內；要延伸同樣的模式到這些附件上，需要先跟使用者確認，不能假設 `img/` 也能兼當一般附件資料夾。

`references/advanced-conventions.md` 記錄了第二組獨立的選用慣例（同樣不屬於 OKF 本身，是從 `knowledge-catalog` 專案的 `reference_agent` 沿用過來的），用在 bundle 需要共用跨文件參考時：

- `references/<slug>.md`——一般參考概念（詞彙表、枚舉／狀態碼表、共用實體定義）。
- `references/metrics/<slug>.md`——每個彙總指標一份文件，內含該指標的唯一權威 SQL，由各來源表格透過 `# Metrics` 章節連結過來。
- `references/joins/<a>__<b>.md`——每一對表格間的外鍵關係一份文件（檔名＝兩個表名依字母排序、以 `__` 連接），由兩端表格透過 `# Joins` 章節連結過來。
- `# Dimensions` 章節（可作為 `# Schema` 子章節或獨立章節）記錄欄位路徑、允許的枚舉值與意義。

這些只有在來源素材真的具備這種形狀時（共用指標、明確的 join 鍵、維度枚舉）才會用到——大部分從一般文件產生的 bundle 都用不到。

## 3. 專案 / Skill 結構

```
okf-bundle-generator/
├── SKILL.md                          # 入口文件：工作流程、抽象能力對應表、規範慣例
├── SPEC.md                           # OKF v0.1 官方規範，原文複製於此供對照
├── references/
│   ├── okf-spec-summary.md           # SKILL.md 實際運作時使用的 OKF v0.1 規範摘要
│   ├── image-handling.md             # 個人圖片歸檔慣例（見上方第 2 節）
│   └── advanced-conventions.md       # 選用的 references/metrics/joins 慣例（見上方第 2 節）
├── skills/
│   ├── kb-search/REFERENCE.md        # 選用輔助技能：透過 `fileskb` MCP 搜尋、讀取既有本機知識庫
│   └── fileset-source/REFERENCE.md   # 選用輔助技能：透過 `md-fileset` MCP 存取 markdown 文件目錄
└── prompts/
    ├── enrich-from-metadata.md       # 從結構化 metadata（schema、API 規格）產生 OKF 文件
    └── ingest-from-web.md            # 透過爬取網頁擴充既有 bundle
```

`kb-search` 與 `fileset-source` 看起來相似，但分別對應兩個不同的特定 MCP server（`fileskb` 與 `md-fileset`）；兩者都非必需——可以其中一個或兩個都不接，此時 Skill 會退回單純的「列出目錄」＋「讀檔」，失去關鍵字搜尋能力。

## 4. 設計原則：不綁定特定 MCP

此 Skill 是依照五種抽象檔案系統能力撰寫——**讀檔、寫檔、建立目錄、列出目錄、搬移檔案**——而不是綁定任何特定 MCP server 的工具名稱。這些抽象能力與目前環境實際連接工具名稱之間的對應，集中放在 `SKILL.md` 裡的一張表格。換到別的檔案系統 MCP server 時，只需要更新這一張表，Skill 其餘部分完全不用修改。

若沒有連接任何檔案系統 MCP，Skill 會改用 Claude 自己的沙盒工具（`create_file`、`bash_tool`），在 `/mnt/user-data/outputs` 產生相同結構，再透過 `present_files` 交付——使用者需要自行下載結果並放到本機預期的資料夾。在這個備援模式下，「搬移檔案」也沒有直接對應的工具，改用 `bash_tool` 的 `mv` 模擬；如果來源圖片原本在使用者自己的電腦而不是已經在沙盒裡，得先用 `copy_file_user_to_claude` 拉進沙盒才能搬。

## 5. 核心工作流程（完整內容見 `SKILL.md`）

1. **確認輸出位置與 bundle 名稱。** 新 bundle：`<輸出位置>\<YYYYMMDD>-OKF`。若目標路徑底下已經有 `index.md`，視為既有 bundle 就地更新，不重建新的日期資料夾。
2. **讀取所有來源、決定分類。** 一個分類約略對應「一個可獨立瀏覽的知識領域」，避免只有一份文件的分類，也避免把不相關主題塞進同一分類。分類資料夾名稱用小寫、連字號分隔的英文，除非使用者要求其他命名方式。
3. **每個知識點寫一份概念文件**，依照上方的 frontmatter／body 規則；內部連結用以 bundle 根目錄為基準的絕對路徑（`/category/doc.md`）。
4. **處理圖片**，依照上方第 2 節的規則。
5. **產生 `index.md`**，在 bundle 根目錄與每個有內容的子目錄。
6. **（選用）更新 `log.md`**，只有在更新既有 bundle 時才補上帶日期的紀錄，新建 bundle 時不用。
7. **自我檢查**：重新列出目錄樹、抽查幾份文件，確認每一份非保留檔名的 `.md` 都有合法 frontmatter 且 `type` 非空（也就是真正的 §9 合規規則——見上方第 1 節），確認每個有圖片的分類底下確實有 `img` 資料夾且沒有漏搬的檔案，確認 `index.md` 裡的連結對應的檔案確實存在。回報結果時附上目錄樹，以及使用者可能不同意的判斷（例如 `type` 判斷不明確、圖片分類是推測的）——不是無條件回報「已完美產生」。

## 6. 選用指令——主流程不適用時

### `prompts/enrich-from-metadata.md`

用於從結構化 metadata（資料表 schema、API 規格）而非使用者原始檔案產生 OKF 文件。與原本改編自的 `reference_agent` Python 工具有三個具體差異：
- 沒有連線資料源的工具——metadata 只能來自使用者在對話中貼上／描述的內容，或使用者提供、由 Skill 讀取的檔案；資訊不足時直接詢問使用者，不用猜測補齊。
- 沒有資料取樣工具——文件裡的範例資料只能用使用者已經提供的資料列，不會憑空生造。
- 沒有寫入前的自動防呆——原本的 `write_concept_doc` 會自動擋下 frontmatter 不完整、或覆寫時 `# Schema`／`# Citations` 筆數比既有文件少的情況；這裡改成人工檢查（該檔案裡「寫入前自我檢查」段落），每次寫入前都要手動核對。

### `prompts/ingest-from-web.md`

用於透過網頁內容擴充既有 bundle。建立在 Claude 內建的 `web_fetch` 之上，有一項值得明確指出的硬性限制：**只能抓取已經出現在對話中的網址**（使用者提供的，或前一次 `web_search`／`web_fetch` 回傳結果中出現的）。要追蹤某頁面上的連結，必須先抓取該頁面本身，讓連結文字出現在對話裡——不能憑空組出一個沒出現過的網址直接抓取。也沒有自動的頁數上限或網域白名單機制（不像它取代的原本 `fetch_url` 工具），這兩項限制都要在爬取過程中自己數、自己守。頁面上找到的指標、維度、join 路徑依照上方第 2 節的進階慣例規則擷取；其他內容則依該檔案自己定義的四道條件測試，決定是否要建立新的 `references/` 文件（不確定時寧可跳過）。

## 7. 已知的限制與需要注意的地方

- 不論是檔案系統 MCP 路徑還是沙盒備援路徑，都沒有同機「複製」的能力，只能搬移。若使用者需要保留原始檔案，這件事需要自己手動處理。
- `web_fetch` 無法獨立爬取，也沒有內建的頁數預算或網域白名單機制；兩者都得自行追蹤，長時間爬取時容易數錯或忘記。
- 選用的輔助技能預設會連接特定 MCP server（`fileskb`、`md-fileset`）；沒有連接時，搜尋會退化成列出全部檔案再逐一手動比對關鍵字，面對大型知識庫時擴展性不佳。
- OKF 合規規則（§9）本身刻意寬鬆——Skill 的自我檢查只硬性檢查 frontmatter／`type`／保留檔案結構這三項。分類邊界劃得好不好、`description` 寫得夠不夠精準、某段內容是否該獨立成 `references/` 文件，都是 Skill 做出判斷後明確講出來的主觀選擇，而不是可以機械式驗證的項目。
