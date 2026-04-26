---
name: git-push
description: Git push 流程規範。當使用者要求 push、推送、推上去、上傳到遠端、發布到 GitHub、把 commit 推上去等情境時觸發。會先檢查受保護分支（master、release/*），列出待推送的 commit，確認遠端與 upstream，並在使用者確認後執行推送。
---

# Git Push Workflow

當使用者要求 push 時，請依以下流程執行：

## 0. 受保護分支檢查（最優先）
- 執行 `git rev-parse --abbrev-ref HEAD` 取得目前分支名稱
- 若分支名稱命中以下任一，**立即中斷推送**：
    - `master`
    - 以 `release/` 開頭（例如 `release/1.0`、`release/hotfix-x`）
- 中斷時輸出明確警告，建議使用者切換到 feature branch（例如 `git checkout -b feat/xxx`）後再重試
- 此規則不可被使用者口頭指令覆寫（公司政策）

## 1. 變更摘要
- 執行 `git log @{u}..HEAD --oneline` 列出待推送的 commit
- 若沒有 upstream 設定，改用 `git log origin/<branch>..HEAD --oneline`，若仍無法判斷，用 `git status` 取代
- 若沒有可推送的 commit → 中斷並告知使用者

## 2. 遠端確認
- 執行 `git remote -v` 確認 origin URL
- 檢查當前分支是否有 upstream：`git rev-parse --abbrev-ref --symbolic-full-name @{u}` 若失敗代表沒設
- 沒設 upstream → 之後 push 改用 `git push -u origin <branch>`

## 3. 使用者確認
- 整理「將推送哪些 commit 到 哪個 remote/branch」摘要展示給使用者
- 取得使用者明確同意（例如 `ok`、`push` 等）後才執行 `git push`

## 4. 執行後驗證
- 顯示 push 結果摘要（哪些 commit 被推送、目標 ref）
- 若失敗，分析錯誤原因並建議解法（不要直接重試）
