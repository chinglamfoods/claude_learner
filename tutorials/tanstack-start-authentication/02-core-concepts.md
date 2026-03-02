# 02：核心概念

> 了解認證（Authentication）與授權（Authorization）之間的差異，並學習 TanStack Start 認證模型背後的架構設計。

## 認證 vs. 授權

這兩個術語在網頁開發中不斷出現，讓我們先釐清它們各自的意思：

- **認證（Authentication）** 回答的是：*「這個使用者是誰？」*
  - 登入與登出
  - 驗證身分（電子郵件／密碼、社群登入等）

- **授權（Authorization）** 回答的是：*「這個使用者可以做什麼？」*
  - 權限與角色（管理員、編輯者、檢視者）
  - 對特定頁面或操作的存取控制

TanStack Start 透過 server functions、sessions 以及路由保護，同時提供**兩者**的工具。你通常會先實作認證，然後再在其上加入授權邏輯。

## 全端架構模型

TanStack Start 使用全端架構，將認證邏輯分為三個層級：

### 伺服器端（安全）

這是處理敏感操作的地方。伺服器是**唯一**應該進行以下操作的位置：

- 儲存與驗證 sessions
- 驗證使用者憑證（密碼、tokens）
- 執行資料庫操作（查詢使用者、儲存 sessions）
- 產生與驗證 tokens
- 執行受保護的 API 端點

> **為什麼只在伺服器端？** 因為用戶端的程式碼任何人都可以透過瀏覽器開發者工具看到。密碼、資料庫查詢以及密鑰絕對不能在瀏覽器中執行。

### 用戶端（公開）

用戶端（瀏覽器）負責處理面向使用者的部分：

- 在 React 中管理認證狀態（是否已登入）
- 根據認證狀態顯示或隱藏 UI
- 登入與登出表單
- 將使用者重新導向至正確的頁面

### 同構（兩端皆執行）

有些程式碼會在伺服器端和用戶端同時執行：

- 檢查認證狀態的路由 loaders（在 SSR 期間於伺服器端執行，在導航時可能會在用戶端重新執行）
- 共用的驗證邏輯
- 使用者個人資料的資料存取

## Session 管理模式

管理使用者 sessions 有幾種方式，以下是比較：

### HTTP-Only Cookies（建議採用）

```
瀏覽器在每次請求時自動送出 cookie
    ↓
伺服器讀取 cookie → 驗證 session → 回傳回應
```

- **最安全** — 設定 `httpOnly: true` 的 cookies 無法被 JavaScript 讀取（可防範 XSS 攻擊）
- **自動化** — 瀏覽器會自動在每次請求中附帶 cookies
- **內建 CSRF 防護** — 透過 `sameSite` 屬性實現
- **最適合** 傳統網頁應用程式（大多數應用程式都屬於此類！）

TanStack Start 預設採用此方式。

### JWT Tokens

- 無狀態 — token 本身包含使用者資訊，不需要伺服器端的儲存
- 適合 API 優先的應用程式
- **注意：** 將 JWT 儲存在 localStorage 中容易受到 XSS 攻擊
- 建議使用 refresh token 輪換機制以提升安全性

### 伺服器端 Sessions

- Session 資料儲存在資料庫或 Redis 中
- 容易撤銷（直接刪除 session 記錄即可）
- 需要一個 session store
- 適合需要立即使 sessions 失效的場景（例如「登出所有裝置」）

## 路由保護架構

TanStack Start 提供多種保護路由的模式：

### Layout Route 模式（建議採用）

透過將頁面群組包裹在一個會檢查認證狀態的 **layout route** 中，來保護整組頁面：

```
routes/
  _authed.tsx          ← 檢查使用者是否已登入
  _authed/
    dashboard.tsx      ← 受保護（_authed 的子路由）
    settings.tsx       ← 受保護（_authed 的子路由）
    admin/
      index.tsx        ← 受保護（_authed 的子路由）
  login.tsx            ← 公開
  index.tsx            ← 公開
```

任何在 `_authed/` 內的路由都會自動要求認證，因為父層的 layout 會先進行檢查。

### 元件層級保護

根據認證狀態顯示或隱藏頁面的部分內容。適用於單一頁面同時包含公開與私有內容的情境。

### Server Function Guards

在執行敏感操作之前，先在伺服器端驗證認證狀態。即使你已經有路由層級的保護，這仍然是必要的 — 務必在伺服器端進行驗證。

## 重點整理

- **認證** = 使用者是誰；**授權** = 使用者可以做什麼
- 敏感邏輯（密碼、sessions、資料庫查詢）只能放在**伺服器端**
- HTTP-only cookies 是建議採用的 session 管理方式
- Layout routes 是保護頁面群組最簡潔的方式
- 即使用戶端也有檢查，仍然務必在伺服器端驗證認證狀態

---

下一篇：[Server Functions 與認證](./03-server-functions.md)
