# 03：Exit Codes 與流程控制

> Hook 透過 exit codes 與 JSON 輸出控制 Claude Code 的行為——阻擋操作、提供回饋、強制繼續。

## Exit Code 行為

| Exit Code | 行為 | 說明 |
|-----------|------|------|
| **0** | 成功 | Hook 正常執行完成。stdout 輸出在 transcript mode（Ctrl-R）中對使用者可見 |
| **2** | 阻擋錯誤 | stderr 內容**自動回饋給 Claude**。具體行為依 hook 類型而定 |
| **其他** | 非阻擋錯誤 | stderr 對使用者可見，但不影響執行流程 |

Exit code 2 是最重要的控制機制。它在不同 hook 中的效果：

| Hook 類型 | Exit Code 2 的效果 |
|-----------|-------------------|
| UserPromptSubmit | **阻擋 prompt**——Claude 不會看到這個 prompt |
| PreToolUse | **阻擋工具執行**——Claude 收到錯誤訊息 |
| Stop | **阻止停止**——Claude 被迫繼續工作 |
| SubagentStop | **阻止 sub-agent 停止**——sub-agent 繼續工作 |
| PostToolUse | 無法阻止（工具已執行），但 stderr 回饋給 Claude |
| Notification / PreCompact / SessionStart / SessionEnd | 無阻擋能力，stderr 僅對使用者可見 |

## 基本阻擋範例

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.8"
# ///

import json
import sys
import re

def main():
    input_data = json.load(sys.stdin)
    tool_name = input_data.get('tool_name', '')
    tool_input = input_data.get('tool_input', {})

    if tool_name == 'Bash':
        command = tool_input.get('command', '')
        # 偵測 rm -rf 變體
        if re.search(r'\brm\s+.*-[a-z]*r[a-z]*f', command.lower()):
            # stderr 的內容會自動送給 Claude
            print("BLOCKED: 偵測到危險的 rm 指令", file=sys.stderr)
            sys.exit(2)  # exit code 2 阻擋工具執行

    # exit code 0 允許正常執行
    sys.exit(0)

if __name__ == '__main__':
    main()
```

## JSON 輸出控制

除了 exit codes，hooks 可以透過 stdout 輸出結構化 JSON 進行更精細的控制。

### 通用 JSON 欄位（所有 hook 類型）

```json
{
  "continue": true,           // false 時 Claude 完全停止（優先級最高）
  "stopReason": "原因說明",    // continue=false 時對使用者顯示的訊息
  "suppressOutput": true      // 隱藏 stdout 不在 transcript 中顯示
}
```

### PreToolUse 的 decision 控制

```python
import json
import sys

input_data = json.load(sys.stdin)
tool_name = input_data.get('tool_name', '')
tool_input = input_data.get('tool_input', {})

# 方法 1：自動核准（跳過權限確認）
if tool_name in ('Read', 'Glob', 'Grep'):
    output = {
        "decision": "approve",
        "reason": "唯讀操作自動核准"
    }
    print(json.dumps(output))
    sys.exit(0)

# 方法 2：阻擋並提供原因
if tool_name == 'Bash' and '.env' in tool_input.get('command', ''):
    output = {
        "decision": "block",
        "reason": "禁止透過 Bash 存取 .env 檔案，請使用 .env.sample"
    }
    print(json.dumps(output))
    sys.exit(0)  # 注意：用 JSON decision 控制時，exit code 為 0

# 方法 3：不設定 decision（走正常權限流程）
sys.exit(0)
```

| decision 值 | 效果 |
|-------------|------|
| `"approve"` | 跳過權限系統，`reason` 對使用者可見 |
| `"block"` | 阻擋工具執行，`reason` 回饋給 Claude |
| 未設定 | 走正常的權限確認流程 |

### PostToolUse 的 decision 控制

```python
# PostToolUse 中的 block 會自動用 reason 提示 Claude 修正
if tool_name == "Write" and not tool_response.get("success"):
    output = {
        "decision": "block",
        "reason": "檔案寫入失敗，請確認權限後重試"
    }
    print(json.dumps(output))
    sys.exit(0)
```

PostToolUse 的 `"block"` 不能撤銷已執行的操作，但會將 `reason` 傳給 Claude 作為自動提示，讓它知道需要修正。

### Stop 的 decision 控制

```python
# 阻止 Claude 停止回應——強制它繼續工作
input_data = json.load(sys.stdin)

# stop_hook_active 防止無限迴圈：
# 如果是 true，代表這是因為 hook 阻擋停止後的重試，不應再次阻擋
if not input_data.get('stop_hook_active', False):
    if not all_tests_passed():
        output = {
            "decision": "block",
            "reason": "測試尚未全部通過，請修復失敗的測試後再停止"
        }
        print(json.dumps(output))
        sys.exit(0)
```

> **警告：** Stop hook 的 `"block"` 如果不正確處理 `stop_hook_active` flag，**會造成無限迴圈**——Claude 嘗試停止 → hook 阻擋 → Claude 繼續 → 嘗試停止 → hook 再阻擋 → 永不結束。

## 流程控制優先順序

多個控制機制同時使用時的優先順序：

1. **`"continue": false`** — 最高優先，Claude 完全停止
2. **`"decision": "block"`** — Hook 特定的阻擋行為
3. **Exit Code 2** — 簡單的 stderr 阻擋
4. **其他 Exit Codes** — 非阻擋錯誤

## 實務模式

### 優雅的錯誤處理

Hook 自身的錯誤不應中斷 Claude 的工作流程：

```python
def main():
    try:
        input_data = json.load(sys.stdin)
        # ... hook 邏輯 ...
        sys.exit(0)
    except json.JSONDecodeError:
        # JSON 解析失敗——不阻擋，靜默通過
        sys.exit(0)
    except Exception:
        # 任何未預期的錯誤——不阻擋，靜默通過
        sys.exit(0)
```

**原則：** Hook 的錯誤不應該變成使用者的問題。除非你明確要阻擋操作（exit 2），否則總是 exit 0。

### 記錄所有事件

```python
def log_event(event_name, input_data):
    log_dir = Path("logs")
    log_dir.mkdir(parents=True, exist_ok=True)
    log_file = log_dir / f'{event_name}.json'

    # 讀取既有日誌或初始化
    if log_file.exists():
        with open(log_file, 'r') as f:
            try:
                log_data = json.load(f)
            except (json.JSONDecodeError, ValueError):
                log_data = []
    else:
        log_data = []

    log_data.append(input_data)

    with open(log_file, 'w') as f:
        json.dump(log_data, f, indent=2)
```

## 重點整理

- Exit code 0 = 通過、2 = 阻擋（stderr 回饋給 Claude）、其他 = 非阻擋錯誤
- JSON `decision` 欄位提供更精細的控制：`"approve"` 跳過權限、`"block"` 阻擋並回饋原因
- `"continue": false` 是最高優先的控制——完全停止 Claude
- Stop hook 的 `stop_hook_active` flag **必須**檢查，否則會造成無限迴圈
- Hook 自身的錯誤應該靜默通過（exit 0），不應阻擋使用者的工作流程

---

下一篇：[安全性 Hooks](./04-security-hooks.md)
