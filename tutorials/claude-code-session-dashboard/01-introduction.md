# 01：Claude Code Session 管理儀表板介紹

> 建構一個即時 Web 儀表板，用來監控多個 Claude Code session 的執行狀態、token 用量與成本。

## 要解決的問題

當你同時在多個 Git work tree 或目錄中執行 Claude Code session 時，管理上會碰到幾個實際的痛點：

- **Session 分散** — 不同的 session 散落在各個終端機分頁中，需要逐一切換才能確認狀態
- **無法追蹤進度** — 不清楚哪些 session 還在跑、哪些已經閒置、各自在處理什麼任務
- **Token 用量不透明** — 缺乏即時的 token 消耗與成本資訊，難以評估訂閱方案是否划算

典型的工作流程：一個 work tree 處理測試覆蓋率改善、一個在 debug 複雜問題、一個處理小型 issue。每個都需要獨立的 Claude session，一天下來可能累積 5–10 個 session，其中某些可能持續運行超過 30 小時。

## 解決方案架構

這個工具是一個 Web 儀表板，直接讀取 Claude Code 的本地檔案系統資料，透過即時介面呈現所有 session 的狀態。

```
┌─────────────────┐     WebSocket     ┌──────────────────┐
│   React 前端     │ ◄──────────────► │    Go 後端        │
│  TypeScript      │                  │    Gin Framework   │
│  Vite + Tailwind │                  │    gorilla/ws     │
│  Chart.js        │                  │                    │
└─────────────────┘                  └────────┬───────────┘
                                              │
                                     ┌────────┴───────────┐
                                     │                     │
                              ┌──────┴──────┐    ┌────────┴────────┐
                              │   SQLite     │    │  ~/.claude/     │
                              │   資料庫     │    │  session 檔案   │
                              └─────────────┘    └─────────────────┘
```

| 元件 | 技術選擇 | 說明 |
|------|---------|------|
| 後端 | Go + Gin | HTTP 伺服器與 API 端點 |
| 即時通訊 | gorilla/websocket | 瀏覽器與後端的雙向通訊 |
| 資料儲存 | SQLite | session 指標與歷史資料 |
| 前端 | React + TypeScript + Vite | 儀表板 UI |
| 樣式 | Tailwind CSS | Utility-first CSS |
| 圖表 | Chart.js | Token 用量隨時間變化的視覺化 |
| 部署 | Docker | 掛載 `~/.claude` 目錄即可運行 |

## 核心功能

- 即時監控所有目錄下的 Claude Code session
- Token 用量追蹤與時間序列圖表
- 各 session 的成本細分
- 每個 session 的檔案變更計數
- 多瀏覽器支援，自動同步更新
- 唯讀設計——只觀察不修改 session，不干擾 Claude 本身的運作

## 技術先決條件

- Go 1.21+
- Node.js 18+（前端建構）
- Docker（部署用）
- 本地安裝的 Claude Code（需要存取 `~/.claude` 目錄）

## 快速啟動（Docker）

```bash
docker pull ksred/claude-session-manager
docker run -p 3000:3000 -v ~/.claude:/root/.claude ksred/claude-session-manager
```

開啟 `http://localhost:3000` 即可查看儀表板。

## 本教學涵蓋的內容

1. **介紹**（本篇）— 問題陳述、架構概覽
2. **Claude 的 Session 檔案結構** — `~/.claude` 目錄的資料格式與解析方式
3. **Go 後端與 SQLite** — Gin 伺服器、session 檔案監控、SQLite 並行寫入處理
4. **WebSocket 即時更新** — 雙向通訊實作與多瀏覽器同步
5. **React 儀表板前端** — TypeScript 元件、Chart.js 圖表、Tailwind 樣式
6. **Docker 部署與實務洞察** — 容器化部署、成本數據分析

## 參考來源

- [Managing Multiple Claude Code Sessions: Building a Real-Time Dashboard](https://www.ksred.com/managing-multiple-claude-code-sessions-building-a-real-time-dashboard/)
- Docker image: `hub.docker.com/r/ksred/claude-session-manager`

## 重點整理

- 這個工具解決多 Claude Code session 管理的可見性問題
- 架構採用 Go 後端 + React 前端 + SQLite + WebSocket 的組合
- 直接讀取 `~/.claude` 目錄中的 session 檔案，唯讀設計不干擾 Claude 運作
- Docker 部署只需掛載 `~/.claude` 目錄即可使用

---

下一篇：[Claude 的 Session 檔案結構](./02-claude-session-filesystem.md)
