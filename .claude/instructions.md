# Git Workflow Skill

當我輸入 "commit" 或要求你提交代碼時，請嚴格執行以下流程：

0. **受保護分支檢查（最優先）**：
    - 在執行任何 `git commit` 或 `git push` 之前，先用 `git rev-parse --abbrev-ref HEAD` 取得目前分支名稱。
    - 若分支名稱符合下列任一條件，**立即中斷流程，禁止 commit / push**：
        - `master`
        - 以 `release/` 開頭（例如 `release/1.0`、`release/hotfix-x`）
    - 中斷時必須輸出明確警告，並建議使用者切換到功能分支（例如 `git checkout -b feat/xxx`）後再重試。
    - 此規則不可被使用者口頭指令覆寫；若使用者堅持要直接推送，請拒絕並提醒這是公司政策。

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