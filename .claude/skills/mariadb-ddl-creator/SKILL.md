---
name: mariadb-ddl-creator
description: 從自然語言描述生成 MariaDB DDL 的工作流程。當使用者要求建表、建立資料表、設計 schema、產生 DDL、寫個 table、create table、generate DDL、MariaDB schema、來個資料表 等情境時觸發。會先釐清模糊需求（NULL、唯一性、外鍵、索引欄位），再依固定規範（InnoDB、utf8mb4、欄位 COMMENT、自動推斷型別、索引建議）輸出 SQL，並在最後附上風險檢查。即使使用者沒有明說「DDL」這三個字，只要意圖是「我想要一張新的表 / 新欄位 / 新索引」也應該觸發。
---

# MariaDB DDL Creator Workflow

當使用者要求生成 MariaDB DDL 時，依下列流程執行。本 skill 只生成 SQL 文字，**不會**自動連線到任何資料庫，也不會自動執行語句。

## 0. 觸發與範圍確認

- 觸發詞（中英混用皆可）：`建表`、`建立資料表`、`設計 schema`、`產生 DDL`、`生成 DDL`、`寫個 table`、`create table`、`generate DDL`、`MariaDB schema`、`來個資料表`
- 即使使用者沒明講「DDL」，只要意圖是「想要新表 / 新欄位 / 新索引」就走本流程
- 確認對象是 **MariaDB**（語法以 MariaDB 10.5+ 為基準，同時相容 MySQL 8.0）。若使用者要的是 PostgreSQL / SQLite / Oracle，明確中止並告知此 skill 僅支援 MariaDB / MySQL。

## 1. 需求釐清（Interactive Clarification）

讀完使用者的自然語言描述後，**在生成 SQL 前**先檢查下列項目。任何一項不明確就先問，問完再生成。**不要猜，也不要用「你應該是這個意思吧」帶過。**

必須問清楚的項目：

1. **表名與命名風格**：使用者沒給表名 → 問；給了但風格不一致（mix camelCase/snake_case）→ 確認偏好。預設使用 `snake_case` 複數（例如 `users`、`orders`）。
2. **NULL 規則**：每個欄位是否允許 NULL？預設策略：
    - 主鍵、外鍵、時間戳記 → `NOT NULL`
    - 業務欄位（姓名、Email 等）→ 預設 `NOT NULL`，但使用者可能有特殊需求 → 若描述中沒講，問一次
3. **唯一性 / 索引需求**：
    - Email、手機、帳號這類欄位是否需要 `UNIQUE`？
    - 哪些欄位會被當作查詢條件（需要 `INDEX`）？
4. **外鍵關聯**：描述中若出現「關聯」、「屬於」、「對應」等字眼 → 問是否要建立 `FOREIGN KEY`，以及 `ON DELETE` / `ON UPDATE` 行為（預設建議 `ON DELETE RESTRICT ON UPDATE CASCADE`）。
5. **時間戳記策略**：是否需要 `created_at` / `updated_at`？預設加上：
    - `created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP`
    - `updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`
6. **軟刪除**：是否需要 `deleted_at`？描述中沒講 → 不主動加，但在風險檢查中提及。
7. **欄位長度疑慮**：使用者只說「姓名」沒說多長 → 採預設 `VARCHAR(255)`，但若描述出現「身分證」、「手機」、「Email」這類有業界慣例長度的欄位 → 主動套用標準（見第 3 段）。

**互動原則**：問題集中問完一次，不要一個一個來回。問題不超過 5 個，超過代表需求太發散，建議使用者先理清。

## 2. 預設規範（不可省略）

**所有 `CREATE TABLE` 都必須符合以下規範**：

- 引擎：`ENGINE=InnoDB`
- 字元集：`DEFAULT CHARSET=utf8mb4`
- 排序規則：`COLLATE=utf8mb4_unicode_ci`
- 表級 `COMMENT`：簡短描述這張表的用途
- **每個欄位都必須有 `COMMENT`**（這是強制規範，不是建議）
- 欄位順序：主鍵在最前 → 業務欄位 → 外鍵 → 時間戳記
- 縮排：4 空白；逗號放在每行尾端

## 3. 型別推斷規則

優先用語意比對，沒命中再用 fallback。

| 語意關鍵字 | 推斷型別 | 備註 |
|-----------|---------|------|
| 主鍵 / id / ID | `BIGINT UNSIGNED AUTO_INCREMENT` | 預設 PK 型別，避免 INT 上限風險 |
| UUID | `CHAR(36)` 或 `BINARY(16)` | 主鍵用 UUID 時提醒效能成本，建議搭配 `created_at` 索引 |
| 姓名 / 名稱 / 標題 | `VARCHAR(255) NOT NULL` | 預設 255，使用者另有要求才調整 |
| Email / 信箱 | `VARCHAR(255) NOT NULL`，建議加 `UNIQUE` | |
| 手機 / 電話 | `VARCHAR(20)` | 國際碼 + 號碼足夠 |
| 身分證 / 護照 | `VARCHAR(20)` | 不存明文時改 `VARBINARY(255)` 並提醒加密 |
| 密碼 | `VARCHAR(255) NOT NULL` | **必須**在 COMMENT 註明「儲存 BCrypt hash，不存明文」並列入風險檢查 |
| 狀態 / status / type | `VARCHAR(32)` 或 `TINYINT UNSIGNED` | 若值是有限列舉，建議用 `TINYINT` + 在 COMMENT 列出 mapping，避免 `ENUM`（變更困難） |
| 是否 / 旗標 / flag | `TINYINT(1) NOT NULL DEFAULT 0` | MariaDB 沒原生 BOOLEAN，TINYINT(1) 是慣例 |
| 金額 / 價格 / 餘額 | `DECIMAL(15,2) NOT NULL` | **絕不**用 FLOAT / DOUBLE（精度問題） |
| 數量 / count | `INT UNSIGNED NOT NULL DEFAULT 0` | |
| 描述 / 說明 / 備註 | `VARCHAR(500)` 或 `TEXT` | 長度 > 500 → 用 `TEXT`，並在風險檢查中提醒 `TEXT` 不能有 default 值 |
| 內文 / content / body | `TEXT` 或 `MEDIUMTEXT` | 預估超過 64KB → 用 `MEDIUMTEXT` |
| JSON / 設定 / metadata | `JSON` | MariaDB 10.2+ 支援，會自動轉成 LONGTEXT |
| 時間 / 日期 | `DATETIME` 或 `DATE` | 不指定時區用 `DATETIME`；需要時區轉換用 `TIMESTAMP` |
| created_at / updated_at | `TIMESTAMP NOT NULL DEFAULT ...` | 見第 1 段策略 |

**Fallback**：完全猜不出來 → 用 `VARCHAR(255)`，並在風險檢查中明列「欄位 X 型別為猜測，請確認」。

## 4. 索引建議邏輯

`CREATE TABLE` 內必須包含：

- `PRIMARY KEY (id)`（或使用者指定的主鍵）
- 所有外鍵欄位的 `INDEX`（MariaDB 不會自動為 FK 欄位建索引）
- 帶有 `UNIQUE` 語意的欄位 → `UNIQUE KEY uk_<table>_<col>`
- 預期會用於 `WHERE` 查詢的欄位 → `INDEX idx_<table>_<col>`

索引命名規範：

- 主鍵：不另外命名（用預設 `PRIMARY`）
- 唯一索引：`uk_<table>_<col1>_<col2>`
- 一般索引：`idx_<table>_<col1>_<col2>`
- 外鍵：`fk_<table>_<ref_table>`

**不要**過度建索引：每個索引都會增加寫入成本與儲存成本。如果使用者沒明說某欄位會被查詢，**只**建立明顯需要的索引（PK、UNIQUE、FK），其他索引以「建議在風險檢查中列出」的方式提供，由使用者決定是否採用。

## 5. 輸出格式（嚴格遵守）

輸出**只能**包含以下兩個區塊，順序固定：

### 區塊一：SQL 程式碼

````
```sql
-- <表名> 的用途簡述（一行）
CREATE TABLE `<table_name>` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主鍵',
    `<col>` <type> [NULL/NOT NULL] [DEFAULT ...] COMMENT '<說明>',
    ...
    `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '建立時間',
    `updated_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新時間',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_<table>_<col>` (`<col>`),
    KEY `idx_<table>_<col>` (`<col>`),
    CONSTRAINT `fk_<table>_<ref>` FOREIGN KEY (`<col>`) REFERENCES `<ref_table>` (`id`) ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='<表的用途>';
```
````

### 區塊二：風險檢查

固定格式，使用者一眼能掃完：

```
## 風險檢查
- [O/X] 索引是否完整：<說明，例如「狀態欄位常被查詢但目前無索引，建議補上 idx_users_status」>
- [O/X] 欄位長度是否合理：<說明，例如「description 用 VARCHAR(500)，若預期 > 500 字應改 TEXT」>
- [O/X] NULL / 預設值：<說明>
- [O/X] 字元集 / 排序規則：<確認是否 utf8mb4_unicode_ci>
- [O/X] 安全性：<例如「password 欄位請務必儲存 BCrypt hash 而非明文」、「身分證號建議加密」>
- [O/X] 命名一致性：<是否符合 snake_case 慣例>
- [建議] <額外可選的優化，例如「考慮增加 deleted_at 做軟刪除」、「若資料量大可考慮 partition」>
```

`[O]` 表示通過，`[X]` 表示需要使用者注意，`[建議]` 為可選優化。

**不要**輸出多餘的客套話、不要解釋型別推斷的全過程，使用者要的是「可以直接複製去跑」的 SQL 加一個快速的健檢。

## 6. 邊界

- **不會**自動連線到任何 DB，**不會**執行 SQL，只生成文字
- **不會**生成 `DROP TABLE` / `TRUNCATE` 等破壞性語句，除非使用者明確要求且確認過後果
- **不會**生成 migration 工具（Flyway / Liquibase）的檔案格式，除非使用者明確要求 → 此時包裝成 `V<timestamp>__<desc>.sql` 並提醒命名規則
- 一次只處理一張表；使用者要建多張表時依序處理，每張獨立輸出（含獨立風險檢查）
- 不對非 MariaDB / MySQL 的方言（PostgreSQL `SERIAL`、Oracle `NUMBER`、SQLite `AUTOINCREMENT` 寫法）做轉換建議
- 若使用者要求修改既有表（`ALTER TABLE`），先請使用者提供原表結構（`SHOW CREATE TABLE` 結果），再產出 `ALTER` 語句；同樣套用第 5 段風險檢查
