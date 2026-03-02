# 10：常見模式與正式環境注意事項

> 載入狀態、「記住我」功能、遷移建議，以及正式環境準備清單。

## 載入狀態

使用者在驗證操作進行時需要回饋。如果沒有載入狀態，介面會讓人覺得壞掉了。

### 帶有載入狀態的登入表單

```tsx
function LoginForm() {
  const [isLoading, setIsLoading] = useState(false)
  const loginMutation = useServerFn(loginFn)

  const handleSubmit = async (data: LoginData) => {
    setIsLoading(true)
    try {
      await loginMutation.mutate(data)
    } catch (error) {
      // 處理錯誤（向使用者顯示訊息）
    } finally {
      // 無論成功或失敗，都要重設載入狀態
      setIsLoading(false)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Email
        <input type="email" name="email" disabled={isLoading} />
      </label>

      <label>
        Password
        <input type="password" name="password" disabled={isLoading} />
      </label>

      <button disabled={isLoading}>
        {isLoading ? 'Logging in...' : 'Login'}
      </button>
    </form>
  )
}
```

**載入狀態的最佳實踐：**
- 載入時停用表單輸入欄位和送出按鈕
- 顯示明確的指示（文字變化、旋轉圖示等）
- 一律在 `finally` 中重設載入狀態（同時處理成功與錯誤情況）

## 「記住我」功能

「記住我」功能會延長 session 的有效時間，讓使用者不必頻繁重新登入。

```tsx
export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator(
    (data: { email: string; password: string; rememberMe?: boolean }) => data,
  )
  .handler(async ({ data }) => {
    const user = await authenticateUser(data.email, data.password)
    if (!user) return { error: 'Invalid credentials' }

    const session = await useAppSession()
    await session.update(
      { userId: user.id },
      {
        // 如果勾選「記住我」：30 天
        // 如果沒有勾選：session cookie（瀏覽器關閉時刪除）
        maxAge: data.rememberMe ? 30 * 24 * 60 * 60 : undefined,
      },
    )

    return { success: true }
  })
```

### 登入表單上的呈現

```tsx
<label>
  <input type="checkbox" name="rememberMe" />
  記住我 30 天
</label>
```

## 從其他方案遷移

### 從用戶端驗證遷移（localStorage、僅 Context）

如果你目前將驗證狀態存放在 `localStorage` 或 React Context 中，而沒有伺服器端驗證：

1. **將驗證邏輯移至 server function** — 憑證驗證、token 產生
2. **用伺服器 session 取代 localStorage** — HTTP-only cookie 更安全
3. **將路由保護改為使用 `beforeLoad`** — 取代用戶端重新導向邏輯
4. **加入 CSRF 防護** — 設定 `sameSite: 'lax'` 的 cookie 可以處理這個問題

### 從 Next.js 遷移

- 將 **API routes** 替換為 TanStack Start 的 **server function**
- 將 **NextAuth.js session** 遷移至 TanStack Start 的 `useSession`
- 將基於 **middleware** 的路由保護改為 `beforeLoad`

### 從 Remix 遷移

- 將 **loader 和 action** 轉換為 server function
- 調整 **session 模式**（Remix session → TanStack Start session）
- 更新**路由保護**模式

## 託管服務 vs. 自建方案決策指南

在自行建置驗證系統與使用託管服務之間做選擇：

### 選擇託管服務（Clerk、WorkOS）的時機：

- 你需要快速取得**企業級功能**（SSO、SCIM、合規性）
- 你想要**預建的 UI 元件**（登入表單、使用者管理）
- 你需要**託管的安全性更新**和監控
- 你的預算允許按使用者計費或訂閱制定價
- 你的團隊規模小，無法投入時間維護驗證系統

### 選擇自建方案的時機：

- 你需要對驗證流程和使用者資料有**完全的控制權**
- 你有**自訂商業邏輯**的需求
- 你想**避免廠商綁定**
- 你的團隊有能力維護和監控驗證系統
- **成本控管**很重要（沒有按使用者計費）

## 正式環境檢查清單

在將驗證系統部署到正式環境之前，請確認以下項目：

### 安全性
- [ ] 密碼使用 bcrypt/scrypt/argon2 雜湊處理（12+ 輪 salt）
- [ ] Session secret 為 32 個以上的隨機字元，存放在環境變數中
- [ ] Cookie 設定為 `secure: true`、`sameSite: 'lax'`、`httpOnly: true`
- [ ] 啟用 HTTPS（正式環境不可妥協）
- [ ] 對登入端點實施速率限制
- [ ] 所有 server function 都有輸入驗證（使用 Zod）
- [ ] 使用通用的錯誤訊息（不要透露 email 是否已存在）

### 功能性
- [ ] 登入、登出和註冊都能正常運作
- [ ] 受保護的路由會將未驗證的使用者重新導向
- [ ] Session 在頁面重新載入後仍然有效
- [ ] Session 在設定的 `maxAge` 後過期
- [ ] 密碼重設流程能完整走完

### 監控
- [ ] 記錄驗證事件（登入、登出、失敗嘗試）
- [ ] 監控異常模式（來自同一 IP 的大量登入失敗）
- [ ] 設定驗證系統錯誤的警報

### 合規性
- [ ] 如適用，依照 GDPR/CCPA 處理個人資料
- [ ] 如有需要，提供帳號刪除功能
- [ ] 記錄你的資料保留政策

## 實作範例

請參考以下 TanStack 官方範例作為參考實作：

- **[Basic Auth with Prisma](https://github.com/TanStack/router/tree/main/examples/react/start-basic-auth)** — 完整的自建實作，包含資料庫和 session
- **[Supabase Integration](https://github.com/TanStack/router/tree/main/examples/react/start-supabase-basic)** — 第三方服務整合
- **[Clerk Integration](https://github.com/TanStack/router/tree/main/examples/react/start-clerk-basic)** — 合作夥伴方案，附帶預建 UI
- **[WorkOS Integration](https://github.com/TanStack/router/tree/main/examples/react/start-workos)** — 企業級驗證
- **[Client-side Context Auth](https://github.com/TanStack/router/tree/main/examples/react/authenticated-routes)** — 僅用戶端的模式

## 重點整理

- 驗證操作期間一律顯示載入狀態 — 停用表單並顯示回饋
- 「記住我」只是將 session cookie 的 `maxAge` 設定得更長
- 從其他框架遷移主要是將邏輯移至 server function
- 需要速度和企業功能時選擇託管驗證；需要控制權和節省成本時選擇自建方案
- 部署前使用正式環境檢查清單 — 涵蓋安全性、功能性和合規性
- 研究官方範例以了解完整且可運作的實作

---

本篇是 TanStack Start 驗證教學的最後一章！回到[簡介](./01-introduction.md)以查看完整的目錄。
