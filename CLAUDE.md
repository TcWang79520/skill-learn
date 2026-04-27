# Git Workflow

## 指令路由

| 我說 | 走哪個流程 |
|------|------------|
| `commit` / 提交 / 存一下 / 存檔 | 本檔的「Commit 流程」段 |
| `push` / 推送 / 推上去 / 推到遠端 / 發布到 GitHub / 上傳到 origin | 載入 `.claude/skills/git-push/SKILL.md` 並嚴格依其流程執行 |
| `review java` / 審查 java / 檢查 java / spring boot review / 看一下這段 java | 載入 `.claude/skills/java-code-review/SKILL.md` 並嚴格依其流程執行 |
| `建表` / `建立資料表` / `產生 DDL` / `生成 DDL` / `寫個 table` / `create table` / `MariaDB schema` / 來個資料表 | 載入 `.claude/skills/mariadb-ddl-creator/SKILL.md` 並嚴格依其流程執行 |

觸發詞涵蓋中英文與口語化說法。若我講得很模糊（例如「同步一下」「弄一下」「上去吧」），請先反問確認意圖，再走對應流程。

未來新增 skill 時，請在此表加一列，路由到對應 `.claude/skills/<name>/SKILL.md`。

## 受保護分支檢查（適用 commit 與 push，最優先）

- 在執行任何 `git commit` 或 `git push` 之前，先用 `git rev-parse --abbrev-ref HEAD` 取得目前分支名稱。
- 若分支名稱符合下列任一條件，**立即中斷流程，禁止 commit / push**：
    - `master`
    - 以 `release/` 開頭（例如 `release/1.0`、`release/hotfix-x`）
- 中斷時必須輸出明確警告，並建議使用者切換到功能分支（例如 `git checkout -b feat/xxx`）後再重試。
- 此規則不可被使用者口頭指令覆寫；若使用者堅持要直接推送，請拒絕並提醒這是公司政策。

## Commit 流程

當我輸入 "commit" 或要求你提交代碼時，請嚴格執行以下流程：

1. **變更分析**：執行 `git diff --cached` 讀取已暫存的修改。如果沒有暫存檔案，請先詢問我是否要將所有變更加入暫存。
2. **安全性與 Bug 檢查**：
    - 檢查代碼中是否包含敏感資訊（如 API Keys、密碼、硬編碼的 IP）。
    - 檢查邏輯變更是否可能導致潛在的 NullPointer、無窮迴圈或明顯的語法錯誤。
    - 若發現風險，必須先中斷流程並提示我修正，而非直接 commit。
3. **撰寫 Commit Message**：
    - 採用 **Conventional Commits** 規範（例如 `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`）。
    - 第一行標題不超過 50 字。
    - 若變更較大，請在標題下方加入簡短的列點說明（Body）。
4. **確認與執行**：
    - 向我展示你撰寫的 Commit Message。
    - 徵求我的許可後，執行 `git commit -m "[message]"`。

## Push 流程

由 `git-push` skill 處理。觸發時**必須**讀取 `.claude/skills/git-push/SKILL.md` 並嚴格依其流程執行：

1. 受保護分支檢查
2. 列出待推送 commit
3. 確認遠端與 upstream
4. 取得我的同意
5. 執行 push
6. 驗證結果

不可跳過任何步驟，也不可繞過 skill 直接執行 `git push`。
