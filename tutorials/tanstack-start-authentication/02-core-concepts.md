# 02：核心概念

> 認證（Authentication）與授權（Authorization）的架構差異，以及 TanStack Start 全端認證模型的設計原理與 session 管理策略。

## 認證 vs. 授權

- **認證（Authentication）**：確認使用者身分。涵蓋登入／登出流程、憑證驗證（電子郵件／密碼、OAuth tokens、WebAuthn 等）。
- **授權（Authorization）**：判定已認證使用者的操作權限。涵蓋角色（admin、editor、viewer）、權限（permissions）、資源層級的存取控制（resource-level access control）。

TanStack Start 透過 server functions、sessions 及路由保護機制同時支援兩者。實務上，認證是授權的前提——先確認身分，再判定權限。值得注意的是，兩者的失敗回應應有所區別：認證失敗回傳 `401 Unauthorized`，授權失敗回傳 `403 Forbidden`。

## 全端架構模型

TanStack Start 的認證邏輯依執行環境分為三個層級，各有明確的安全邊界：

### 伺服器端（信任邊界內）

所有涉及機密資料的操作**必須**在伺服器端執行：

- Session 的建立、驗證與銷毀
- 憑證驗證（密碼比對、token 驗簽）
- 資料庫操作（使用者查詢、session 持久化）
- Token 簽發與驗證（JWT signing、HMAC）
- 受保護的 API 端點邏輯

> **安全原則：** 用戶端程式碼可透過瀏覽器 DevTools 完整檢視與修改。密碼雜湊、資料庫連線字串、簽章密鑰等機密資訊一旦出現在 client bundle 中即視為洩漏。TanStack Start 的 server functions 透過 RPC boundary 確保這些程式碼不會被打包至用戶端。

### 用戶端（非信任區域）

瀏覽器負責 UI 層的認證體驗：

- 維護認證狀態（透過 React context 或 state management）
- 根據認證狀態切換 UI（顯示／隱藏元素、條件渲染）
- 登入與註冊表單的呈現與前端驗證
- 導向邏輯（未認證使用者重新導向至登入頁）

**注意：** 用戶端的任何認證檢查都僅為 UX 層面的優化，不可作為安全屏障。攻擊者可直接繞過用戶端邏輯呼叫 API。所有安全關鍵的檢查必須在伺服器端重複執行。

### 同構層（Server 與 Client 皆執行）

部分程式碼在 SSR 與 CSR 中皆會執行，需特別注意其行為一致性：

- **路由 loaders**：SSR 時於伺服器端執行（可直接存取 session store），client-side navigation 時在瀏覽器端重新執行（透過 RPC 呼叫 server function）
- **共用驗證邏輯**：如 email 格式、密碼強度等純函式驗證
- **使用者資料存取**：序列化後的使用者 profile（注意不要在此層洩漏敏感欄位如 `passwordHash`）

```
┌─────────────────────────────────────────────────┐
│                   同構層                          │
│  Route Loaders / 共用驗證 / 使用者資料序列化        │
├────────────────────┬────────────────────────────┤
│   伺服器端（信任）    │      用戶端（非信任）         │
│                    │                            │
│  - Session store   │  - React 認證 state        │
│  - 密碼雜湊/比對    │  - UI 條件渲染              │
│  - DB 查詢         │  - 表單呈現 & 前端驗證       │
│  - Token 簽發/驗證  │  - 導向邏輯                 │
│  - API 端點保護     │  - （僅 UX，非安全屏障）      │
└────────────────────┴────────────────────────────┘
         ↑ RPC boundary（server functions）↑
```

## Session 管理模式

### HTTP-Only Cookies（建議採用）

```
Browser ──[自動附帶 cookie]──→ Server
                                 ├─ 讀取 cookie
                                 ├─ 驗證 session（查詢 store 或解密 token）
                                 ├─ 附加使用者資訊至 request context
                                 └─ 回傳回應（視需要更新 cookie）
```

**優勢：**
- `httpOnly: true` 防止 JavaScript 存取，有效緩解 XSS 攻擊
- 瀏覽器自動於每次同源請求附帶 cookie，無需手動管理
- 搭配 `sameSite: 'lax'`（或 `'strict'`）提供 CSRF 防護
- 搭配 `secure: true` 確保僅透過 HTTPS 傳輸

**實務注意事項：**
- Cookie 大小限制約 4KB，避免儲存大量資料於 cookie 本體——應僅存 session ID，實際資料存於伺服器端 store
- `sameSite: 'lax'` 允許 top-level navigation 的 GET 請求攜帶 cookie（如從外部連結點入），`'strict'` 則完全禁止跨站攜帶
- Session cookie 應設定合理的 `maxAge` 或 `expires`，並實作 sliding expiration（每次活動時刷新過期時間）
- TanStack Start 預設採用此模式

### JWT Tokens

**適用場景：** API-first 架構、微服務間認證、無狀態水平擴展需求。

**特性：**
- 無狀態——token 本身攜帶 claims（使用者 ID、角色、過期時間），伺服器不需維護 session store
- 適合跨服務認證（service-to-service），各服務只需共用公鑰即可驗證

**常見陷阱：**
- 將 JWT 存於 `localStorage` 會暴露於 XSS 攻擊——任何注入的腳本皆可讀取並外洩 token
- JWT 一旦簽發即無法撤銷（除非維護 blacklist，此時已失去無狀態優勢）
- Token 過大會增加每次請求的 overhead（JWT 含 header + payload + signature）
- 務必實作 refresh token rotation：短效 access token（5-15 分鐘）搭配長效 refresh token（存於 httpOnly cookie），refresh 時同時輪換 refresh token 以偵測 token 盜用

### 伺服器端 Sessions（搭配外部 Store）

**適用場景：** 需要即時撤銷能力的應用（如「登出所有裝置」、帳號被盜時的緊急停用）。

**特性：**
- Session 資料儲存於 Redis、PostgreSQL 或其他 persistent store
- 撤銷即時生效——刪除 store 中的記錄，下次請求即失效
- 可儲存任意大小的 session 資料（不受 cookie 4KB 限制）

**取捨：**
- 每次請求需查詢 session store，增加延遲（Redis 通常為亞毫秒級，可接受）
- 需維護 store 的可用性與一致性
- 水平擴展時需確保所有 server instances 存取同一個 store（或使用 sticky sessions，但不建議）

## 路由保護架構

TanStack Start 提供多層保護機制，建議採用 defense-in-depth 策略——同時在多個層級實施檢查。

### Layout Route 模式（建議作為主要防線）

透過 layout route 的 `beforeLoad` hook 檢查認證狀態，保護整個子路由樹：

```
routes/
  _authed.tsx              ← beforeLoad: 驗證 session，失敗則 redirect('/login')
  _authed/
    dashboard.tsx          ← 自動受保護（繼承父 layout 的認證檢查）
    settings.tsx           ← 自動受保護
    admin/
      _admin.tsx           ← 額外的授權檢查（驗證 role === 'admin'）
      _admin/
        users.tsx          ← 需同時通過認證 + admin 授權
        analytics.tsx      ← 需同時通過認證 + admin 授權
  login.tsx                ← 公開（已登入使用者應 redirect 至 dashboard）
  register.tsx             ← 公開
  index.tsx                ← 公開
```

**優勢：** 新增受保護頁面時只需放入對應目錄，無需逐頁加入認證邏輯。支援巢狀保護（如上方的 `_admin` 層）實現細粒度的授權控制。

**注意：** `login.tsx` 等公開頁面應檢查使用者是否已登入，若已登入則重新導向至 dashboard，避免已認證使用者看到登入表單。

### 元件層級保護

在單一頁面中根據認證狀態或權限條件渲染不同內容。適用於混合公開與私有內容的頁面（例如：文章頁面對所有人顯示內容，但僅對作者顯示編輯按鈕）。

此模式僅控制 UI 呈現，**不提供安全保護**。對應的 mutation 操作仍須在 server function 中獨立驗證權限。

### Server Function Guards（最終防線）

在每個執行敏感操作的 server function 入口處驗證認證與授權狀態。即使路由層級已有保護，server function guards 仍**不可省略**——原因：

1. 路由保護可被繞過（直接呼叫 API endpoint）
2. 重構時可能意外移除路由層級的保護
3. Server function 可能被多個路由共用，部分路由未必有保護

這是唯一真正的安全屏障，其餘層級皆為 defense-in-depth 的輔助措施。

## 重點整理

- 認證（身分確認）與授權（權限判定）應分開設計，對應不同的失敗回應（`401` vs `403`）
- 所有機密操作必須在伺服器端執行，用戶端的認證檢查僅為 UX 優化
- HTTP-only cookies 搭配 `secure`、`sameSite`、`httpOnly` 屬性是最安全且最易維護的 session 管理方式
- Layout routes 提供宣告式的路由保護，支援巢狀授權層級
- Server function guards 是唯一真正的安全屏障——無論其他層級是否已有保護，皆不可省略

---

下一篇：[Server Functions 與認證](./03-server-functions.md)
