# 01：Claude Code Hooks 精通指南介紹

> Claude Code Hooks 的定位、架構、UV single-file scripts 模式，以及本教學的涵蓋範圍。

## Claude Code Hooks 是什麼

Hooks 是 Claude Code 提供的**確定性控制機制**。它們在 Claude 的工作流程中的特定時間點觸發，執行你定義的外部指令。與 LLM 的非確定性行為不同，hooks 保證在正確的時機執行你指定的邏輯——無論是安全驗證、日誌記錄、程式碼品質檢查，還是工作流程控制。

核心能力：

- **攔截與阻止** — 在工具執行前阻擋危險操作（`rm -rf`、存取 `.env`）
- **注入上下文** — 在 prompt 送到 Claude 之前附加專案資訊
- **流程控制** — 阻止 Claude 在任務未完成時停止回應
- **記錄與稽核** — 記錄所有 prompt、工具使用、錯誤事件
- **自動驗證** — 在檔案寫入後自動執行 linting 與型別檢查

## 架構概覽

```
你的 Prompt → UserPromptSubmit hook → Claude 處理
                                         │
                                    PreToolUse hook → 工具執行 → PostToolUse hook
                                         │
                                    Stop hook → 回應完成
```

所有 hooks 設定在 `.claude/settings.json` 中，以 JSON 格式定義每個事件對應的指令：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "uv run $CLAUDE_PROJECT_DIR/.claude/hooks/pre_tool_use.py"
          }
        ]
      }
    ]
  }
}
```

| 設定項 | 說明 |
|--------|------|
| `matcher` | 篩選條件（空字串 `""` 匹配所有工具）。可設定特定工具名稱如 `"Write\|Edit"` |
| `type` | 固定為 `"command"` |
| `command` | 要執行的指令。使用 `$CLAUDE_PROJECT_DIR` 確保路徑在任何工作目錄下都正確 |

## UV Single-File Scripts 架構

本專案所有 hooks 使用 [Astral UV](https://docs.astral.sh/uv/) 的 single-file scripts 模式。每個 Python 腳本在檔案開頭宣告自己的相依套件：

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.11"
# dependencies = [
#     "python-dotenv",
# ]
# ///
```

**為什麼用這種模式：**

| 優勢 | 說明 |
|------|------|
| **隔離** | Hook 的相依套件不會污染主專案的 dependency tree |
| **可攜性** | 每個腳本自包含相依宣告，可獨立複製到其他專案 |
| **無需虛擬環境** | UV 自動管理相依安裝與快取 |
| **快速啟動** | UV 的相依解析極快，hook 啟動延遲幾乎不可察覺 |

## 專案結構

```
.claude/
├── settings.json              ← Hook 設定與權限
├── hooks/                     ← Hook 腳本
│   ├── user_prompt_submit.py  ← Prompt 驗證、日誌、上下文注入
│   ├── pre_tool_use.py        ← 安全阻擋（rm -rf、.env 存取）
│   ├── post_tool_use.py       ← 工具結果日誌
│   ├── stop.py                ← 完成通知（TTS、AI 生成訊息）
│   ├── session_start.py       ← 開發環境上下文載入
│   ├── session_end.py         ← Session 清理
│   ├── setup.py               ← 環境初始化與維護
│   ├── pre_compact.py         ← 壓縮前備份
│   ├── notification.py        ← 通知 TTS
│   ├── permission_request.py  ← 權限稽核
│   ├── subagent_start.py      ← Sub-agent 啟動日誌
│   ├── subagent_stop.py       ← Sub-agent 完成通知
│   ├── post_tool_use_failure.py ← 錯誤日誌
│   ├── validators/            ← 程式碼品質驗證
│   │   ├── ruff_validator.py  ← Python linting
│   │   └── ty_validator.py    ← Python 型別檢查
│   └── utils/                 ← TTS 與 LLM 工具
├── agents/                    ← Sub-agent 設定
├── commands/                  ← 自訂 slash commands
├── output-styles/             ← 輸出格式定義
└── status_lines/              ← 狀態列腳本
```

## 技術先決條件

- [Astral UV](https://docs.astral.sh/uv/getting-started/installation/) — Python 套件管理與執行
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — Anthropic 的 CLI 工具

可選：ElevenLabs / OpenAI（TTS）、Ollama（本地 LLM）、Firecrawl MCP（Web scraping）

## 本教學涵蓋的內容

1. **介紹**（本篇）— 架構概覽與 UV scripts 模式
2. **Hook 生命週期** — 13 個 hook 事件的觸發時機與 payload
3. **Exit Codes 與流程控制** — 阻擋、通過、JSON 輸出控制
4. **安全性 Hooks** — PreToolUse 阻擋、UserPromptSubmit 驗證
5. **Session 與 Setup Hooks** — 環境初始化、上下文注入、session 管理
6. **Sub-Agents** — 子代理架構、Meta-Agent、agent chaining
7. **團隊驗證系統** — Builder/Validator 模式、Task 系統協調
8. **Status Lines 與 Output Styles** — 終端機狀態列、輸出格式自訂

## 參考來源

- [claude-code-hooks-mastery (GitHub)](https://github.com/disler/claude-code-hooks-mastery)
- [Claude Code Hooks 官方文件](https://docs.anthropic.com/en/docs/claude-code/hooks)
- [Claude Code Sub-Agents 文件](https://docs.anthropic.com/en/docs/claude-code/sub-agents)

## 重點整理

- Hooks 提供 Claude Code 行為的確定性控制——安全阻擋、日誌、上下文注入、流程控制
- 所有設定在 `.claude/settings.json`，每個事件可綁定一或多個 command
- UV single-file scripts 讓每個 hook 自包含相依宣告，不影響主專案
- `$CLAUDE_PROJECT_DIR` 環境變數確保 hook 路徑在任何工作目錄下都正確解析

---

下一篇：[Hook 生命週期](./02-hook-lifecycle.md)
