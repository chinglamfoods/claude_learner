# 07：實作模式

> 涵蓋憑證驗證、RBAC、OAuth 整合與密碼重設的生產級實作，包含安全性邊界情境處理與業界慣用模式。

## 憑證驗證（Email / Password）

### 使用者註冊

密碼雜湊建議採用 bcrypt（cost factor 12）或 argon2id。bcrypt 久經考驗、生態系支援廣泛；argon2id 則為 OWASP 目前推薦的首選，具備記憶體硬化（memory-hard）特性，能有效抵禦 GPU/ASIC 暴力破解。若專案無特殊限制，優先考慮 argon2id。

```tsx
// server/auth.ts
import bcrypt from 'bcryptjs'
import { createServerFn } from '@tanstack/react-start'
import { z } from 'zod'

const registerSchema = z.object({
  email: z.string().email().max(255).toLowerCase(),
  password: z
    .string()
    .min(8)
    .max(128)
    .regex(/[A-Z]/, 'Must contain uppercase')
    .regex(/[0-9]/, 'Must contain digit'),
  name: z.string().min(1).max(100).trim(),
})

export const registerFn = createServerFn({ method: 'POST' })
  .inputValidator((data: unknown) => registerSchema.parse(data))
  .handler(async ({ data }) => {
    const existingUser = await getUserByEmail(data.email)

    // 無論使用者是否存在，皆回傳一致的成功回應，
    // 防止攻擊者透過差異化回應進行 email enumeration。
    if (existingUser) {
      return { success: true }
    }

    const hashedPassword = await bcrypt.hash(data.password, 12)

    const user = await createUser({
      email: data.email,
      password: hashedPassword,
      name: data.name,
    })

    const session = await useAppSession()
    await session.update({ userId: user.id })

    return { success: true }
  })
```

**Email enumeration 防護要點：** 註冊與登入端點都不應透過錯誤訊息揭露特定 email 是否已存在。上例中，即使 email 已被註冊，仍回傳 `{ success: true }`。實務上可搭配寄送通知信（「若此 email 已有帳號，請嘗試登入」）來維持使用者體驗。

### 使用者登入

```tsx
async function authenticateUser(
  email: string,
  password: string,
): Promise<User | null> {
  const user = await getUserByEmail(email.toLowerCase())

  // 即使使用者不存在，仍執行一次 bcrypt.compare 以維持固定的回應時間，
  // 避免 timing attack 洩漏使用者是否存在。
  if (!user) {
    await bcrypt.compare(password, DUMMY_HASH)
    return null
  }

  const isValid = await bcrypt.compare(password, user.password)
  return isValid ? user : null
}

// 預先計算的假雜湊值，用於保持回應時間一致
const DUMMY_HASH = await bcrypt.hash('dummy', 12)
```

### bcrypt vs argon2 取捨

| 面向 | bcrypt | argon2id |
|------|--------|----------|
| 成熟度 | 1999 年至今，廣泛部署 | 2015 PHC 冠軍，OWASP 推薦 |
| 抗 GPU/ASIC | 中等 | 優異（memory-hard） |
| 可調參數 | cost factor | 時間、記憶體、並行度 |
| Node.js 生態系 | `bcryptjs`（pure JS）、`bcrypt`（native） | `argon2`（需 native binding） |
| 部署考量 | 無 native dependency 選項 | Serverless 環境可能需要額外處理 native module |

在 serverless 或 edge runtime 環境中，`bcryptjs`（pure JS）部署最為簡便。若 runtime 支援 native module，argon2id 為安全性更強的選擇。

## 角色型存取控制（RBAC）

### 定義角色階層與權限

```tsx
// utils/auth.ts
export const Role = {
  USER: 'user',
  MODERATOR: 'moderator',
  ADMIN: 'admin',
} as const

export type Role = (typeof Role)[keyof typeof Role]

const ROLE_HIERARCHY: Record<Role, number> = {
  [Role.USER]: 0,
  [Role.MODERATOR]: 1,
  [Role.ADMIN]: 2,
} as const

export function hasMinRole(userRole: Role, requiredRole: Role): boolean {
  return ROLE_HIERARCHY[userRole] >= ROLE_HIERARCHY[requiredRole]
}

// 細粒度權限檢查（適用於需要超越單純階層的場景）
export type Permission = 'posts:delete' | 'users:ban' | 'settings:edit'

const ROLE_PERMISSIONS: Record<Role, ReadonlySet<Permission>> = {
  [Role.USER]: new Set([]),
  [Role.MODERATOR]: new Set(['posts:delete', 'users:ban']),
  [Role.ADMIN]: new Set(['posts:delete', 'users:ban', 'settings:edit']),
}

export function hasPermission(userRole: Role, permission: Permission): boolean {
  return ROLE_PERMISSIONS[userRole].has(permission)
}
```

### 在路由保護中套用 RBAC

```tsx
// routes/_authed/admin/index.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'
import { hasMinRole, Role } from '../../utils/auth'

export const Route = createFileRoute('/_authed/admin/')({
  beforeLoad: async ({ context }) => {
    if (!hasMinRole(context.user.role, Role.ADMIN)) {
      throw redirect({ to: '/unauthorized' })
    }
  },
})
```

若需同時支援階層式角色與細粒度權限，可將兩者組合：路由層級使用 `hasMinRole`，個別操作使用 `hasPermission`。

## OAuth 整合

### Provider 設定與 PKCE

對於 public client（SPA、mobile app），務必採用 PKCE（Proof Key for Code Exchange）。即使是 confidential client（有 server-side secret），啟用 PKCE 仍為 OAuth 2.1 的建議做法。

```tsx
// server/oauth.ts
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'
import crypto from 'node:crypto'

interface OAuthProviderConfig {
  clientId: string
  clientSecret: string
  authorizeUrl: string
  tokenUrl: string
  redirectUri: string
  scopes: string[]
}

const providers: Record<string, OAuthProviderConfig> = {
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID!,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    authorizeUrl: 'https://accounts.google.com/o/oauth2/v2/auth',
    tokenUrl: 'https://oauth2.googleapis.com/token',
    redirectUri: `${process.env.APP_URL}/auth/google/callback`,
    scopes: ['openid', 'email', 'profile'],
  },
  github: {
    clientId: process.env.GITHUB_CLIENT_ID!,
    clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    authorizeUrl: 'https://github.com/login/oauth/authorize',
    tokenUrl: 'https://github.com/login/oauth/access_token',
    redirectUri: `${process.env.APP_URL}/auth/github/callback`,
    scopes: ['read:user', 'user:email'],
  },
}

// PKCE: 產生 code_verifier 與 code_challenge
function generatePKCE(): { verifier: string; challenge: string } {
  const verifier = crypto.randomBytes(32).toString('base64url')
  const challenge = crypto
    .createHash('sha256')
    .update(verifier)
    .digest('base64url')
  return { verifier, challenge }
}

export const initiateOAuthFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { provider: string }) => {
    if (!providers[data.provider]) {
      throw new Error(`Unsupported provider: ${data.provider}`)
    }
    return data as { provider: keyof typeof providers }
  })
  .handler(async ({ data }) => {
    const provider = providers[data.provider]
    const state = crypto.randomBytes(32).toString('base64url')
    const { verifier, challenge } = generatePKCE()

    const session = await useAppSession()
    await session.update({
      oauthState: state,
      oauthCodeVerifier: verifier,
      oauthProvider: data.provider,
    })

    const params = new URLSearchParams({
      client_id: provider.clientId,
      redirect_uri: provider.redirectUri,
      response_type: 'code',
      scope: provider.scopes.join(' '),
      state,
      code_challenge: challenge,
      code_challenge_method: 'S256',
    })

    throw redirect({ href: `${provider.authorizeUrl}?${params}` })
  })
```

### OAuth Callback 與帳號連結

```tsx
export const handleOAuthCallbackFn = createServerFn({ method: 'GET' })
  .inputValidator((data: { code: string; state: string }) => data)
  .handler(async ({ data }) => {
    const session = await useAppSession()
    const { oauthState, oauthCodeVerifier, oauthProvider } = session.data

    // 驗證 state 參數以防禦 CSRF
    if (!oauthState || data.state !== oauthState) {
      throw new Error('Invalid OAuth state — potential CSRF attack')
    }

    // 清除已使用的 state，避免 replay
    await session.update({
      oauthState: null,
      oauthCodeVerifier: null,
      oauthProvider: null,
    })

    const provider = providers[oauthProvider!]

    // 以 authorization code + code_verifier 換取 token
    let tokenResponse: Response
    try {
      tokenResponse = await fetch(provider.tokenUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: new URLSearchParams({
          grant_type: 'authorization_code',
          code: data.code,
          redirect_uri: provider.redirectUri,
          client_id: provider.clientId,
          client_secret: provider.clientSecret,
          code_verifier: oauthCodeVerifier!,
        }),
      })
    } catch {
      // OAuth provider 無法連線（downtime、DNS 失敗等）
      throw redirect({ to: '/auth/error?reason=provider_unavailable' })
    }

    if (!tokenResponse.ok) {
      throw redirect({ to: '/auth/error?reason=token_exchange_failed' })
    }

    const tokens = await tokenResponse.json()
    const profile = await fetchUserProfile(oauthProvider!, tokens.access_token)

    // 帳號連結策略：相同 email 來自不同 OAuth provider
    const existingUser = await getUserByEmail(profile.email)

    if (existingUser) {
      // 將新的 OAuth provider 連結至既有帳號
      await linkOAuthAccount({
        userId: existingUser.id,
        provider: oauthProvider!,
        providerAccountId: profile.id,
      })
      await session.update({ userId: existingUser.id })
    } else {
      const newUser = await createUser({
        email: profile.email,
        name: profile.name,
        emailVerified: true, // OAuth provider 已驗證 email
      })
      await linkOAuthAccount({
        userId: newUser.id,
        provider: oauthProvider!,
        providerAccountId: profile.id,
      })
      await session.update({ userId: newUser.id })
    }

    throw redirect({ to: '/' })
  })
```

**OAuth state 參數 CSRF 風險：** 若省略 `state` 驗證，攻擊者可建構惡意 callback URL，誘導受害者完成 OAuth 流程後將攻擊者的帳號綁定至受害者的 session。`state` 必須為 cryptographically random、存放於 server-side session，並在 callback 時嚴格比對後立即銷毀。

**帳號連結注意事項：** 當不同 OAuth provider 回傳相同 email 時，自動連結帳號是常見做法，但需注意：僅在 OAuth provider 確認 email 已驗證（`email_verified: true`）時才執行自動連結，否則攻擊者可在第三方 provider 註冊未驗證的 email 來劫持帳號。

**Provider downtime 處理：** 對 token endpoint 的請求應設定合理 timeout（建議 5–10 秒），並在失敗時導向統一的錯誤頁面，附帶可識別的錯誤代碼供使用者回報。

## 密碼重設流程

### 請求重設

```tsx
import crypto from 'node:crypto'

export const requestPasswordResetFn = createServerFn({ method: 'POST' })
  .inputValidator((data: unknown) =>
    z.object({ email: z.string().email().toLowerCase() }).parse(data),
  )
  .handler(async ({ data }) => {
    const user = await getUserByEmail(data.email)

    // 無論使用者是否存在，回傳一致的回應（防止 email enumeration）
    if (!user) {
      return { success: true }
    }

    // 使 token 為 cryptographically random，並儲存其雜湊值而非明文
    const rawToken = crypto.randomBytes(32).toString('base64url')
    const hashedToken = crypto
      .createHash('sha256')
      .update(rawToken)
      .digest('hex')

    const expiresAt = new Date(Date.now() + 60 * 60 * 1000) // 1 小時

    // 存入 DB 的是雜湊後的 token；寄給使用者的是原始 token
    await savePasswordResetToken({
      userId: user.id,
      tokenHash: hashedToken,
      expiresAt,
    })

    await sendPasswordResetEmail(user.email, rawToken)

    return { success: true }
  })
```

### 執行重設

```tsx
export const resetPasswordFn = createServerFn({ method: 'POST' })
  .inputValidator((data: unknown) =>
    z
      .object({
        token: z.string().min(1),
        newPassword: z.string().min(8).max(128),
      })
      .parse(data),
  )
  .handler(async ({ data }) => {
    // 將使用者提供的 token 雜湊後與 DB 中的雜湊值比對
    const hashedToken = crypto
      .createHash('sha256')
      .update(data.token)
      .digest('hex')

    const resetRecord = await getPasswordResetToken(hashedToken)

    if (!resetRecord || resetRecord.expiresAt < new Date()) {
      return { error: 'Invalid or expired token' }
    }

    const hashedPassword = await bcrypt.hash(data.newPassword, 12)
    await updateUserPassword(resetRecord.userId, hashedPassword)

    // 刪除 token 確保一次性使用，並撤銷該使用者所有既有 session
    await deletePasswordResetToken(hashedToken)
    await invalidateAllSessions(resetRecord.userId)

    return { success: true }
  })
```

**Token 儲存陷阱：** 重設 token 切勿以明文存入資料庫。若資料庫遭入侵，攻擊者可直接使用明文 token 重設任何使用者密碼。正確做法：將 SHA-256 雜湊值存入 DB，寄出原始 token，驗證時將使用者提供的 token 雜湊後比對。

**Token replay 防護：** 除了使用後刪除 token 外，密碼重設成功後應同時撤銷該使用者所有既有 session，強制重新登入，以確保先前可能被盜用的 session 失效。

## Refresh Token Rotation

對於需要長效登入的應用，refresh token rotation 是防止 token 被盜用後無限期存取的關鍵模式。

```tsx
interface TokenPair {
  accessToken: string  // 短效，例如 15 分鐘
  refreshToken: string // 長效，例如 7 天，每次使用後輪換
}

async function rotateRefreshToken(
  currentRefreshToken: string,
): Promise<TokenPair | null> {
  const stored = await getRefreshToken(currentRefreshToken)

  if (!stored || stored.expiresAt < new Date()) {
    return null
  }

  // 偵測 token reuse：若該 token 已被標記為已使用，
  // 代表可能發生 token theft，應撤銷整個 token family。
  if (stored.usedAt) {
    await revokeTokenFamily(stored.familyId)
    return null
  }

  // 標記目前 token 為已使用
  await markRefreshTokenUsed(currentRefreshToken)

  // 產生新的 token pair，保留相同的 family ID
  const newRefreshToken = crypto.randomBytes(32).toString('base64url')
  const newAccessToken = generateAccessToken(stored.userId)

  await saveRefreshToken({
    token: newRefreshToken,
    userId: stored.userId,
    familyId: stored.familyId,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
  })

  return { accessToken: newAccessToken, refreshToken: newRefreshToken }
}
```

**Rotation 的核心價值：** 每次 refresh 時舊 token 即失效。若攻擊者竊取了 refresh token 並嘗試使用，而正常使用者也在使用同一 token，系統會偵測到同一 token 被使用兩次（reuse detection），觸發整個 token family 撤銷，強制所有裝置重新登入。

## 重點整理

- 密碼雜湊優先採用 argon2id；若部署環境受限（serverless / edge），bcrypt（cost 12）為穩健替代方案
- 註冊、登入、密碼重設端點皆須防範 email enumeration，回應保持一致
- 登入驗證加入 timing attack 防護：使用者不存在時仍執行一次雜湊比對
- OAuth 必須驗證 `state` 參數防禦 CSRF，並採用 PKCE 保護 authorization code
- 帳號連結僅在 OAuth provider 確認 email 已驗證時才自動執行
- 密碼重設 token 以雜湊形式儲存於 DB，使用後刪除並撤銷所有既有 session
- Refresh token rotation 搭配 reuse detection 可有效偵測 token theft
- 對 OAuth provider 的外部請求須設定 timeout 並妥善處理 downtime 場景

---

下一篇：[安全性最佳實務](./08-security-best-practices.md)
