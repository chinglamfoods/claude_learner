# 05：Session 與 Setup Hooks

> 利用 Setup、SessionStart、SessionEnd、PreCompact、Stop hooks 管理 session 生命週期——環境初始化、上下文載入、完成通知、壓縮備份。

## Setup Hook：環境初始化與維護

Setup hook 在 Claude 進入 repository 時（`init`）或定期維護時（`maintenance`）觸發。它有兩個獨特能力：

1. **`additionalContext`** — 注入文字到 Claude 的系統提示中
2. **`CLAUDE_ENV_FILE`** — 持久化環境變數跨 hook 共享

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.11"
# dependencies = ["python-dotenv"]
# ///

import json
import os
import sys
import subprocess
from pathlib import Path

def get_project_info(cwd):
    """收集專案資訊作為上下文"""
    info = []

    # Git branch
    try:
        result = subprocess.run(
            ['git', 'rev-parse', '--abbrev-ref', 'HEAD'],
            capture_output=True, text=True, timeout=5, cwd=cwd
        )
        if result.returncode == 0:
            info.append(f"Git branch: {result.stdout.strip()}")
    except Exception:
        pass

    # 偵測專案類型
    project_files = [
        ('package.json', 'Node.js project'),
        ('pyproject.toml', 'Python project'),
        ('Cargo.toml', 'Rust project'),
        ('go.mod', 'Go project'),
    ]
    for filename, description in project_files:
        if Path(cwd, filename).exists():
            info.append(f"Detected: {description}")

    return info

def persist_env_variable(name, value):
    """透過 CLAUDE_ENV_FILE 持久化環境變數"""
    env_file = os.environ.get('CLAUDE_ENV_FILE')
    if env_file:
        with open(env_file, 'a') as f:
            f.write(f'export {name}="{value}"\n')
        return True
    return False

def main():
    input_data = json.loads(sys.stdin.read())
    cwd = input_data.get('cwd', os.getcwd())
    trigger = input_data.get('trigger', 'init')

    context_parts = [f"Setup triggered: {trigger}"]

    if trigger == 'init':
        # 持久化專案根目錄
        persist_env_variable('PROJECT_ROOT', cwd)

        # 收集專案資訊
        project_info = get_project_info(cwd)
        if project_info:
            context_parts.append("\n--- Project Information ---")
            context_parts.extend(project_info)

    elif trigger == 'maintenance':
        # 定期維護：檢查日誌大小等
        logs_dir = Path(cwd, 'logs')
        if logs_dir.exists():
            total_size = sum(f.stat().st_size for f in logs_dir.rglob('*') if f.is_file())
            size_mb = total_size / (1024 * 1024)
            if size_mb > 10:
                context_parts.append(f"Warning: logs 目錄已達 {size_mb:.2f}MB")

    # 透過 additionalContext 注入系統提示
    output = {
        "hookSpecificOutput": {
            "hookEventName": "Setup",
            "additionalContext": "\n".join(context_parts)
        }
    }
    print(json.dumps(output))
    sys.exit(0)
```

## SessionStart Hook：開發上下文載入

SessionStart 在新 session 啟動或恢復時觸發。適合載入讓 Claude 瞭解當前開發狀態的資訊：

```python
def main():
    input_data = json.loads(sys.stdin.read())
    source = input_data.get('source', 'startup')
    cwd = input_data.get('cwd', os.getcwd())

    # 記錄 session 事件
    log_event('session_start', input_data)

    if source == 'startup':
        # 新 session：載入完整上下文
        load_git_context(cwd)
        load_recent_issues(cwd)
    elif source == 'resume':
        # 恢復 session：僅更新變更
        load_git_diff_since_last(cwd)
    elif source == 'clear':
        # 清除後重新開始
        pass

    sys.exit(0)

def load_git_context(cwd):
    """載入 git 狀態作為上下文"""
    try:
        # 最近的 commit 摘要
        result = subprocess.run(
            ['git', 'log', '--oneline', '-5'],
            capture_output=True, text=True, timeout=5, cwd=cwd
        )
        if result.returncode == 0:
            print(f"Recent commits:\n{result.stdout}")

        # 目前的變更
        result = subprocess.run(
            ['git', 'status', '--short'],
            capture_output=True, text=True, timeout=5, cwd=cwd
        )
        if result.returncode == 0 and result.stdout.strip():
            print(f"Uncommitted changes:\n{result.stdout}")
    except Exception:
        pass
```

## SessionEnd Hook：清理與記錄

```python
def main():
    input_data = json.loads(sys.stdin.read())
    reason = input_data.get('reason', 'exit')
    session_id = input_data.get('session_id', '')
    transcript_path = input_data.get('transcript_path', '')

    # 記錄 session 結束事件
    log_event('session_end', input_data)

    # 可選的清理任務
    if reason in ('exit', 'sigint'):
        cleanup_temp_files()
        cleanup_stale_logs()

    sys.exit(0)

def cleanup_temp_files():
    """移除 session 產生的暫存檔案"""
    temp_patterns = ['*.tmp', '*.bak', '.~*']
    for pattern in temp_patterns:
        for f in Path('.').glob(pattern):
            try:
                f.unlink()
            except Exception:
                pass

def cleanup_stale_logs():
    """清理超過 7 天的日誌檔案"""
    import time
    logs_dir = Path('logs')
    if not logs_dir.exists():
        return
    cutoff = time.time() - 7 * 86400
    for f in logs_dir.iterdir():
        if f.is_file() and f.stat().st_mtime < cutoff:
            try:
                f.unlink()
            except Exception:
                pass
```

## PreCompact Hook：壓縮前備份

Claude 在對話過長時會進行壓縮（compaction），丟棄部分對話歷史。PreCompact 讓你在壓縮前備份完整的對話記錄：

```python
def main():
    input_data = json.loads(sys.stdin.read())
    trigger = input_data.get('trigger', 'auto')
    session_id = input_data.get('session_id', 'unknown')

    log_event('pre_compact', input_data)

    # 備份當前對話記錄
    transcript_path = input_data.get('transcript_path', '')
    if transcript_path and Path(transcript_path).exists():
        backup_dir = Path('logs/compaction_backups')
        backup_dir.mkdir(parents=True, exist_ok=True)

        from datetime import datetime
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        backup_path = backup_dir / f'transcript_{session_id[:8]}_{timestamp}.jsonl'

        import shutil
        shutil.copy2(transcript_path, backup_path)

    if trigger == 'manual':
        # 手動壓縮可能有自訂指示
        custom_instructions = input_data.get('custom_instructions', '')
        if custom_instructions:
            print(f"Manual compaction with instructions: {custom_instructions}",
                  file=sys.stderr)

    sys.exit(0)
```

## Stop Hook：完成通知與 transcript 擷取

Stop hook 在 Claude 完成回應時觸發。可以用於：
- 透過 TTS 語音宣告完成
- 將對話記錄轉換為可讀格式
- AI 生成的完成摘要

```python
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--chat', action='store_true',
                        help='將 transcript 複製為可讀 JSON')
    parser.add_argument('--notify', action='store_true',
                        help='啟用 TTS 完成通知')
    args = parser.parse_args()

    input_data = json.load(sys.stdin)
    stop_hook_active = input_data.get('stop_hook_active', False)

    # 記錄事件
    log_event('stop', input_data)

    # 將 JSONL transcript 轉換為可讀 JSON
    if args.chat and 'transcript_path' in input_data:
        convert_transcript_to_json(input_data['transcript_path'])

    # TTS 完成通知（使用 LLM 生成的訊息）
    if args.notify:
        announce_completion()

    sys.exit(0)

def convert_transcript_to_json(transcript_path):
    """將 JSONL 格式的 transcript 轉換為可讀的 JSON 陣列"""
    if not os.path.exists(transcript_path):
        return

    chat_data = []
    with open(transcript_path, 'r') as f:
        for line in f:
            line = line.strip()
            if line:
                try:
                    chat_data.append(json.loads(line))
                except json.JSONDecodeError:
                    pass

    chat_file = os.path.join('logs', 'chat.json')
    with open(chat_file, 'w') as f:
        json.dump(chat_data, f, indent=2)
```

> **注意：** `chat.json` 只保留最近一次對話。每次新的對話會完全覆蓋前一次的內容，不像其他日誌檔案是累加的。

## TTS 通知系統

Stop hook 的 TTS 系統依可用服務自動選擇語音引擎：

```
優先順序：ElevenLabs → OpenAI → pyttsx3（本地）
```

完成訊息由 LLM 生成（同樣有 fallback 機制）：

```
優先順序：OpenAI → Anthropic → Ollama（本地）→ 隨機預設訊息
```

這兩層 fallback 確保即使所有外部服務都不可用，hook 仍然能正常完成（使用本地方案或靜默通過）。

## 重點整理

- Setup hook 透過 `additionalContext` 注入系統提示，透過 `CLAUDE_ENV_FILE` 持久化環境變數
- SessionStart 的 `source` 區分 `startup`（新 session）、`resume`（恢復）、`clear`（清除）
- SessionEnd 適合清理暫存檔案與過期日誌
- PreCompact 在壓縮前備份完整對話記錄——壓縮後資訊不可恢復
- Stop hook 可將 JSONL transcript 轉換為可讀 JSON，搭配 TTS 語音通知
- 所有通知系統都有多層 fallback（雲端 → 本地 → 靜默），確保不因外部服務故障而中斷

---

下一篇：[Sub-Agents](./06-sub-agents.md)
