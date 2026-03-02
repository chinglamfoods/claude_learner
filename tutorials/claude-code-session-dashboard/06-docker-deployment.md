# 06：Docker 部署與實務洞察

> Docker 容器化部署的設定方式，以及從實際使用中觀察到的 Claude Code session 成本與行為模式。

## Dockerfile

使用 multi-stage build 分離前端建構與後端編譯，產出精簡的最終 image：

```dockerfile
# 階段 1：建構前端
FROM node:20-alpine AS frontend
WORKDIR /app/web
COPY web/package*.json ./
RUN npm ci
COPY web/ ./
RUN npm run build

# 階段 2：編譯 Go 後端
FROM golang:1.22-alpine AS backend
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY cmd/ ./cmd/
COPY internal/ ./internal/
RUN CGO_ENABLED=1 go build -o server ./cmd/server

# 階段 3：最終 image
FROM alpine:3.19
RUN apk add --no-cache sqlite-libs
WORKDIR /app

COPY --from=backend /app/server .
COPY --from=frontend /app/web/dist ./web/dist

EXPOSE 3000

ENTRYPOINT ["./server"]
```

> **注意 `CGO_ENABLED=1`：** `go-sqlite3` 是 CGo 綁定，需要啟用 CGo 編譯。Alpine image 需要安裝 `gcc` 和 `musl-dev`（在 backend stage 中）。如果要避免 CGo，可改用純 Go 的 SQLite 實作如 `modernc.org/sqlite`。

## 執行容器

```bash
# 基本執行：掛載 ~/.claude 目錄
docker run -d \
  --name claude-dashboard \
  -p 3000:3000 \
  -v ~/.claude:/root/.claude:ro \
  ksred/claude-session-manager

# 開啟 http://localhost:3000
```

| 參數 | 說明 |
|------|------|
| `-v ~/.claude:/root/.claude:ro` | 以唯讀模式掛載 Claude 的資料目錄，工具不會修改任何 session 資料 |
| `-p 3000:3000` | 對應容器內的服務埠號 |
| `-d` | 背景執行 |

### 掛載為唯讀

`:ro` flag 確保容器內的程式無法修改 `~/.claude` 目錄中的任何檔案。這是重要的安全措施——儀表板的設計原則是唯讀觀察，不應該有任何修改 session 資料的能力。

## Docker Compose

如果需要持久化 SQLite 資料庫（跨容器重啟保留歷史資料）：

```yaml
# docker-compose.yml
services:
  dashboard:
    image: ksred/claude-session-manager
    ports:
      - "3000:3000"
    volumes:
      - ~/.claude:/root/.claude:ro      # Claude session 資料（唯讀）
      - dashboard-data:/app/data         # SQLite 資料庫持久化
    restart: unless-stopped

volumes:
  dashboard-data:
```

```bash
docker compose up -d
```

## 部署到遠端伺服器

如果需要從其他裝置（如手機）監控 session：

```bash
# 在開發機器上同步 .claude 目錄到遠端（需要 rsync）
rsync -avz ~/.claude/ remote-server:~/.claude/

# 或在遠端伺服器上使用 SSH tunnel
ssh -R 3000:localhost:3000 remote-server
```

更好的做法是在遠端伺服器上部署 Docker 容器，搭配 reverse proxy（如 Caddy 或 nginx）加上 HTTPS。

## 實務使用洞察

作者在實際使用中觀察到的數據模式：

### 成本數據

| 指標 | 數值 |
|------|------|
| 單一 session 最高成本 | > $190 USD |
| 每日平均 API 等價成本 | > $200 USD（每週至少 4 天） |
| Max Plan 月費 | $200 USD |

以 Max Plan 每月 $200 的訂閱費用計算，如果每天的 API 等價用量超過 $200，每週工作 4–5 天，月度 API 等價總成本約 $3,200–$4,000。訂閱方案的投資報酬率極高。

### Session 行為模式

- **短 session（< 1 小時）** — 處理小型 issue、快速修復、程式碼審查
- **中 session（1–4 小時）** — 功能開發、測試撰寫、重構
- **長 session（4–30+ 小時）** — 複雜 debug、大型功能、涉及多個系統的問題排查

長 session 通常橫跨工作日——開發者離開後 session 保持開啟，隔天繼續。這類 session 的 token 消耗特別高，因為累積的對話上下文持續增長。

### 並行 Session 管理

作者的典型工作流程使用 3 個以上的並行 work tree：

```
work-tree-1/  ← 測試覆蓋率改善（長期 session）
work-tree-2/  ← 複雜 bug debug（可能持續數天）
work-tree-3/  ← 小型 issue 處理（一天內多次開關 session）
```

儀表板在超過 3 個 session 時最有價值——一眼就能看到所有 session 的狀態，不需要在終端機分頁之間切換。

## 搭配 CC Switch 使用

[CC Switch](https://github.com/ksred/cc-switch) 是同一作者開發的 Git work tree 管理工具，與 session 儀表板互補：

- **CC Switch** — 管理 work tree 的建立、切換、清理
- **Session Dashboard** — 監控各 work tree 中 Claude session 的狀態與成本

兩者搭配形成完整的多工作流管理方案。

## 延伸開發方向

如果你要基於這個架構建構自己的工具，以下是一些可考慮的方向：

- **告警機制** — 當 session 成本超過閾值時發送通知（Slack、email）
- **歷史趨勢** — 跨日、跨週的 token 用量趨勢圖
- **Session 標籤** — 為 session 添加自訂標籤（功能名稱、issue 編號）
- **團隊儀表板** — 共享伺服器上的多使用者 session 監控
- **API 整合** — 提供 REST API 讓其他工具查詢 session 狀態

## 重點整理

- Docker multi-stage build 產出精簡的 image，包含 Go 後端與前端靜態檔案
- 以 `:ro` 唯讀模式掛載 `~/.claude`，確保儀表板不會修改 session 資料
- 實際使用數據顯示 Max Plan 的投資報酬率極高——單日 API 等價成本經常超過月費
- 長 session（30+ 小時）常見於複雜 debug，token 消耗隨對話上下文累積而增長
- 儀表板在 3 個以上並行 session 時最能發揮價值，消除終端機分頁切換的摩擦
- 搭配 CC Switch 等 work tree 管理工具使用效果最佳
