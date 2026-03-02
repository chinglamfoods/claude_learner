# 02：Claude 的 Session 檔案結構

> 解析 `~/.claude` 目錄的結構——session 資料存放方式、如何從中擷取活躍 session 資訊與 token 用量。

## `~/.claude` 目錄結構

Claude Code 將所有 session 資料儲存在使用者家目錄下的 `.claude` 目錄中。這個目錄的結構較為複雜，需要逆向工程才能理解各檔案的用途。

```
~/.claude/
├── projects/                        ← 以專案路徑為基礎的目錄
│   ├── -home-user-repos-project-a/  ← 路徑中的 / 轉換為 -
│   │   ├── sessions/
│   │   │   ├── <session-id>/
│   │   │   │   ├── conversation.jsonl    ← 對話記錄（JSONL 格式）
│   │   │   │   └── metadata.json         ← session 元資料
│   │   │   └── ...
│   │   └── ...
│   └── -home-user-repos-project-b/
│       └── sessions/
│           └── ...
├── settings.json                    ← 全域設定
└── ...
```

**關鍵觀察：** 專案目錄名稱是將絕對路徑中的 `/` 替換為 `-`。例如 `/home/user/repos/my-project` 對應的目錄名稱為 `-home-user-repos-my-project`。

## Session 資料解析

每個 session 目錄中的核心檔案：

### conversation.jsonl

JSONL（JSON Lines）格式，每一行是一個獨立的 JSON 物件，記錄對話中的每一次互動：

```go
// 對話記錄的基本結構（Go struct）
type ConversationEntry struct {
    Type      string    `json:"type"`       // "human", "assistant", "system"
    Timestamp time.Time `json:"timestamp"`
    Message   string    `json:"message"`
    // token 用量資訊嵌入在 assistant 類型的 entry 中
    Usage     *Usage    `json:"usage,omitempty"`
}

type Usage struct {
    InputTokens  int `json:"input_tokens"`
    OutputTokens int `json:"output_tokens"`
    CacheRead    int `json:"cache_read_input_tokens,omitempty"`
    CacheWrite   int `json:"cache_creation_input_tokens,omitempty"`
}
```

### 解析 JSONL 檔案

```go
package parser

import (
    "bufio"
    "encoding/json"
    "os"
)

// ParseConversation 逐行讀取 JSONL 檔案，解析為結構化資料
func ParseConversation(filePath string) ([]ConversationEntry, error) {
    file, err := os.Open(filePath)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    var entries []ConversationEntry
    scanner := bufio.NewScanner(file)
    // JSONL 中某些行可能很長，需要加大 buffer
    scanner.Buffer(make([]byte, 0, 1024*1024), 10*1024*1024)

    for scanner.Scan() {
        var entry ConversationEntry
        if err := json.Unmarshal(scanner.Bytes(), &entry); err != nil {
            // 跳過格式不正確的行，避免單行錯誤中斷整個解析
            continue
        }
        entries = append(entries, entry)
    }

    return entries, scanner.Err()
}
```

## 掃描所有專案與 Session

```go
package scanner

import (
    "os"
    "path/filepath"
    "strings"
)

type SessionInfo struct {
    ProjectPath string // 原始專案路徑
    SessionID   string // session 目錄名稱
    Directory   string // session 資料目錄的完整路徑
    IsActive    bool   // 是否仍在執行中
}

// ScanSessions 掃描 ~/.claude/projects 下所有 session
func ScanSessions(claudeDir string) ([]SessionInfo, error) {
    projectsDir := filepath.Join(claudeDir, "projects")
    entries, err := os.ReadDir(projectsDir)
    if err != nil {
        return nil, err
    }

    var sessions []SessionInfo
    for _, projectEntry := range entries {
        if !projectEntry.IsDir() {
            continue
        }

        // 將目錄名稱還原為原始路徑
        // -home-user-repos-project → /home/user/repos/project
        projectPath := dirNameToPath(projectEntry.Name())

        sessionsDir := filepath.Join(projectsDir, projectEntry.Name(), "sessions")
        sessionEntries, err := os.ReadDir(sessionsDir)
        if err != nil {
            continue // 該專案可能沒有 sessions 目錄
        }

        for _, sessionEntry := range sessionEntries {
            if !sessionEntry.IsDir() {
                continue
            }
            sessions = append(sessions, SessionInfo{
                ProjectPath: projectPath,
                SessionID:   sessionEntry.Name(),
                Directory:   filepath.Join(sessionsDir, sessionEntry.Name()),
            })
        }
    }

    return sessions, nil
}

// dirNameToPath 將 Claude 的目錄命名慣例還原為檔案系統路徑
func dirNameToPath(dirName string) string {
    // 移除開頭的 -，然後將 - 替換為 /
    // 注意：這是簡化版，實際可能需要更精細的處理
    if strings.HasPrefix(dirName, "-") {
        dirName = dirName[1:]
    }
    return "/" + strings.ReplaceAll(dirName, "-", "/")
}
```

> **注意：** `dirNameToPath` 的實作是簡化版。如果專案路徑本身包含 `-`（例如 `my-project`），轉換會不正確。實務上需要更精細的對應邏輯，或保留原始目錄名稱作為唯一識別，另外從 session 的 metadata 中讀取實際路徑。

## 判斷 Session 是否活躍

判斷一個 session 是否正在執行中，可以透過以下方式：

```go
import (
    "os"
    "time"
)

// IsSessionActive 根據 conversation 檔案的最後修改時間判斷 session 是否活躍
// 如果最後修改時間在 threshold 內，視為活躍
func IsSessionActive(sessionDir string, threshold time.Duration) bool {
    convFile := filepath.Join(sessionDir, "conversation.jsonl")
    info, err := os.Stat(convFile)
    if err != nil {
        return false
    }

    return time.Since(info.ModTime()) < threshold
}
```

以 5 分鐘作為 threshold 是合理的起點——如果 conversation 檔案在過去 5 分鐘內有更新，代表 session 仍在互動中。

## Token 用量計算

從 conversation 記錄中累計 token 用量，並根據 API 定價計算成本：

```go
type SessionMetrics struct {
    TotalInputTokens  int
    TotalOutputTokens int
    CacheReadTokens   int
    CacheWriteTokens  int
    EstimatedCostUSD  float64
    Duration          time.Duration
    MessageCount      int
}

// CalculateMetrics 從對話記錄中計算 session 的用量指標
func CalculateMetrics(entries []ConversationEntry) SessionMetrics {
    var m SessionMetrics
    var firstTimestamp, lastTimestamp time.Time

    for _, entry := range entries {
        m.MessageCount++

        if firstTimestamp.IsZero() {
            firstTimestamp = entry.Timestamp
        }
        lastTimestamp = entry.Timestamp

        if entry.Usage != nil {
            m.TotalInputTokens += entry.Usage.InputTokens
            m.TotalOutputTokens += entry.Usage.OutputTokens
            m.CacheReadTokens += entry.Usage.CacheRead
            m.CacheWriteTokens += entry.Usage.CacheWrite
        }
    }

    m.Duration = lastTimestamp.Sub(firstTimestamp)

    // 基於 Claude API 定價估算成本（Sonnet 4 為例）
    // 實際價格需根據使用的模型調整
    m.EstimatedCostUSD = float64(m.TotalInputTokens) / 1_000_000 * 3.0 +
        float64(m.TotalOutputTokens) / 1_000_000 * 15.0 +
        float64(m.CacheReadTokens) / 1_000_000 * 0.30 +
        float64(m.CacheWriteTokens) / 1_000_000 * 3.75

    return m
}
```

## 重點整理

- Claude Code 的 session 資料存放在 `~/.claude/projects/` 下，以專案路徑轉換後的名稱作為目錄
- 對話記錄使用 JSONL 格式，每行一個 JSON 物件，token 用量嵌入在 assistant 回應中
- 解析 JSONL 時需加大 scanner buffer，某些行可能非常長
- 判斷 session 是否活躍可透過 conversation 檔案的最後修改時間
- 目錄名稱還原為路徑的邏輯需要注意原始路徑中包含 `-` 的情況

---

下一篇：[Go 後端與 SQLite](./03-backend-and-sqlite.md)
