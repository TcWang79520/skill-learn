## TOOL
kimi code
open code + api key
claude code cli + api key

## AI Teams
open code
claude code
roles skills agent


## Multi Agent 
Concurrent Execution

## Claude Code CLI 常用指令 Top 10
1. `claude` — 啟動互動式對話模式
2. `claude "問題或任務"` — 直接帶入提示，執行單次任務
3. `/clear` — 清除對話歷史，重新開始
4. `/help` — 顯示所有可用指令與說明
5. `/init` — 產生 `CLAUDE.md`，讓 Claude 認識專案
6. `/model` — 切換使用的模型（Opus / Sonnet / Haiku）
7. `/config` — 開啟設定介面，調整主題、權限、預設模型
8. `/review` — 對當前變更或 PR 進行程式碼審查
9. `/compact` — 壓縮對話歷史以延長可用 context
10. `claude --resume` — 繼續上一次中斷的對話 session

## CLAUDE.md vs Skill — 兩者的差別

### 一句話區分
| 檔案/資料夾 | 是 Skill 嗎？ | 真正身份 |
|-------------|---------------|----------|
| `CLAUDE.md` | ❌ 不是 | **專案記憶（Project Memory）** |
| `.claude/skills/ui-ux-pro-max/` | ✅ 是 | **Skill / Plugin** |

### CLAUDE.md — 專案記憶
- 每次對話「自動」載入到 context
- 永遠在背景，不需要被呼叫
- 適合放規則、慣例、背景資訊
- 沒有 frontmatter、沒有腳本
- 不能用 `/xxx` 觸發

> 比喻：員工手冊 — 進公司就要看，整天都在腦中。

### Skill — 可呼叫的能力包
- 必須有 `SKILL.md` + YAML frontmatter（`name` + `description`）
- 可以包含 `references/`、`scripts/`、`templates/` 等子資源
- 可以用 `/<skill-name>` 手動呼叫
- Claude 看 `description` 自動判斷要不要載入
- 不用時不佔 context

> 比喻：辦公室工具箱 — 需要才拿出來用，用完放回去。

### Plugin — 多個 Skill 的集合
含 `.claude-plugin/plugin.json` 的資料夾，內部可包多個 skills（例如 `ui-ux-pro-max` 內含 7 個 skills：design、brand、slides、ui-styling 等）。

### 觸發機制比較
```
新對話開啟
    │
    ├─→ CLAUDE.md ──────── 自動載入（永遠在 context）
    │
    └─→ Skills 列表只有「描述」進 context
                                │
                                ├─ 講到關鍵字 → Claude 自動載入對應 skill
                                │
                                └─ 打 /skill-name → 手動觸發
```

### 場景對照
| 你的輸入 | CLAUDE.md | Skill |
|----------|-----------|-------|
| `commit` | ✅ 已在 context，立即套用 Git 流程 | ❌ 不載入 ui-ux-pro-max（無關） |
| `幫我設計 dashboard` | ✅ 在 context（不影響） | ✅ 載入 ui-ux-pro-max（關鍵字命中） |

### 重點記憶
> **CLAUDE.md = 一直在腦中**
> **Skill = 需要才從工具箱拿出來**

---

## Plugin 安裝指南（重建環境必看）

把專案搬到新機器（例如公司網域內的另一台電腦）時，**plugin 不會跟著 git clone 一起來**，因為它們裝在使用者目錄、不在 repo 裡。需要在新機器上重新走一次安裝流程。

### 安裝位置（為什麼不在 repo 裡）

| 內容 | 實際路徑 | 跟著 git 嗎 |
|------|----------|-------------|
| 專案內的 skill（例如 `git-push`、`mariadb-ddl-creator`） | `<repo>/.claude/skills/<name>/` | ✅ 跟著 |
| 透過 `/plugin install` 裝的 plugin（例如 `skill-creator`） | `~/.claude/plugins/cache/<marketplace>/<plugin>/` | ❌ 不跟著（每台機器各自裝） |
| 個人設定 | `~/.claude/settings.json` 或 repo 內 `settings.local.json` | ❌ 不跟著（`.gitignore` 已擋） |

### 標準安裝流程（以 skill-creator 為例）

在 Claude Code 互動模式裡輸入下列指令（輸入 `/` 會跳出指令選單）：

```
1. /plugin marketplace add anthropics/skills
   → 把官方 skills marketplace 加到本機 plugin 來源
   → 成功訊息：Successfully added marketplace: anthropic-agent-skills

2. /plugin install skill-creator
   → 從上面註冊的 marketplace 安裝 skill-creator
   → 成功訊息：✓ Installed skill-creator. Run /reload-plugins to apply.

3. /reload-plugins
   → 讓剛裝的 plugin 在當前 session 立即生效（不必重啟 Claude Code）
```

### 驗證安裝

裝完後輸入 `/` 應該能在指令選單看到 `/skill-creator:skill-creator`。或在對話中說「我要建立一個新 skill」，Claude 應該會自動觸發 skill-creator。

### 常見可用 plugin

| Plugin | Marketplace | 用途 |
|--------|-------------|------|
| `skill-creator` | `anthropics/skills` | 建立 / 改進 / 評測自訂 skill |
| `ui-ux-pro-max` | `anthropics/skills` | UI / UX 設計、dashboard、品牌 |

### 換機器時的完整 checklist

1. `git clone <repo>` 把專案拉下來
2. 確認 `~/.claude/` 目錄存在（首次跑 Claude Code 會自動建）
3. 依上面標準安裝流程重裝需要的 plugin
4. **不要** commit `.claude/skills/<plugin-name>/` 形式的空殼目錄（cwd 副作用，已被 `.gitignore` 擋掉）
5. **不要** commit `settings.local.json`（個人設定，已被 `.gitignore` 擋掉）

### 移除 plugin

```
/plugin uninstall <plugin-name>
```

---

## .gitignore 規則說明

目前 repo 根目錄的 `.gitignore` 主要擋掉：

- `**/settings.local.json` — Claude Code 個人本機設定（權限白名單、debug 開關）
- `.claude/skills/skill-creator/` — plugin 工作目錄殘留（plugin 真實安裝位置在 `~/.claude/plugins/`）
- 編輯器 / OS 雜訊（`.DS_Store`、`.idea/`、`.vscode/` 等）

未來新增其他 plugin 時，若發現 git status 列出 `.claude/skills/<plugin-name>/` 的空殼目錄，照樣加進 `.gitignore` 即可。

