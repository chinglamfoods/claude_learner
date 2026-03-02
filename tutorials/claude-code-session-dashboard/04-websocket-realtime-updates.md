# 04：WebSocket 即時更新

> 使用 gorilla/websocket 實作後端與瀏覽器之間的雙向通訊，支援多個瀏覽器同時連線並自動同步。

## 為什麼用 WebSocket 而非 Polling

| 方式 | 延遲 | 伺服器負載 | 實作複雜度 |
|------|------|-----------|-----------|
| 短輪詢（Short polling） | 取決於輪詢間隔（通常 1–5 秒） | 高（大量空請求） | 低 |
| 長輪詢（Long polling） | 低 | 中 | 中 |
| Server-Sent Events (SSE) | 低 | 低 | 低（但只能單向） |
| **WebSocket** | **即時** | **低** | **中** |

WebSocket 在建立連線後保持雙向通訊通道，session 更新時伺服器主動推送，不需要客戶端輪詢。對於儀表板這種需要即時更新的場景，WebSocket 是最合適的選擇。

## Hub 模式

gorilla/websocket 的標準做法是使用 Hub 管理所有連線：

```go
// internal/ws/hub.go
package ws

import (
    "encoding/json"
    "log"
    "net/http"
    "sync"

    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    // 開發環境允許跨域，生產環境應限制來源
    CheckOrigin: func(r *http.Request) bool {
        return true
    },
}

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    mu         sync.RWMutex
}

type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
    }
}
```

## Hub 的事件迴圈

```go
// Run 啟動 Hub 的主迴圈，處理連線註冊、斷線與廣播
func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client] = true
            h.mu.Unlock()
            log.Printf("新連線，目前連線數: %d", len(h.clients))

        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            h.mu.Unlock()
            log.Printf("連線斷開，目前連線數: %d", len(h.clients))

        case message := <-h.broadcast:
            h.mu.RLock()
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    // 如果 client 的 send channel 已滿，視為斷線
                    close(client.send)
                    delete(h.clients, client)
                }
            }
            h.mu.RUnlock()
        }
    }
}
```

## 處理 WebSocket 連線

```go
// HandleWebSocket 將 HTTP 連線升級為 WebSocket
func (h *Hub) HandleWebSocket(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("WebSocket 升級失敗: %v", err)
        return
    }

    client := &Client{
        hub:  h,
        conn: conn,
        send: make(chan []byte, 256),
    }

    h.register <- client

    // 每個 client 啟動兩個 goroutine：讀取與寫入
    go client.writePump()
    go client.readPump()
}
```

## Client 的讀寫 Goroutine

```go
import (
    "time"
)

const (
    writeWait      = 10 * time.Second    // 寫入操作的超時時間
    pongWait       = 60 * time.Second    // 等待 pong 回應的超時時間
    pingPeriod     = (pongWait * 9) / 10 // ping 的發送間隔（pongWait 的 90%）
    maxMessageSize = 512 * 1024          // 最大訊息大小
)

// readPump 持續讀取 client 傳來的訊息
// 儀表板場景中 client 較少主動發送訊息，但需要維持心跳
func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        // 收到 pong 時重設讀取超時
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, _, err := c.conn.ReadMessage()
        if err != nil {
            break
        }
    }
}

// writePump 將 send channel 中的訊息寫入 WebSocket 連線
func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                // Hub 已關閉此 client 的 send channel
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)

            // 批次寫入：如果 send channel 中還有排隊的訊息，一併發送
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte("\n"))
                w.Write(<-c.send)
            }

            if err := w.Close(); err != nil {
                return
            }

        case <-ticker.C:
            // 定期發送 ping 維持連線
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

## 從 Watcher 推送更新

當檔案監控偵測到 session 變更時，透過 Hub 廣播給所有連線的瀏覽器：

```go
// Broadcast 將結構化資料序列化後推送給所有連線的 client
func (h *Hub) Broadcast(data interface{}) {
    payload, err := json.Marshal(data)
    if err != nil {
        log.Printf("序列化廣播資料失敗: %v", err)
        return
    }

    h.broadcast <- payload
}

// 在 watcher 中呼叫
func (w *Watcher) handleSessionUpdate(filePath string) {
    session := w.parseAndStore(filePath)

    // 推送更新給所有連線的瀏覽器
    w.hub.Broadcast(map[string]interface{}{
        "type":    "session_update",
        "payload": session,
    })
}
```

## 訊息格式

定義統一的 WebSocket 訊息格式，方便前端處理不同類型的更新：

```go
type WSMessage struct {
    Type    string      `json:"type"`
    Payload interface{} `json:"payload"`
}

// 常見的訊息類型
// "session_update"    — 單一 session 的狀態更新
// "sessions_snapshot" — 所有 session 的完整快照（新連線時發送）
// "session_removed"   — session 被移除
```

新的瀏覽器連線時，立即推送所有 session 的完整快照，讓儀表板可以立即顯示當前狀態，而非等待下一次變更事件。

## 重點整理

- WebSocket 提供即時雙向通訊，適合儀表板等需要即時更新的場景
- Hub 模式管理所有 client 連線，統一處理註冊、斷線與廣播
- 每個 client 有獨立的 read/write goroutine，透過 channel 通訊
- Ping/pong 心跳機制維持連線活躍並偵測斷線
- 新連線時發送完整快照，後續變更以增量更新推送
- `CheckOrigin` 在生產環境中應限制允許的來源網域

---

下一篇：[React 儀表板前端](./05-react-dashboard.md)
