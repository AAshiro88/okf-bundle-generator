# OKF 進階慣例（非官方規範，來自 reference_agent 實作經驗）

此文件記錄 `knowledge-catalog` 專案中 `reference_agent` 實作累積的進階慣例。這些慣例**不是** OKF v0.1 規範的一部分，而是實務上證明有用的組織方式，可視需求選用。

## 1. `references/` 子目錄慣例

除了 bundle 中主要的分類子目錄（如 `tables/`、`datasets/`、`it-support/`），OKF 也允許在 `references/` 下建立各種參考型 concept，用來被主要文件 cross-link 引用。

### 1.1 一般參考文件（`references/<slug>.md`）

適合放：
- 商業實體定義
- 枚舉值/狀態碼參考
- 欄位/參數詞彙表
- 跨多個 table 共用的 enum 定義

Frontmatter 範例：
```yaml
---
type: Reference
title: Event Parameters Reference
description: 所有 Google Analytics 4 event parameter 的完整列表與說明。
resource: https://developers.google.com/analytics/devguides/reporting/data/v1/audience-list
tags: [ga4, reference, event]
---
```

### 1.2 指標參考（`references/metrics/<slug>.md`）

每個彙總指標一份文件，擁有 SQL 表達式的單一來源（single source of truth）。

**存放規則**：
- 目錄：`references/metrics/`
- `type: Reference`、`tags: [metric]`
- Body 包含一行定義、SQL 程式碼區塊、`# Citations`
- 在各 contributing table 的主要文件中用 `# Metrics` 章節連結到此文件

**範例結構**：
```
references/metrics/
├── daily_active_users.md
└── conversion_rate.md
```

**`daily_active_users.md` 範例**：
```markdown
---
type: Reference
title: Daily Active Users (DAU)
description: 每日不重複活躍使用者數，用來追蹤產品日常使用規模。
tags: [metric, engagement]
resource: https://example.com/metrics/dau
---

每日不重複活躍使用者數，以 `user_pseudo_id` 去重計算。

```sql
SELECT
  event_date,
  COUNT(DISTINCT user_pseudo_id) AS daily_active_users
FROM `project.dataset.events_*`
GROUP BY event_date
```

# Citations

[1] [DAU 定義](https://example.com/metrics/dau)
```

### 1.3 Join 路徑參考（`references/joins/<a>__<b>.md`）

每對 table 之間的 foreign-key 關聯一份文件。

**存放規則**：
- 目錄：`references/joins/`
- 檔名規則：兩個 table 名稱按字母排序，以雙底線 `__` 連接（例如 `events___users.md`）
- `type: Reference`、`tags: [join]`
- Body 包含具體的 `ON` clause、何時使用此 join 的說明、`# Citations`
- 在兩端 table 的主要文件中用 `# Joins` 章節連結到此文件

**範例結構**：
```
references/joins/
├── events___users.md
└── orders___products.md
```

**`events___users.md` 範例**：
```markdown
---
type: Reference
title: Events ↔ Users Join
description: 透過 user_pseudo_id 將事件表與使用者表關聯，取得使用者屬性。
tags: [join, ga4]
---

```sql
SELECT *
FROM `project.dataset.events_*` AS e
LEFT JOIN `project.dataset.users` AS u
  ON e.user_pseudo_id = u.user_pseudo_id
```

當需要在事件分析中加入使用者維度（例如使用者所屬國家、裝置類別）時使用此 join。

# Citations

[1] [GA4 資料模型](https://example.com/ga4/data-model)
```

## 2. 主要文件中的慣例章節

當一個 table 或 dataset 有對應的指標、維度或 join 關係時，在主要 concept 文件中加入以下章節：

### 2.1 `# Metrics` 章節

位於 `# Schema` 之後、`# Citations` 之前：

```markdown
# Metrics

- [Daily Active Users](/references/metrics/daily_active_users.md) — DISTINCT user_pseudo_id per day
- [Conversion Rate](/references/metrics/conversion_rate.md) — 完成轉換的使用者比例
```

### 2.2 `# Joins` 章節

```markdown
# Joins

- [users](/references/joins/events___users.md) — join on user_pseudo_id 來附加使用者屬性到事件
```

### 2.3 `# Dimensions` 章節

可作為 `# Schema` 的子章節或獨立的 `# Dimensions`：

```markdown
# Dimensions

| 欄位路徑 | 說明 | 允許值 |
|---|---|---|
| `event_name` | 事件名稱 | `page_view`, `purchase`, `add_to_cart`, ... |
| `device.category` | 裝置類別 | `mobile`, `desktop`, `tablet` |
| `traffic_source.medium` | 流量媒介 | `organic`, `cpc`, `referral`, `email` |
```

## 3. 範例 Bundle 參考

`knowledge-catalog` 專案中的 `okf/bundles/` 目錄下提供了完整的 OKF bundle 範例，可作為實作參考：

| Bundle | 主題 | 特色 |
|---|---|---|
| `ga4` | Google Analytics 4 資料集與表格 | 包含 `references/` 子目錄、`viz.html` 視覺化 |
| `crypto_bitcoin` | 比特幣區塊鏈公開資料集 | 簡潔的兩層結構（datasets / tables） |
| `stackoverflow` | Stack Overflow 公開資料集 | 包含 `references/` 與跨文件連結 |

參考這些範例時，請到 `knowledge-catalog` 專案的 `okf/bundles/` 目錄下閱讀原始檔案。

## 4. 與主技能的工作流程整合

這些進階慣例在以下情境中使用：

- **enrich-from-metadata**：當從結構化 metadata（BigQuery schema 等）產生文件時，若資料包含指標定義或 join 關係，按照 §1.2 與 §1.3 建立參考文件。
- **ingest-from-web**：從網頁擷取內容時，metrics、dimensions、join paths 必須按照 §1.2、§1.3 與 §2 的規則存放。
- **一般 bundle 產生**：當使用者提供的來源文件中包含跨文件的共用定義（例如 enum 列表、共用欄位說明），可考慮提取到 `references/<slug>.md` 而非在每份文件中重複。
