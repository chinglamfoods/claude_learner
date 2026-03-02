# 03：Go 後端與 SQLite

> 使用 Gin 框架建構 HTTP 伺服器，搭配 SQLite 儲存 session 指標資料，並處理 Go goroutine 環境下的並行寫入問題。

## 專案結構

```
claude-session-manager/
├── cmd/
│   └── server/
│       └── main.go            ← 進入點
├── internal/
│   ├── parser/                ← session 檔案解析
│   ├── scanner/               ← 目錄掃描
│   ├── store/                 ← SQLite 資料層
│   ├── watcher/               ← 檔案系統監控
│   └── ws/                    ← WebSocket 管理
├── web/                       ← React 前端（建構後的靜態檔案）
├── go.mod
├── go.sum
└── Dockerfile
```

## Gin 伺服器設定

```go
// cmd/server/main.go
package main

import (
    "log"
    "os"
    "path/filepath"

    "github.com/gin-gonic/gin"
    "claude-session-manager/internal/store"
    "claude-session-manager/internal/watcher"
    "claude-session-manager/internal/ws"
)

func main() {
    homeDir, _ := os.UserHomeDir()
    claudeDir := filepath.Join(homeDir, ".claude")

    // 初始化 SQLite
    db, err := store.New("sessions.db")
    if err != nil {
        log.Fatalf("無法初始化資料庫: %v", err)
    }
    defer db.Close()

    // 初始化 WebSocket hub
    hub := ws.NewHub()
    go hub.Run()

    // 啟動檔案系統監控
    w, err := watcher.New(claudeDir, db, hub)
    if err != nil {
        log.Fatalf("無法啟動檔案監控: %v", err)
    }
    go w.Start()

    // 設定 Gin 路由
    r := gin.Default()

    // API 端點
    api := r.Group("/api")
    {
        api.GET("/sessions", func(c *gin.Context) {
            sessions, err := db.GetAllSessions()
            if err != nil {
                c.JSON(500, gin.H{"error": err.Error()})
                return
            }
            c.JSON(200, sessions)
        })

        api.GET("/sessions/:id/metrics", func(c *gin.Context) {
            id := c.Param("id")
            metrics, err := db.GetSessionMetrics(id)
            if err != nil {
                c.JSON(404, gin.H{"error": "session not found"})
                return
            }
            c.JSON(200, metrics)
        })
    }

    // WebSocket 端點
    r.GET("/ws", func(c *gin.Context) {
        hub.HandleWebSocket(c.Writer, c.Request)
    })

    // 前端靜態檔案
    r.Static("/assets", "./web/dist/assets")
    r.StaticFile("/", "./web/dist/index.html")

    r.Run(":3000")
}
```

## SQLite 資料層與並行寫入

這是整個專案中最容易出問題的部分。SQLite 一次只能處理一個寫入操作。在 Go 中大量使用 goroutine 的情境下，如果不做並行控制，會迅速遇到 `database is locked` 錯誤甚至資料庫檔案損毀。

### 使用 Mutex 保護寫入

```go
// internal/store/store.go
package store

import (
    "database/sql"
    "sync"
    "time"

    _ "github.com/mattn/go-sqlite3"
)

type Store struct {
    db *sql.DB
    mu sync.Mutex // 保護所有寫入操作
}

func New(dbPath string) (*Store, error) {
    // WAL mode 允許讀取不被寫入阻塞
    db, err := sql.Open("sqlite3", dbPath+"?_journal_mode=WAL&_busy_timeout=5000")
    if err != nil {
        return nil, err
    }

    // SQLite 在並行環境下只允許一個連線寫入
    // 設定 MaxOpenConns 為 1 是另一種防止寫入衝突的方式
    // 但搭配 mutex 更有彈性——讀取不受限制
    db.SetMaxOpenConns(10)

    s := &Store{db: db}
    if err := s.migrate(); err != nil {
        return nil, err
    }

    return s, nil
}

func (s *Store) migrate() error {
    s.mu.Lock()
    defer s.mu.Unlock()

    _, err := s.db.Exec(`
        CREATE TABLE IF NOT EXISTS sessions (
            id TEXT PRIMARY KEY,
            project_path TEXT NOT NULL,
            started_at DATETIME,
            last_active_at DATETIME,
            is_active BOOLEAN DEFAULT 0,
            total_input_tokens INTEGER DEFAULT 0,
            total_output_tokens INTEGER DEFAULT 0,
            estimated_cost_usd REAL DEFAULT 0,
            file_changes INTEGER DEFAULT 0,
            message_count INTEGER DEFAULT 0
        );

        CREATE TABLE IF NOT EXISTS token_snapshots (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            session_id TEXT NOT NULL,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
            input_tokens INTEGER,
            output_tokens INTEGER,
            cumulative_cost REAL,
            FOREIGN KEY (session_id) REFERENCES sessions(id)
        );

        CREATE INDEX IF NOT EXISTS idx_snapshots_session
            ON token_snapshots(session_id, timestamp);
    `)
    return err
}
```

### 寫入操作全部上鎖

```go
// UpsertSession 更新或新增 session 記錄
// 所有寫入操作都必須先取得 mutex lock
func (s *Store) UpsertSession(session SessionRecord) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    _, err := s.db.Exec(`
        INSERT INTO sessions (id, project_path, started_at, last_active_at,
            is_active, total_input_tokens, total_output_tokens,
            estimated_cost_usd, file_changes, message_count)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        ON CONFLICT(id) DO UPDATE SET
            last_active_at = excluded.last_active_at,
            is_active = excluded.is_active,
            total_input_tokens = excluded.total_input_tokens,
            total_output_tokens = excluded.total_output_outputs,
            estimated_cost_usd = excluded.estimated_cost_usd,
            file_changes = excluded.file_changes,
            message_count = excluded.message_count
    `, session.ID, session.ProjectPath, session.StartedAt,
        session.LastActiveAt, session.IsActive,
        session.TotalInputTokens, session.TotalOutputTokens,
        session.EstimatedCostUSD, session.FileChanges,
        session.MessageCount)

    return err
}

// InsertTokenSnapshot 記錄 token 用量快照（用於時間序列圖表）
func (s *Store) InsertTokenSnapshot(sessionID string, input, output int, cost float64) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    _, err := s.db.Exec(`
        INSERT INTO token_snapshots (session_id, input_tokens, output_tokens, cumulative_cost)
        VALUES (?, ?, ?, ?)
    `, sessionID, input, output, cost)

    return err
}
```

### 讀取操作不需要上鎖

```go
// GetAllSessions 查詢所有 session，讀取操作不需 mutex
// WAL mode 下讀取不會被寫入阻塞
func (s *Store) GetAllSessions() ([]SessionRecord, error) {
    rows, err := s.db.Query(`
        SELECT id, project_path, started_at, last_active_at,
               is_active, total_input_tokens, total_output_tokens,
               estimated_cost_usd, file_changes, message_count
        FROM sessions
        ORDER BY last_active_at DESC
    `)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var sessions []SessionRecord
    for rows.Next() {
        var s SessionRecord
        err := rows.Scan(&s.ID, &s.ProjectPath, &s.StartedAt,
            &s.LastActiveAt, &s.IsActive, &s.TotalInputTokens,
            &s.TotalOutputTokens, &s.EstimatedCostUSD,
            &s.FileChanges, &s.MessageCount)
        if err != nil {
            continue
        }
        sessions = append(sessions, s)
    }

    return sessions, nil
}

// GetSessionMetrics 取得特定 session 的 token 用量時間序列
func (s *Store) GetSessionMetrics(sessionID string) ([]TokenSnapshot, error) {
    rows, err := s.db.Query(`
        SELECT timestamp, input_tokens, output_tokens, cumulative_cost
        FROM token_snapshots
        WHERE session_id = ?
        ORDER BY timestamp ASC
    `, sessionID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var snapshots []TokenSnapshot
    for rows.Next() {
        var snap TokenSnapshot
        err := rows.Scan(&snap.Timestamp, &snap.InputTokens,
            &snap.OutputTokens, &snap.CumulativeCost)
        if err != nil {
            continue
        }
        snapshots = append(snapshots, snap)
    }

    return snapshots, nil
}
```

## SQLite 並行處理的關鍵要點

| 策略 | 說明 |
|------|------|
| **WAL mode** | `_journal_mode=WAL` 允許讀寫並行，讀取不會被寫入阻塞 |
| **Busy timeout** | `_busy_timeout=5000` 讓寫入操作在鎖定時等待最多 5 秒再回報錯誤 |
| **Mutex** | 所有寫入操作使用 `sync.Mutex` 序列化，避免多個 goroutine 同時寫入 |
| **讀取不鎖** | 讀取操作不需要 mutex，WAL mode 保證讀取不受影響 |

> **為什麼不用 PostgreSQL？** SQLite 以單一檔案運作，不需要額外的資料庫服務。這讓整個工具可以打包成一個 Docker image，使用者不需要設定資料庫連線。對於這種單機工具，SQLite 是最簡單的選擇。

## 檔案系統監控

使用 `fsnotify` 監控 `~/.claude` 目錄的變更，當 session 檔案被修改時自動更新資料庫：

```go
// internal/watcher/watcher.go
package watcher

import (
    "log"
    "path/filepath"
    "strings"
    "time"

    "github.com/fsnotify/fsnotify"
)

type Watcher struct {
    claudeDir string
    store     *store.Store
    hub       *ws.Hub
    fsWatcher *fsnotify.Watcher
}

func (w *Watcher) Start() {
    // 監控 projects 目錄下所有 session 目錄
    w.addWatchRecursive(filepath.Join(w.claudeDir, "projects"))

    // 定期全量掃描，補捉 fsnotify 可能遺漏的變更
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case event := <-w.fsWatcher.Events:
            if event.Op&(fsnotify.Write|fsnotify.Create) != 0 {
                if strings.HasSuffix(event.Name, "conversation.jsonl") {
                    w.handleSessionUpdate(event.Name)
                }
            }
        case err := <-w.fsWatcher.Errors:
            log.Printf("檔案監控錯誤: %v", err)
        case <-ticker.C:
            w.fullScan()
        }
    }
}

func (w *Watcher) handleSessionUpdate(filePath string) {
    // 解析 session 資料、更新 SQLite、透過 WebSocket 推送更新
    // ...
    w.hub.Broadcast(updatedSession)
}
```

搭配 30 秒的定期全量掃描，可以補捉 `fsnotify` 在某些邊界情況下遺漏的檔案變更（例如目錄被整個搬移時）。

## 重點整理

- Gin 框架提供 HTTP 伺服器、API 路由、以及靜態檔案服務
- SQLite 在 Go 的 goroutine 環境下需要特別處理並行寫入——使用 `sync.Mutex` 序列化所有寫入
- 開啟 WAL mode 與 busy timeout 是 SQLite 並行使用的基本設定
- 讀取操作不需要 mutex，WAL mode 保證讀寫不互相阻塞
- `fsnotify` 搭配定期全量掃描提供可靠的檔案變更偵測

---

下一篇：[WebSocket 即時更新](./04-websocket-realtime-updates.md)
