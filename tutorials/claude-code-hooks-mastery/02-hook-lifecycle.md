# 02：Hook 生命週期

> Claude Code 的 13 個 hook 事件——各自的觸發時機、接收的 JSON payload、以及能做什麼。

## 生命週期總覽

Hooks 分為三個階段：Session 生命週期、主對話迴圈、維護作業。

```
Session 生命週期：  Setup → SessionStart → ... → SessionEnd

主對話迴圈：
  UserPromptSubmit → Claude 處理 → PreToolUse → 工具執行 → PostToolUse
                                  ↘ SubagentStart → Subagent 工作 → SubagentStop
                                  → Stop → 等待下一個 prompt

維護：  PreCompact（壓縮前備份）
通知：  Notification（非同步，Claude 等待輸入時）
權限：  PermissionRequest（使用者看到權限對話框時）
錯誤：  PostToolUseFailure（工具執行失敗時）
```

## 全部 13 個 Hook 事件

### 1. Setup

**觸發時機：** Claude 進入一個 repository（`init`）或定期維護（`maintenance`）

```json
{
  "session_id": "...",
  "cwd": "/home/user/project",
  "trigger": "init",           // "init" 或 "maintenance"
  "hook_event_name": "Setup"
}
```

**可回傳 `additionalContext`** 注入到 Claude 的系統提示中：

```python
output = {
    "hookSpecificOutput": {
        "hookEventName": "Setup",
        "additionalContext": "Git branch: main\nDetected: Node.js project"
    }
}
print(json.dumps(output))
```

**可透過 `CLAUDE_ENV_FILE` 持久化環境變數：**

```python
env_file = os.environ.get('CLAUDE_ENV_FILE')
if env_file:
    with open(env_file, 'a') as f:
        f.write(f'export PROJECT_ROOT="{cwd}"\n')
```

### 2. SessionStart

**觸發時機：** 新 session 啟動或恢復既有 session

```json
{
  "session_id": "...",
  "source": "startup",    // "startup"、"resume"、"clear"
  "cwd": "/home/user/project"
}
```

適合載入開發環境上下文（git status、近期 issue、context 檔案）。

### 3. SessionEnd

**觸發時機：** Session 結束（正常退出、Ctrl+C、或錯誤）

```json
{
  "session_id": "...",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/home/user/project",
  "permission_mode": "default",
  "reason": "exit"         // "exit"、"sigint"、"error"
}
```

適合 session 日誌記錄、暫存檔案清理。

### 4. UserPromptSubmit

**觸發時機：** 使用者送出 prompt 後、Claude 開始處理前

```json
{
  "session_id": "...",
  "prompt": "Write a new API endpoint"
}
```

**這是最重要的控制點之一。** 可以：
- 記錄所有 prompt（稽核日誌）
- 阻擋 prompt（exit code 2）
- 注入上下文（stdout 輸出會附加在 prompt 之前讓 Claude 看到）
- 驗證內容（安全檢查、政策合規）

### 5. PreToolUse

**觸發時機：** 工具執行之前

```json
{
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test"
  },
  "session_id": "..."
}
```

**可阻擋工具執行。** Exit code 2 或 JSON `"decision": "block"` 阻止工具執行並將錯誤訊息回饋給 Claude。

### 6. PostToolUse

**觸發時機：** 工具成功執行之後

```json
{
  "tool_name": "Write",
  "tool_input": { "file_path": "src/app.ts" },
  "tool_response": { "success": true },
  "session_id": "..."
}
```

**無法阻止工具執行**（已經完成了），但可以用 `"decision": "block"` 回饋問題給 Claude，讓它修正。

### 7. PostToolUseFailure

**觸發時機：** 工具執行失敗

```json
{
  "tool_name": "Bash",
  "tool_input": { "command": "..." },
  "tool_use_id": "...",
  "error": { "message": "Command failed with exit code 1" }
}
```

適合結構化錯誤日誌記錄。

### 8. PermissionRequest

**觸發時機：** 使用者看到權限確認對話框

```json
{
  "tool_name": "Write",
  "tool_input": { "file_path": "..." },
  "tool_use_id": "...",
  "session_id": "..."
}
```

可用於權限稽核，或自動允許安全操作（如 Read、Glob、Grep）。

### 9. Notification

**觸發時機：** Claude Code 發送通知（例如等待使用者輸入）

```json
{
  "message": "Claude needs your input"
}
```

適合觸發 TTS 語音提示或桌面通知。

### 10. Stop

**觸發時機：** Claude 完成回應時

```json
{
  "session_id": "...",
  "stop_hook_active": false,
  "transcript_path": "/path/to/transcript.jsonl"
}
```

**可以阻止 Claude 停止。** 用 `"decision": "block"` 強制 Claude 繼續工作。`stop_hook_active` flag 防止無限迴圈。

### 11. SubagentStart

**觸發時機：** Sub-agent 被啟動

```json
{
  "agent_id": "...",
  "agent_type": "...",
  "session_id": "..."
}
```

### 12. SubagentStop

**觸發時機：** Sub-agent 完成工作

```json
{
  "stop_hook_active": false,
  "session_id": "..."
}
```

**可以阻止 sub-agent 停止，** 行為類似 Stop hook。

### 13. PreCompact

**觸發時機：** Claude 執行對話壓縮之前（手動或自動）

```json
{
  "trigger": "manual",      // "manual" 或 "auto"
  "custom_instructions": "保留所有程式碼範例",  // 僅 manual 時有值
  "session_id": "..."
}
```

適合在壓縮前備份完整的對話記錄。

## Hook 執行環境

| 項目 | 說明 |
|------|------|
| 超時 | 每個 hook 最多執行 60 秒 |
| 並行 | 同一事件的多個 hook 同時執行 |
| 環境變數 | 繼承 Claude Code 的環境變數 |
| 工作目錄 | 當前專案目錄 |
| 輸入 | stdin 接收 JSON（包含 session 資訊與事件資料） |
| 輸出 | stdout / stderr + exit code |

## 重點整理

- Claude Code 有 13 個 hook 事件，涵蓋 session 生命週期、工具執行、sub-agent、壓縮與通知
- 每個 hook 透過 stdin 接收 JSON payload，包含該事件的完整上下文
- 部分 hooks 可以阻擋操作（UserPromptSubmit、PreToolUse、Stop），部分僅供觀察
- Setup hook 可透過 `additionalContext` 注入系統提示，透過 `CLAUDE_ENV_FILE` 持久化環境變數
- 所有 hooks 有 60 秒超時限制，同一事件的多個 hook 平行執行

---

下一篇：[Exit Codes 與流程控制](./03-exit-codes-and-flow-control.md)
