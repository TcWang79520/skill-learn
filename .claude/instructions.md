# Git Workflow Skill

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