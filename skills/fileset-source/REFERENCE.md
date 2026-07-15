---
name: fileset-source
description: >
  使用 fileset 來源在 markdown 文件目錄中尋找相關文件並擷取資訊，作為 bundle 產生的參考素材。
---

`md-fileset` MCP server（或同等功能的工具）提供以下工具來從 markdown 文件目錄階層中擷取相關資訊：

* **list_fileset_contents** — 瀏覽並導航目錄樹，列出指定路徑下的內容，項目可能是檔案或子目錄。對應抽象能力「列出目錄」。
* **read_fileset_file** — 讀取知識庫中某個檔案的完整內容，根據正在產生的文件需求來擷取並摘要相關資訊。對應抽象能力「讀檔」。
* **search_fileset_content** — 搜尋知識庫，回傳符合條件的檔案及對應的行號與行片段。可用來快速比對而不需逐一列出和讀取所有檔案。

使用 fileset 時，建立搜尋查詢（使用簡單的關鍵字查詢搭配個別 token）來找到相關文件，然後讀取檔案取得相關資訊。如果一次查詢沒有結果，嘗試其他關鍵字。

## 若 `md-fileset` 未連接

改用 `SKILL.md` 前置條件定義的抽象能力，行為對照如下（表中的 `Filesystem:*` 是此環境目前連接的專屬工具名稱，換到別的 MCP 只需要換這一欄）：

| 原工具 | 抽象能力 | 此環境目前的具體工具 |
|---|---|---|
| `list_fileset_contents` | 列出目錄 | `Filesystem:list_directory` |
| `read_fileset_file` | 讀檔 | `Filesystem:read_text_file` |
| `search_fileset_content` | 沒有對應能力 | 無 |

沒有關鍵字搜尋能力時，先「列出目錄」列出所有檔案，再逐一「讀檔」後自行比對關鍵字；檔案數量多時，先跟使用者確認要不要縮小搜尋範圍（例如指定子目錄），避免逐一讀取整個知識庫造成大量工具呼叫。

沒有接 `md-fileset` 也沒有需要參考既有知識庫時，這個輔助技能可以整個跳過不用。
