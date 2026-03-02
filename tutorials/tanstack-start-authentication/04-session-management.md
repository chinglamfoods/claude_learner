# 04：Session 管理

> 學習如何使用 TanStack Start 內建的 session 管理機制，搭配 HTTP-only cookie 來設定安全的 session。

## 什麼是 Session？

**Session**（工作階段）是伺服器用來在多次請求之間「記住」使用者身份的機制。流程如下：

1. 使用者成功登入
2. 伺服器建立一個 session，並將它儲存在 **HTTP-only cookie** 中
3. 瀏覽器在後續的每一次請求中自動附帶這個 cookie
4. 伺服器讀取 cookie 來辨識使用者身份

可以把它想像成遊樂園的手環——進場時（登入）在入口處領到手環後，就可以自由進出園區內的各個設施（頁面），不用每次都重新出示門票。

## 使用 `useSession` 設定 Session

TanStack Start 提供了一個 `useSession` 函式，來自 `@tanstack/react-start/server`。以下是設定方式：

```tsx
// utils/session.ts
import { useSession } from '@tanstack/react-start/server'

// 定義 session 資料的型別
type SessionData = {
  userId?: string
  email?: string
  role?: string
}

// 建立一個可重複使用的函式來存取 session
export function useAppSession() {
  return useSession<SessionData>({
    // cookie 的名稱
    name: 'app-session',

    // 用來加密 cookie 的密鑰（至少 32 個字元！）
    password: process.env.SESSION_SECRET!,

    // Cookie 安全性設定
    cookie: {
      secure: process.env.NODE_ENV === 'production', // 正式環境僅允許 HTTPS
      sameSite: 'lax',    // 防範 CSRF 攻擊
      httpOnly: true,      // JavaScript 無法讀取（防範 XSS 攻擊）
    },
  })
}
```

讓我們逐一說明每個設定：

### `name`
儲存在瀏覽器中的 cookie 名稱。你可以自由命名——`'app-session'` 是常見的選擇。

### `password`
一組用來**加密** session 資料的密鑰字串。這確保即使 cookie 儲存在瀏覽器中，其內容也無法被讀取或竄改。

> **重要：**密鑰長度必須至少 32 個字元。請將它存放在環境變數中，千萬不要直接寫在程式碼裡！

範例 `.env` 檔案：
```
SESSION_SECRET=at-least-32-characters-long-random-string-here
```

### `cookie` 選項

| 選項 | 說明 |
|------|------|
| `secure: true` | Cookie 僅透過 HTTPS 傳送（正式環境設為 `true`） |
| `sameSite: 'lax'` | 瀏覽器僅在同站請求與頂層導覽時傳送 cookie——防範 CSRF 攻擊 |
| `httpOnly: true` | JavaScript 無法存取 cookie（`document.cookie`）——防範 XSS 攻擊 |

## 讀取 Session 資料

一旦 session 存在，你可以在任何伺服器函式中讀取它：

```tsx
const session = await useAppSession()

// 存取 session 資料
const userId = session.data.userId    // string | undefined
const email = session.data.email      // string | undefined
const role = session.data.role        // string | undefined
```

## 寫入 Session 資料

要將資料存入 session（例如登入後）：

```tsx
const session = await useAppSession()

// 將使用者資訊寫入 session
await session.update({
  userId: user.id,
  email: user.email,
  role: user.role,
})
```

`update` 方法會用你提供的資料取代現有的 session 內容。加密後的 cookie 會自動回傳給瀏覽器。

## 清除 Session

要讓使用者登出，請清除 session：

```tsx
const session = await useAppSession()
await session.clear()
```

這會移除所有 session 資料，等同於將使用者登出。

## Session 過期時間

你可以透過 `maxAge` 來控制 session 的有效期間：

```tsx
export function useAppSession() {
  return useSession<SessionData>({
    name: 'app-session',
    password: process.env.SESSION_SECRET!,
    cookie: {
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      httpOnly: true,
      maxAge: 7 * 24 * 60 * 60, // 7 天（以秒為單位）
    },
  })
}
```

如果未設定 `maxAge`，cookie 將會是一個 **session cookie**——當使用者關閉瀏覽器時就會被刪除。

## 完整範例：Session 實際運作

以下展示從登入到存取受保護頁面的完整流程：

```tsx
// server/auth.ts
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'
import { useAppSession } from '../utils/session'

// 登入：建立 session
export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    const user = await authenticateUser(data.email, data.password)
    if (!user) return { error: 'Invalid credentials' }

    const session = await useAppSession()
    await session.update({ userId: user.id, email: user.email })

    throw redirect({ to: '/dashboard' })
  })

// 驗證身份：讀取 session
export const getCurrentUserFn = createServerFn({ method: 'GET' }).handler(
  async () => {
    const session = await useAppSession()
    if (!session.data.userId) return null
    return await getUserById(session.data.userId)
  }
)

// 登出：清除 session
export const logoutFn = createServerFn({ method: 'POST' }).handler(
  async () => {
    const session = await useAppSession()
    await session.clear()
    throw redirect({ to: '/' })
  }
)
```

## 重點整理

- Session 讓伺服器能在多次請求之間記住使用者身份
- TanStack Start 使用加密的 HTTP-only cookie——預設就是安全的
- `useSession`（來自 `@tanstack/react-start/server`）是核心 API
- `SESSION_SECRET` 務必存放在環境變數中（至少 32 個字元）
- 使用 `session.update()` 寫入資料、`session.data` 讀取資料、`session.clear()` 登出
- 正確設定 `cookie` 選項（`secure`、`sameSite`、`httpOnly`）以確保安全性

---

下一篇：[Authentication Context](./05-authentication-context.md)
