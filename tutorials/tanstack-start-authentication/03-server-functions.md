# 03：用於驗證的 Server Functions

> 學習如何使用 TanStack Start 的 server functions 在伺服器端安全地處理登入、登出與使用者資料取得。

## 什麼是 Server Functions？

Server functions 是**只在伺服器上執行**的函式，即使你是從 React 元件中呼叫它們。TanStack Start 讓這件事變得無縫銜接——你只需撰寫函式，框架會自動處理用戶端與伺服器之間的通訊。

這對驗證機制來說至關重要，原因如下：
- 密碼與憑證絕不能暴露給瀏覽器
- 資料庫查詢應該只在伺服器端執行
- Session 管理需要伺服器端才能存取 HTTP cookies

## 使用 `createServerFn` 建立 Server Functions

TanStack Start 提供了 `createServerFn` 來定義僅在伺服器執行的函式。以下是基本用法：

```tsx
import { createServerFn } from '@tanstack/react-start'

const myServerFunction = createServerFn({ method: 'POST' })
  .inputValidator((data: { someField: string }) => data)
  .handler(async ({ data }) => {
    // 這段程式碼只會在伺服器上執行
    // 可以安全地存取資料庫、密鑰等
    return { result: 'something' }
  })
```

讓我們拆解說明：
- **`method: 'POST'`** — 用戶端呼叫此函式時使用的 HTTP 方法（對於登入/登出等變更操作使用 POST，讀取操作使用 GET）
- **`.inputValidator()`** — 驗證並定義輸入資料的型別
- **`.handler()`** — 實際的伺服器端邏輯

## 登入 Server Function

以下是一個完整的登入函式：

```tsx
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'

export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    // 步驟 1：用資料庫驗證使用者的憑證
    const user = await authenticateUser(data.email, data.password)

    // 步驟 2：如果憑證錯誤，回傳錯誤訊息
    if (!user) {
      return { error: 'Invalid credentials' }
    }

    // 步驟 3：建立 session（將使用者資訊存入安全的 cookie）
    const session = await useAppSession()
    await session.update({
      userId: user.id,
      email: user.email,
    })

    // 步驟 4：重新導向到受保護的區域
    throw redirect({ to: '/dashboard' })
  })
```

**重點說明：**
- `authenticateUser` 是你自己撰寫的函式，用來查詢資料庫（我們會在後面的章節中建立）
- `useAppSession()` 讓你可以存取 session（下一節會詳細說明）
- `throw redirect(...)` 在登入成功後將使用者導向到新頁面——注意這裡使用的是 `throw`，而不是 `return`

## 登出 Server Function

登出比較簡單——只需清除 session：

```tsx
export const logoutFn = createServerFn({ method: 'POST' }).handler(
  async () => {
    // 清除所有 session 資料
    const session = await useAppSession()
    await session.clear()

    // 重新導向到首頁
    throw redirect({ to: '/' })
  }
)
```

不需要輸入驗證——登出時沒有什麼需要驗證的。

## 取得目前使用者的函式

要檢查某人是否已登入並取得他們的資訊：

```tsx
export const getCurrentUserFn = createServerFn({ method: 'GET' }).handler(
  async () => {
    // 讀取 session
    const session = await useAppSession()
    const userId = session.data.userId

    // 如果 session 中沒有 userId，表示使用者尚未登入
    if (!userId) {
      return null
    }

    // 從資料庫查詢完整的使用者記錄
    return await getUserById(userId)
  }
)
```

**為什麼使用 `method: 'GET'`？** 這個函式不會修改任何資料——它只是讀取資料。GET 適用於唯讀操作。

## 整體運作流程

```
使用者點擊「登入」
    ↓
React 元件呼叫 loginFn({ email, password })
    ↓
TanStack Start 向伺服器發送 POST 請求
    ↓
伺服器：驗證憑證 → 建立 session → 重新導向
    ↓
瀏覽器：收到重新導向 → 導覽至 /dashboard
    ↓
Dashboard 路由：呼叫 getCurrentUserFn() → 取得使用者資料 → 渲染頁面
```

## 重點整理

- **Server functions** 只在伺服器上執行，確保憑證與資料庫存取的安全性
- 使用 `createServerFn` 搭配 `method: 'POST'` 處理登入/登出（變更操作），搭配 `method: 'GET'` 讀取使用者資料
- `.inputValidator()` 提供型別安全的輸入驗證
- 使用 `throw redirect(...)` 在驗證動作完成後將使用者導向新頁面
- `useAppSession()` 提供對 session 儲存的存取（下一節會詳細說明）

---

下一篇：[Session 管理](./04-session-management.md)
