# 09：測試認證系統

> 如何為你的認證系統撰寫單元測試與整合測試。

## 為什麼要測試認證？

認證是關鍵的基礎設施——一旦出問題，使用者可能無法登入，更糟的是，未經授權的使用者可能獲得存取權限。測試能確保：

- 使用有效憑證時登入正常運作
- 使用無效憑證時登入會被拒絕
- 受保護的路由會將未認證的使用者重新導向
- Session 管理正確運作
- 邊界情況能被妥善處理（過期的 token、重複的電子郵件等）

## 單元測試伺服器函式

單元測試用於驗證個別邏輯片段是否能獨立正確運作。

### 使用 Vitest 進行設定

```tsx
// __tests__/auth.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { loginFn } from '../server/auth'

describe('Authentication', () => {
  // 在每個測試前重設測試資料庫
  beforeEach(async () => {
    await setupTestDatabase()
  })

  it('should login with valid credentials', async () => {
    // 準備：先建立一個測試使用者
    // （setupTestDatabase 應該會初始化種子資料）

    // 執行：嘗試登入
    const result = await loginFn({
      data: { email: 'test@example.com', password: 'password123' },
    })

    // 驗證：沒有錯誤，回傳使用者資料
    expect(result.error).toBeUndefined()
    expect(result.user).toBeDefined()
  })

  it('should reject invalid credentials', async () => {
    const result = await loginFn({
      data: { email: 'test@example.com', password: 'wrongpassword' },
    })

    expect(result.error).toBe('Invalid credentials')
  })
})
```

### 單元測試中應測試的項目

| 測試案例 | 驗證內容 |
|-----------|----------------------|
| 有效登入 | 正確的憑證回傳使用者資料 |
| 無效密碼 | 錯誤的密碼回傳錯誤訊息 |
| 不存在的使用者 | 不明的電子郵件回傳錯誤訊息 |
| 註冊 | 新使用者被建立且 session 已啟動 |
| 重複註冊 | 已存在的電子郵件回傳錯誤訊息 |
| 密碼雜湊 | 密碼經過雜湊處理，而非明文儲存 |
| Session 建立 | 登入後設定 session 資料 |
| Session 清除 | 登出後清除 session 資料 |

## 整合測試

整合測試驗證系統的多個部分是否能正確協同運作——包括路由、認證檢查、重新導向與畫面渲染。

### 使用 React Testing Library 測試認證流程

```tsx
// __tests__/auth-flow.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import { RouterProvider, createMemoryHistory } from '@tanstack/react-router'
import { router } from '../router'

describe('Authentication Flow', () => {
  it('should redirect to login when accessing protected route', async () => {
    // 建立一個從受保護路由開始的記憶體內歷史紀錄
    const history = createMemoryHistory()
    history.push('/dashboard')  // 這是一個受保護的路由

    // 渲染應用程式
    render(<RouterProvider router={router} history={history} />)

    // 使用者應該被重新導向到登入頁面
    await waitFor(() => {
      expect(screen.getByText('Login')).toBeInTheDocument()
    })
  })

  it('should show dashboard after successful login', async () => {
    const history = createMemoryHistory()
    history.push('/login')

    render(<RouterProvider router={router} history={history} />)

    // 填寫並送出登入表單
    fireEvent.change(screen.getByLabelText('Email'), {
      target: { value: 'test@example.com' },
    })
    fireEvent.change(screen.getByLabelText('Password'), {
      target: { value: 'password123' },
    })
    fireEvent.click(screen.getByText('Login'))

    // 登入後，應重新導向到儀表板
    await waitFor(() => {
      expect(screen.getByText('Welcome')).toBeInTheDocument()
    })
  })
})
```

### 整合測試中應測試的項目

| 測試案例 | 驗證內容 |
|-----------|----------------------|
| 受保護路由的重新導向 | 未認證 → 登入頁面 |
| 登入 → 重新導向 | 登入成功 → 受保護頁面 |
| 登出 → 重新導向 | 登出 → 公開頁面 |
| 角色型重新導向 | 非管理員 → 未授權頁面 |
| 跨頁面瀏覽時保持狀態 | Session 在頁面切換間維持有效 |

## 測試技巧

### 1. 使用測試資料庫

絕對不要對正式環境或開發環境的資料庫執行測試。請使用記憶體內資料庫（例如 SQLite）或獨立的測試資料庫。

### 2. 初始化種子測試資料

建立輔助函式來設定已知的測試使用者：

```tsx
async function setupTestDatabase() {
  // 清除所有資料
  await db.user.deleteMany()

  // 建立一個具有已知密碼的測試使用者
  await db.user.create({
    data: {
      email: 'test@example.com',
      password: await bcrypt.hash('password123', 12),
      name: 'Test User',
      role: 'user',
    },
  })
}
```

### 3. 模擬外部服務

如果你使用 OAuth 供應商，在測試中進行模擬：

```tsx
// 模擬 OAuth token 交換
vi.mock('../server/oauth', () => ({
  exchangeCodeForToken: vi.fn().mockResolvedValue({
    access_token: 'mock-token',
    user: { email: 'oauth@example.com' },
  }),
}))
```

### 4. 測試邊界情況

不要只測試正常流程。也要測試：
- 過期的 session
- 格式錯誤的輸入
- 同時發生的多次登入嘗試
- 認證過程中的網路錯誤

## 重點整理

- 單元測試驗證個別認證函式（登入、註冊、session 管理）
- 整合測試驗證完整流程（導覽 → 重新導向 → 登入 → 存取）
- 務必使用獨立的測試資料庫並初始化種子測試資料
- 模擬外部服務（OAuth 供應商、電子郵件寄送）
- 同時測試正常流程與邊界情況（過期的 token、無效的輸入）
- 使用 `vitest` + `@testing-library/react` 作為你的測試技術堆疊

---

下一篇：[常見模式與正式環境部署](./10-common-patterns-and-production.md)
