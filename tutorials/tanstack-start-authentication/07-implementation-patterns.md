# 07：實作模式

> 電子郵件／密碼驗證、角色型存取控制、社群登入，以及密碼重設流程的實務實作。

## 基本電子郵件／密碼驗證

這是最常見的驗證模式。讓我們一步一步來實作。

### 使用者註冊

```tsx
// server/auth.ts
import bcrypt from 'bcryptjs'
import { createServerFn } from '@tanstack/react-start'

export const registerFn = createServerFn({ method: 'POST' })
  .inputValidator(
    (data: { email: string; password: string; name: string }) => data,
  )
  .handler(async ({ data }) => {
    // 步驟 1：檢查是否已有使用者使用此電子郵件
    const existingUser = await getUserByEmail(data.email)
    if (existingUser) {
      return { error: 'User already exists' }
    }

    // 步驟 2：儲存之前先將密碼雜湊處理
    // 絕對不要儲存明文密碼！
    // 「12」是 salt rounds — 數值越高越安全，但速度越慢
    const hashedPassword = await bcrypt.hash(data.password, 12)

    // 步驟 3：將使用者儲存至資料庫
    const user = await createUser({
      email: data.email,
      password: hashedPassword,
      name: data.name,
    })

    // 步驟 4：建立 session 讓使用者自動登入
    const session = await useAppSession()
    await session.update({ userId: user.id })

    return { success: true, user: { id: user.id, email: user.email } }
  })
```

**為什麼使用 `bcrypt`？**
- 它是單向雜湊 — 無法反向還原出原始密碼
- Salt rounds（12）使暴力破解攻擊變得極為緩慢
- 每組密碼都會產生獨立的隨機 salt，因此相同的密碼會產生不同的雜湊值

### 使用者登入（憑證驗證）

```tsx
async function authenticateUser(email: string, password: string) {
  // 透過電子郵件查詢使用者
  const user = await getUserByEmail(email)
  if (!user) return null

  // 將輸入的密碼與儲存的雜湊值進行比對
  const isValid = await bcrypt.compare(password, user.password)
  return isValid ? user : null
}
```

`bcrypt.compare` 會自動處理 salt — 你不需要額外擷取或儲存它。

## 角色型存取控制（RBAC）

RBAC 讓你可以控制不同類型的使用者能執行哪些操作。

### 定義角色與權限

```tsx
// utils/auth.ts

// 將角色定義為常數
export const roles = {
  USER: 'user',
  ADMIN: 'admin',
  MODERATOR: 'moderator',
} as const

type Role = (typeof roles)[keyof typeof roles]

// 檢查使用者的角色是否符合所需等級
export function hasPermission(userRole: Role, requiredRole: Role): boolean {
  // 定義階層：數值越高 = 權限越大
  const hierarchy = {
    [roles.USER]: 0,
    [roles.MODERATOR]: 1,
    [roles.ADMIN]: 2,
  }

  return hierarchy[userRole] >= hierarchy[requiredRole]
}
```

### 在路由保護中使用 RBAC

```tsx
// routes/_authed/admin/index.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'
import { hasPermission, roles } from '../../utils/auth'

export const Route = createFileRoute('/_authed/admin/')({
  beforeLoad: async ({ context }) => {
    // context.user 來自 _authed 佈局
    if (!hasPermission(context.user.role, roles.ADMIN)) {
      throw redirect({ to: '/unauthorized' })
    }
  },
})
```

## 社群驗證（OAuth）

OAuth 讓使用者可以使用既有帳號（Google、GitHub 等）登入。

### 設定 OAuth 提供者

```tsx
// server/oauth.ts
export const authProviders = {
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID!,
    redirectUri: `${process.env.APP_URL}/auth/google/callback`,
  },
  github: {
    clientId: process.env.GITHUB_CLIENT_ID!,
    redirectUri: `${process.env.APP_URL}/auth/github/callback`,
  },
}
```

### 啟動 OAuth 流程

```tsx
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'

export const initiateOAuthFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { provider: 'google' | 'github' }) => data)
  .handler(async ({ data }) => {
    const provider = authProviders[data.provider]

    // 產生隨機 state 參數以防範 CSRF 攻擊
    const state = generateRandomState()

    // 將 state 儲存在 session 中，以便稍後驗證
    const session = await useAppSession()
    await session.update({ oauthState: state })

    // 建構 OAuth 授權 URL 並將使用者重新導向
    const authUrl = generateOAuthUrl(provider, state)
    throw redirect({ href: authUrl })
  })
```

**OAuth 流程簡述：**
1. 你的應用程式將使用者重新導向至 Google／GitHub 的登入頁面
2. 使用者在該頁面登入並授予權限
3. Google／GitHub 將使用者重新導向回你的 callback URL，並附帶一組 code
4. 你的伺服器用該 code 換取使用者資訊
5. 你在資料庫中建立或更新使用者，並啟動一個 session

### 為什麼需要 `state` 參數？

`state` 是你在重新導向之前產生的一組隨機字串。當 OAuth 提供者重新導向回來時，你要驗證 `state` 是否與先前儲存的相符。這可以防止 CSRF 攻擊 — 也就是有人可能誘使使用者以攻擊者的身分登入。

## 密碼重設流程

安全的密碼重設包含兩個步驟：請求重設與確認重設。

### 步驟 1：請求密碼重設

```tsx
export const requestPasswordResetFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string }) => data)
  .handler(async ({ data }) => {
    const user = await getUserByEmail(data.email)
    if (!user) {
      // 重要：不要透露該電子郵件是否存在！
      // 始終回傳相同的回應，以防止電子郵件列舉攻擊
      return { success: true }
    }

    // 產生安全的隨機 token
    const token = generateSecureToken()

    // Token 在 1 小時後過期
    const expires = new Date(Date.now() + 60 * 60 * 1000)

    // 將 token 儲存至資料庫
    await savePasswordResetToken(user.id, token, expires)

    // 寄送包含 token 連結的重設密碼電子郵件
    await sendPasswordResetEmail(user.email, token)

    return { success: true }
  })
```

### 步驟 2：重設密碼

```tsx
export const resetPasswordFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { token: string; newPassword: string }) => data)
  .handler(async ({ data }) => {
    // 查詢 token
    const resetToken = await getPasswordResetToken(data.token)

    // 檢查 token 是否有效且尚未過期
    if (!resetToken || resetToken.expires < new Date()) {
      return { error: 'Invalid or expired token' }
    }

    // 將新密碼雜湊處理後更新使用者資料
    const hashedPassword = await bcrypt.hash(data.newPassword, 12)
    await updateUserPassword(resetToken.userId, hashedPassword)

    // 刪除已使用的 token，使其無法被重複使用
    await deletePasswordResetToken(data.token)

    return { success: true }
  })
```

**安全注意事項：**
- Token 應使用密碼學安全的隨機值（使用 `crypto.randomBytes`）
- 務必設定過期時間（1 小時是常見做法）
- 使用後刪除 token
- 絕對不要在錯誤訊息中透露電子郵件是否存在於你的系統中

## 重點整理

- 務必使用 `bcrypt` 進行密碼雜湊（salt rounds 設為 12 是不錯的預設值）
- RBAC 使用角色階層來檢查權限 — 簡單且有效
- OAuth 需要透過 `state` 參數來防範 CSRF 攻擊
- 密碼重設 token 應該是隨機的、有時間限制的，且只能使用一次
- 絕對不要在錯誤訊息中透露電子郵件是否存在

---

下一篇：[安全性最佳實務](./08-security-best-practices.md)
