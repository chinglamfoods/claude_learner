# 07：團隊驗證系統

> Builder/Validator agent 配對模式、Task 系統協調、`/plan_w_team` 自我驗證 meta prompt、以及 PostToolUse 程式碼品質驗證。

## Builder/Validator 模式

增加運算量來提升工作品質的信賴度。每個實作任務由兩個 agent 處理：

| Agent | 角色 | 工具存取 | 自我驗證 |
|-------|------|---------|---------|
| **Builder** | 實作功能、寫程式碼 | 所有工具 | Ruff + Ty（寫入 .py 時自動觸發） |
| **Validator** | 驗證 builder 的工作 | 唯讀（禁止 Write/Edit） | 無 |

### Builder Agent

```yaml
---
name: builder
description: 通用工程 agent，一次執行一個任務。在需要寫程式碼、建立檔案、實作功能時使用。
model: opus
color: cyan
hooks:
  PostToolUse:
    - matcher: "Write|Edit"
      hooks:
        - type: command
          command: >-
            uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/ruff_validator.py
        - type: command
          command: >-
            uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/ty_validator.py
---
```

Builder 的 `hooks` 欄位在 agent 層級定義了 PostToolUse hooks——每次 builder 寫入或編輯 `.py` 檔案時，Ruff（linting）和 Ty（型別檢查）自動執行。如果有錯誤，`decision: "block"` 會回饋給 builder，讓它立即修正。

### Validator Agent

```yaml
---
name: validator
description: 唯讀驗證 agent，檢查任務是否成功完成。在 builder 完成後使用以驗證工作是否符合驗收標準。
model: opus
disallowedTools: Write, Edit, NotebookEdit
color: yellow
---
```

Validator **不能修改檔案**。它只能讀取、搜尋、執行指令來驗證 builder 的工作。這確保驗證結果的獨立性。

## Task 系統協調

Claude Code 的 Task 系統用於建立和管理 agent 之間的工作分配：

| Tool | 功能 |
|------|------|
| `TaskCreate` | 建立任務，指定 owner、描述、相依性 |
| `TaskUpdate` | 更新狀態、加入阻擋關係、標記完成 |
| `TaskList` | 查看所有任務及目前狀態 |
| `TaskGet` | 取得特定任務的完整詳情 |

### 工作流程

```
1. 主 agent 建立任務列表，指定 owner（builder/validator）
2. 任務可以平行執行或設定相依阻擋
3. Sub-agent 完成工作後透過 TaskUpdate 回報
4. 主 agent 即時反應完成的任務
5. 被阻擋的任務在相依性解決後自動解鎖
```

這消除了使用 bash sleep 迴圈等待的需要——Task 系統自動處理協調。

## `/plan_w_team` Meta Prompt

`/plan_w_team` 是一個**自我驗證的模板 meta prompt**，有三個核心能力：

### 1. 自我驗證

Prompt 的 frontmatter 包含嵌入式 hooks，在規劃 agent 完成後自動驗證輸出：

```yaml
hooks:
  stop:
    - command: "uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/validate_new_file.py specs/*.md"
    - command: "uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/validate_file_contains.py"
```

驗證確保：
- 在正確的目錄中建立了 spec 檔案
- 檔案包含必要的區段（團隊編成、步驟化任務等）

如果驗證失敗，agent 收到回饋並繼續工作直到輸出符合標準。

### 2. 模板化輸出

產生的計畫遵循固定格式，確保結果可預測：

```markdown
### {{PLAN_NAME}}
**Task:** {{TASK_DESCRIPTION}}
**Objective:** {{OBJECTIVE}}

### Team Orchestration
{{TEAM_MEMBERS}}

### Step-by-Step Tasks
{{TASKS}}
```

這是 agentic engineering 與隨意使用 AI 的區別——你**知道** agent 會產生什麼格式的輸出。

### 3. Agent 協調

計畫中指定每個任務的 owner 與相依性：

```markdown
#### Task 1: 實作使用者認證 API
- **Owner:** auth_builder
- **Dependencies:** 無

#### Task 2: 驗證認證 API
- **Owner:** auth_validator
- **Dependencies:** Task 1
- **Acceptance Criteria:**
  - [ ] API endpoint 回傳正確的 HTTP status codes
  - [ ] Token 過期機制正常運作
  - [ ] 錯誤情境有適當處理
```

## PostToolUse 程式碼品質驗證

Builder agent 的 hooks 設定中使用 PostToolUse validators 自動執行品質檢查：

### Ruff Validator

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.8"
# ///

import json
import sys
import subprocess

def main():
    input_data = json.load(sys.stdin)
    tool_name = input_data.get('tool_name', '')
    tool_input = input_data.get('tool_input', {})

    # 只在 Write 或 Edit Python 檔案時觸發
    if tool_name not in ('Write', 'Edit'):
        sys.exit(0)

    file_path = tool_input.get('file_path', '')
    if not file_path.endswith('.py'):
        sys.exit(0)

    # 執行 Ruff linting
    try:
        result = subprocess.run(
            ['ruff', 'check', file_path],
            capture_output=True, text=True, timeout=30
        )
        if result.returncode != 0:
            # 有 lint 錯誤——阻擋並回饋給 Claude
            output = {
                "decision": "block",
                "reason": f"Ruff lint 錯誤：\n{result.stdout}\n請修正後重試"
            }
            print(json.dumps(output))
    except Exception:
        pass

    sys.exit(0)

if __name__ == '__main__':
    main()
```

### Ty Validator（型別檢查）

```python
# 結構類似 Ruff validator
result = subprocess.run(
    ['ty', 'check', file_path],
    capture_output=True, text=True, timeout=30
)
if result.returncode != 0:
    output = {
        "decision": "block",
        "reason": f"型別檢查錯誤：\n{result.stdout}\n請修正型別問題後重試"
    }
    print(json.dumps(output))
```

### Matcher 篩選

在 builder 的 hooks 設定中，`matcher: "Write|Edit"` 確保 validators 只在寫入或編輯操作時觸發，不會在 Read 或 Grep 時浪費資源：

```yaml
hooks:
  PostToolUse:
    - matcher: "Write|Edit"      # 只匹配 Write 和 Edit 工具
      hooks:
        - type: command
          command: >-
            uv run $CLAUDE_PROJECT_DIR/.claude/hooks/validators/ruff_validator.py
```

## 完整的工作流程範例

```bash
# 1. 使用 /plan_w_team 建立計畫
/plan_w_team

# 使用者 prompt："更新 hooks 文件並加入缺少的 status lines"
# 協調 prompt："為每個 hook 建立 builder 和 validator agent 群組"

# 2. 計畫自動產生，包含：
#    - 團隊成員（session_end_builder, session_end_validator 等）
#    - 步驟化任務與相依性
#    - 驗證指令

# 3. 執行計畫
/build

# 4. 觀察 agents 平行工作：
#    - Builders 實作功能（寫入 .py 時自動 lint + type check）
#    - Validators 驗證完成度（唯讀檢查）
#    - Task 系統自動協調相依性
```

## 重點整理

- Builder/Validator 配對透過增加運算量提升工作品質的信賴度
- Builder 有完整工具存取，寫入 `.py` 時自動觸發 Ruff + Ty 驗證
- Validator 只有唯讀存取，確保驗證的獨立性
- Task 系統（TaskCreate/TaskUpdate/TaskList/TaskGet）處理 agent 之間的工作協調
- `/plan_w_team` 是自我驗證的模板 meta prompt——產生格式可預測的計畫
- PostToolUse validators 使用 `matcher` 篩選只在特定工具觸發時執行

---

下一篇：[Status Lines 與 Output Styles](./08-status-lines-and-output-styles.md)
