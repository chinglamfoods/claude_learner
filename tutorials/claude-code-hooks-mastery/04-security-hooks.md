# 04：安全性 Hooks

> 使用 PreToolUse 和 UserPromptSubmit 建立多層安全防線——阻擋危險指令、保護敏感檔案、驗證 prompt 內容。

## 安全性分層

```
第一層：UserPromptSubmit — 在 prompt 到達 Claude 之前攔截
第二層：PreToolUse       — 在工具執行之前阻擋
第三層：PostToolUse      — 工具執行後驗證結果
第四層：PermissionRequest — 權限稽核與自動核准
```

## PreToolUse：阻擋危險操作

### 完整的 rm -rf 偵測

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.8"
# ///

import json
import sys
import re
from pathlib import Path

def is_dangerous_rm_command(command):
    """偵測各種形式的危險 rm 指令"""
    normalized = ' '.join(command.lower().split())

    # rm -rf 的各種變體
    patterns = [
        r'\brm\s+.*-[a-z]*r[a-z]*f',      # rm -rf, rm -Rf, rm -rfi
        r'\brm\s+.*-[a-z]*f[a-z]*r',      # rm -fr
        r'\brm\s+--recursive\s+--force',   # rm --recursive --force
        r'\brm\s+--force\s+--recursive',   # rm --force --recursive
        r'\brm\s+-r\s+.*-f',              # rm -r ... -f（分開寫）
        r'\brm\s+-f\s+.*-r',              # rm -f ... -r
    ]

    for pattern in patterns:
        if re.search(pattern, normalized):
            return True

    # 有 recursive flag 時，檢查是否指向危險路徑
    dangerous_paths = [
        r'/',         # 根目錄
        r'/\*',       # 根目錄 wildcard
        r'~',         # 家目錄
        r'\$HOME',    # 家目錄環境變數
        r'\.\.',      # 上層目錄
        r'\.',        # 當前目錄
    ]

    if re.search(r'\brm\s+.*-[a-z]*r', normalized):
        for path in dangerous_paths:
            if re.search(path, normalized):
                return True

    return False
```

### .env 檔案存取保護

```python
def is_env_file_access(tool_name, tool_input):
    """阻擋任何對 .env 檔案的存取（.env.sample 除外）"""
    if tool_name in ['Read', 'Edit', 'MultiEdit', 'Write']:
        file_path = tool_input.get('file_path', '')
        if '.env' in file_path and not file_path.endswith('.env.sample'):
            return True

    elif tool_name == 'Bash':
        command = tool_input.get('command', '')
        env_patterns = [
            r'\b\.env\b(?!\.sample)',           # .env 但不是 .env.sample
            r'cat\s+.*\.env\b(?!\.sample)',     # cat .env
            r'echo\s+.*>\s*\.env\b(?!\.sample)', # echo > .env
        ]
        for pattern in env_patterns:
            if re.search(pattern, command):
                return True

    return False
```

### 完整的 PreToolUse Hook

```python
def main():
    try:
        input_data = json.load(sys.stdin)
        tool_name = input_data.get('tool_name', '')
        tool_input = input_data.get('tool_input', {})

        # 安全檢查 1：.env 檔案存取
        if is_env_file_access(tool_name, tool_input):
            print("BLOCKED: 禁止存取包含敏感資料的 .env 檔案", file=sys.stderr)
            print("請改用 .env.sample 作為模板", file=sys.stderr)
            sys.exit(2)

        # 安全檢查 2：危險的 rm 指令
        if tool_name == 'Bash':
            command = tool_input.get('command', '')
            if is_dangerous_rm_command(command):
                print("BLOCKED: 偵測到危險的 rm 指令", file=sys.stderr)
                sys.exit(2)

        # 日誌記錄（所有事件）
        log_event('pre_tool_use', input_data)

        sys.exit(0)

    except json.JSONDecodeError:
        sys.exit(0)
    except Exception:
        sys.exit(0)
```

## UserPromptSubmit：Prompt 層級控制

UserPromptSubmit 是第一道防線，在 Claude 看到 prompt 之前執行。

### 三種操作模式

```python
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--validate', action='store_true',
                        help='啟用 prompt 安全驗證')
    parser.add_argument('--log-only', action='store_true',
                        help='僅記錄，不驗證或阻擋')
    parser.add_argument('--store-last-prompt', action='store_true',
                        help='儲存最後一個 prompt（供 status line 使用）')
    parser.add_argument('--name-agent', action='store_true',
                        help='為 session 產生代理名稱')
    args = parser.parse_args()

    input_data = json.loads(sys.stdin.read())
    session_id = input_data.get('session_id', 'unknown')
    prompt = input_data.get('prompt', '')

    # 1. 永遠記錄
    log_user_prompt(session_id, input_data)

    # 2. 儲存 session 資料（供 status line 使用）
    if args.store_last_prompt or args.name_agent:
        manage_session_data(session_id, prompt, name_agent=args.name_agent)

    # 3. 驗證（如果啟用）
    if args.validate and not args.log_only:
        is_valid, reason = validate_prompt(prompt)
        if not is_valid:
            print(f"Prompt blocked: {reason}", file=sys.stderr)
            sys.exit(2)

    sys.exit(0)
```

### 上下文注入

UserPromptSubmit 的 stdout 輸出會被附加在 prompt 之前，讓 Claude 看到額外的上下文：

```python
# 如果啟用上下文注入
if args.context:
    context = f"""專案：E-commerce API
標準：遵循 REST 慣例與 OpenAPI 3.0
時間：{datetime.now().isoformat()}
Git branch：{get_git_branch()}"""
    print(context)  # stdout 內容會附加在 prompt 前面
```

Claude 會看到：

```
專案：E-commerce API
標準：遵循 REST 慣例與 OpenAPI 3.0
時間：2026-03-02T15:30:00
Git branch：feat/new-endpoint

Write a new API endpoint     ← 使用者的原始 prompt
```

### settings.json 設定

```json
"UserPromptSubmit": [
  {
    "hooks": [
      {
        "type": "command",
        "command": "uv run $CLAUDE_PROJECT_DIR/.claude/hooks/user_prompt_submit.py --log-only --store-last-prompt --name-agent"
      }
    ]
  }
]
```

Flag 組合決定行為：
- `--log-only` — 僅記錄（最安全的起點）
- `--validate` — 啟用安全驗證
- `--store-last-prompt` — 儲存 prompt 供 status line 顯示
- `--name-agent` — 為 session 產生 LLM 命名

## PermissionRequest：自動核准唯讀操作

```python
def main():
    input_data = json.load(sys.stdin)
    tool_name = input_data.get('tool_name', '')
    tool_input = input_data.get('tool_input', {})

    # 記錄所有權限請求
    log_event('permission_request', input_data)

    # 唯讀工具自動核准
    read_only_tools = {'Read', 'Glob', 'Grep'}
    if tool_name in read_only_tools:
        output = {
            "decision": "approve",
            "reason": "唯讀操作自動核准"
        }
        print(json.dumps(output))
        sys.exit(0)

    # 安全的 Bash 指令自動核准
    if tool_name == 'Bash':
        command = tool_input.get('command', '')
        safe_prefixes = ['ls', 'cat', 'head', 'tail', 'wc', 'find', 'grep']
        if any(command.strip().startswith(p) for p in safe_prefixes):
            output = {
                "decision": "approve",
                "reason": "安全的 Bash 指令自動核准"
            }
            print(json.dumps(output))

    sys.exit(0)
```

## 可擴充的安全規則

以下是常見的安全規則模式，可依專案需求加入：

```python
# 擴充的危險指令模式
dangerous_patterns = [
    r'rm\s+.*-[rf]',            # rm -rf 變體
    r'sudo\s+rm',               # sudo rm
    r'chmod\s+777',             # 過於寬鬆的權限
    r'>\s*/etc/',               # 寫入系統目錄
    r'curl.*\|\s*sh',           # 下載並執行
    r'eval\s*\(',               # 動態執行
    r'DROP\s+TABLE',            # SQL 資料刪除
    r'TRUNCATE\s+TABLE',        # SQL 資料清除
]

# 敏感檔案路徑
sensitive_paths = [
    '.env',
    'credentials.json',
    'secrets.yaml',
    '*.pem',
    '*.key',
    'id_rsa',
]
```

## 重點整理

- 安全性分四層：UserPromptSubmit → PreToolUse → PostToolUse → PermissionRequest
- PreToolUse 是阻擋危險操作的主要控制點——exit code 2 阻止工具執行
- UserPromptSubmit 可以阻擋 prompt、注入上下文、以及記錄稽核日誌
- `.env` 檔案保護需要同時攔截 Read/Write/Edit 和 Bash 中的存取
- PermissionRequest 可以自動核准安全操作，減少使用者的確認負擔
- 所有 hook 的錯誤處理都應靜默通過（exit 0），避免 hook 故障影響正常工作

---

下一篇：[Session 與 Setup Hooks](./05-session-and-setup-hooks.md)
