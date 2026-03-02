# 08：Status Lines 與 Output Styles

> 自訂終端機底部的即時狀態列——從基本 git 資訊到 token 用量追蹤，以及 Claude 的回應格式自訂。

## Status Lines

Status lines 在 Claude Code session 期間顯示在終端機底部，提供即時的上下文資訊。本專案包含 9 個漸進式版本：

| 版本 | 特色 | 適用情境 |
|------|------|---------|
| v1 | Git branch、目錄、模型 | 基本資訊 |
| v2 | 最新 prompt（250 字元）、顏色依任務類型 | 追蹤當前任務 |
| v3 | Agent 名稱、模型、最近 3 個 prompt | 多 session 識別 |
| v4 | 自訂 key-value metadata | 專案標記、狀態追蹤 |
| v5 | 模型、成本 ($)、行數變更 (+/-)、session 時長 | 成本監控 |
| v6 | 視覺化 context window 使用量條 | 上下文管理 |
| v7 | Session 計時器、開始/結束時間 | 時間追蹤 |
| v8 | Input/output tokens、cache 統計 | 詳細 token 分析 |
| v9 | Powerline 風格、精簡 | 視覺美觀 |

### 設定方式

在 `.claude/settings.json` 中選擇版本：

```json
{
  "statusLine": {
    "type": "command",
    "command": "uv run $CLAUDE_PROJECT_DIR/.claude/status_lines/status_line_v6.py",
    "padding": 0
  }
}
```

Status line 腳本透過 stdin 接收 JSON 輸入（包含 session 資訊、token 用量等），輸出格式化的文字到 stdout。更新頻率有 300ms 的 throttle。

### Session 資料儲存

Status lines 使用 `.claude/data/sessions/<session_id>.json` 儲存 session 資料：

```json
{
  "session_id": "unique-session-id",
  "prompts": ["第一個 prompt", "第二個 prompt"],
  "agent_name": "Phoenix",
  "extras": {
    "project": "myapp",
    "status": "debugging"
  }
}
```

### Agent 命名

UserPromptSubmit hook 的 `--name-agent` flag 會為每個 session 產生唯一的 agent 名稱：

```
命名優先順序：Ollama（本地）→ Anthropic → OpenAI → 預設名稱
```

名稱是單字、好記的識別詞（如 Phoenix、Sage、Nova），方便在多 session 環境中區分。

### 自訂 Metadata（v4）

使用 `/update_status_line` 指令加入自訂 key-value 對：

```
/update_status_line <session_id> project myapp
/update_status_line <session_id> status debugging
/update_status_line <session_id> environment prod
```

metadata 以青色括號顯示在 status line 末端。

### 任務類型顏色指示（v2/v3）

Status line 根據 prompt 內容自動判斷任務類型並上色：

| 顏色 | 類型 | 觸發關鍵字 |
|------|------|-----------|
| 🔍 紫色 | 分析/搜尋 | analyze, search, find |
| 💡 綠色 | 建立/實作 | create, implement, build |
| 🔧 黃色 | 修復/除錯 | fix, debug, repair |
| 🗑️ 紅色 | 刪除 | delete, remove |
| ❓ 藍色 | 提問 | ? 結尾 |
| 💬 預設 | 一般對話 | 其他 |

## Output Styles

Output styles 修改 Claude 的系統提示來改變回應格式，不影響核心功能。

### 可用的 Styles

| Style | 說明 | 適用場景 |
|-------|------|---------|
| **genui** | 產生嵌入式 CSS 的 HTML | 視覺化輸出、瀏覽器預覽 |
| **table-based** | 所有資訊以 markdown 表格呈現 | 比較、結構化資料、狀態報告 |
| **yaml-structured** | YAML 配置格式 | 設定檔、階層資料 |
| **bullet-points** | 巢狀列表 | 行動項目、文件 |
| **ultra-concise** | 最少文字、最快速度 | 資深開發者、快速原型 |
| **html-structured** | 語意化 HTML5 | Web 文件 |
| **markdown-focused** | 完整 markdown 特性 | 複雜文件、混合內容 |
| **tts-summary** | 透過 ElevenLabs TTS 語音播報 | 語音回饋、無障礙 |

### 使用方式

```
/output-style table-based
```

### 檔案位置

| 位置 | 優先級 |
|------|--------|
| `.claude/output-styles/*.md`（專案） | 較高 |
| `~/.claude/output-styles/*.md`（全域） | 較低 |

### Output Style 檔案結構

每個 style 是一個 markdown 檔案，包含 YAML frontmatter 定義名稱和描述，主體是格式化指示：

```yaml
---
name: ultra-concise
description: Minimal words, maximum speed
---

# Ultra-Concise Style

- 每個回應最多 3 行
- 只列出變更的檔案路徑
- 不解釋為什麼，只說做了什麼
- 程式碼區塊不加註解
```

### GenUI Style 的特殊用途

`genui` style 讓 Claude 產生完整的 HTML 頁面（含嵌入式 CSS），可以直接在瀏覽器中開啟：

```markdown
---
name: genui
description: Generates beautiful HTML with embedded styling
---

# GenUI Output Style

產生自包含的 HTML 檔案：
- 嵌入式 CSS（不依賴外部樣式）
- 響應式設計
- 深色主題
- 適合快速原型的 UI 預覽
```

## 自訂 Slash Commands

除了 output styles，專案還示範了自訂 slash commands（`.claude/commands/*.md`）：

| 指令 | 功能 |
|------|------|
| `/prime` | 分析專案結構與理解程式碼 |
| `/plan_w_team` | 團隊協調的計畫產生（見第 7 篇） |
| `/cook` | 進階任務執行 |
| `/update_status_line` | 動態更新 status line metadata |

## 重點整理

- Status lines 在終端機底部顯示即時資訊，9 個版本從基本到進階
- 設定在 `settings.json` 的 `statusLine` 欄位，透過 stdin/stdout 互動
- Session 資料儲存在 `.claude/data/sessions/`，支援 agent 命名與自訂 metadata
- Output styles 修改 Claude 的回應格式，不影響功能——用 `/output-style` 切換
- GenUI style 可產生自包含的 HTML 頁面，適合快速視覺化原型
- 所有自訂（status lines、output styles、slash commands）都遵循 `.claude/` 目錄結構
