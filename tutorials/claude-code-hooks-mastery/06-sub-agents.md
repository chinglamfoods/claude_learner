# 06：Sub-Agents

> Claude Code 的子代理架構——系統提示 vs. 使用者提示的區別、agent 檔案格式、Meta-Agent、以及 agent chaining。

## Sub-Agent 運作模式

Sub-agent 是具有獨立系統提示、工具權限、以及上下文視窗的專門化 AI 助手。關鍵概念：

```
你（使用者）→ 主 Agent → Sub-Agent → 主 Agent → 你（使用者）
```

**最常見的誤解：** `.claude/agents/*.md` 中的內容是**系統提示**，不是使用者提示。Sub-agent 永遠不會直接跟你溝通。

| 事實 | 說明 |
|------|------|
| Sub-agent 不與使用者溝通 | 它們只回報給主 agent |
| Sub-agent 從零開始 | 沒有先前的對話歷史 |
| Sub-agent 回應主 agent 的 prompt | 不是你的原始 prompt |
| `description` 決定何時使用 | 主 agent 依據 description 決定是否委派 |

## Agent 檔案格式

```yaml
---
name: code-reviewer
description: 在使用者要求程式碼審查時主動使用。專門審查 TypeScript 與 Python 程式碼的品質與安全性。
tools: Read, Grep, Glob           # 可選——省略時繼承所有工具
color: cyan                       # 終端機視覺標識
model: sonnet                     # haiku | sonnet | opus，預設 sonnet
---

# Purpose

你是一個程式碼審查專家，專注於 TypeScript 與 Python。

## Instructions

1. 接收到程式碼審查請求後，使用 Glob 找到相關檔案
2. 使用 Read 逐一閱讀每個檔案
3. 檢查：型別安全、錯誤處理、安全漏洞、效能問題
4. 依嚴重程度分類發現的問題

## Report

以下列格式回報：

### 審查結果
- 🔴 嚴重：[問題描述]
- 🟡 警告：[問題描述]
- 🟢 建議：[改善建議]
```

### 儲存位置

| 位置 | 優先級 | 適用範圍 |
|------|--------|---------|
| `.claude/agents/` | 較高 | 專案特定 |
| `~/.claude/agents/` | 較低 | 跨專案通用 |

### description 的重要性

`description` 欄位決定主 agent **何時**自動委派任務給 sub-agent：

```yaml
# ❌ 不好的 description — 太模糊
description: Handles various tasks

# ✅ 好的 description — 明確觸發條件
description: 在使用者要求程式碼審查時主動使用。專門審查 TypeScript 與 Python。

# ✅ 含有觸發關鍵字
description: Use PROACTIVELY when user mentions TTS, audio, or voice feedback.
```

使用 `"Use PROACTIVELY"` 或特定觸發詞讓主 agent 更積極地自動委派。

### 工具限制

```yaml
# 完整工具存取
tools: Read, Write, Edit, Bash, Grep, Glob

# 唯讀存取（驗證用 agent）
tools: Read, Grep, Glob

# 禁止特定工具（替代 tools 列舉）
disallowedTools: Write, Edit, NotebookEdit
```

省略 `tools` 欄位時 sub-agent 繼承主 agent 的所有工具。使用 `disallowedTools` 可以用排除法限制。

## Agent Chaining

主 agent 可以串接多個 sub-agent 完成複雜工作流程：

```
使用者："分析市場趨勢然後找投資機會"
         │
         ├→ crypto-market-agent（分析）→ 回報市場狀態
         │
         └→ crypto-investment-plays（策略）→ 基於市場分析找機會
                                             │
                                             → 主 agent 綜合回報給使用者
```

實際指令可以是：
- "先用 debugger agent 修復錯誤，然後讓 code-reviewer 檢查修改"
- "用 meta-agent 建立一個新的 agent，然後測試它"

## Meta-Agent：建立 Agent 的 Agent

Meta-Agent（`.claude/agents/meta-agent.md`）是專門**產生新 sub-agent 設定檔**的 agent：

```yaml
---
name: meta-agent
description: 從描述產生新的 Claude Code sub-agent 設定檔。在使用者要求建立新 sub-agent 時主動使用。
tools: Write, WebFetch, mcp__firecrawl-mcp__firecrawl_scrape, MultiEdit
color: cyan
model: opus
---
```

### Meta-Agent 的工作流程

1. **抓取最新文件** — 從 Anthropic 官方文件取得最新的 sub-agent API
2. **分析需求** — 理解使用者描述的 agent 用途
3. **決定名稱** — 產生 kebab-case 名稱
4. **選擇最少工具** — 根據任務需求推導最小工具集
5. **撰寫系統提示** — 包含明確的步驟指示與回報格式
6. **寫入檔案** — 建立 `.claude/agents/<name>.md`

### 使用方式

```
"建立一個新的 sub-agent，專門執行測試並修復失敗的測試案例"
```

Claude 會自動委派給 meta-agent，產生完整的 agent 設定檔。

### Meta-Agent 的價值

- **快速擴展** — 幾分鐘內建立大量專門化 agent
- **結構一致** — 確保所有 agent 遵循最佳實務與格式規範
- **自動更新** — 每次執行都抓取最新的官方文件
- **複合效益** — 「建造建造工具的工具」——加速工程產出

## 建立有效 Sub-Agent 的原則

**1. Problem → Solution → Technology**

先釐清 agent 要解決什麼問題，再設計解決方案，最後選擇需要的工具。

**2. 系統提示的撰寫重點**

- 明確定義角色（"你是一個..."）
- 提供編號的步驟指示（agent 從零開始，需要明確指引）
- 定義回報格式（sub-agent 回報給主 agent，格式需要結構化）
- 包含該領域的最佳實務

**3. 常見的 Agent 類型**

| 類型 | 用途 | 建議工具 |
|------|------|---------|
| 審查者 | 程式碼審查、文件檢查 | Read, Grep, Glob |
| 建構者 | 實作功能、修改程式碼 | Read, Write, Edit, Bash |
| 研究者 | 資料收集、分析 | Read, WebFetch, Grep |
| 除錯者 | 錯誤排查、修復 | Read, Edit, Bash, Grep |
| 測試者 | 執行測試、驗證 | Read, Bash, Grep |

## 重點整理

- Sub-agent 的 markdown 內容是**系統提示**，不是使用者提示
- `description` 欄位是主 agent 決定何時委派的關鍵——寫得越明確，自動委派越精準
- Sub-agent 從零開始、無對話歷史、只與主 agent 通訊
- 工具權限可以透過 `tools`（允許列表）或 `disallowedTools`（排除列表）控制
- Meta-Agent 是「建造 agent 的 agent」，從描述自動產生完整的 agent 設定檔
- Agent chaining 讓主 agent 可以串接多個 sub-agent 完成複雜工作流程

---

下一篇：[團隊驗證系統](./07-team-validation.md)
