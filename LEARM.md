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
