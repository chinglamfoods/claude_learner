# 10：常見模式與正式環境注意事項

> 非同步操作的 UX 處理、Optimistic updates、「記住我」的安全考量、跨框架遷移策略、託管 vs. 自建的成本分析，以及完整的 production 部署檢查清單。

## 驗證操作的 UX 模式

### 表單提交與 Mutation 狀態

登入、註冊、登出等驗證操作皆為非同步 mutation。表單需正確反映 pending、success、error 三種狀態，並防止重複提交。

```tsx
import { z } from 'zod'

const loginSchema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(8).max(100),
})

type LoginFormData = z.infer<typeof loginSchema>

function LoginForm() {
  const router = useRouter()
  const loginMutation = useServerFn(loginFn)
  const [fieldErrors, setFieldErrors] = useState<
    Partial<Record<keyof LoginFormData, string>>
  >({})

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    setFieldErrors({})

    const formData = Object.fromEntries(new FormData(e.currentTarget))
    const parsed = loginSchema.safeParse(formData)

    if (!parsed.success) {
      const errors = parsed.error.flatten().fieldErrors
      setFieldErrors({
        email: errors.email?.[0],
        password: errors.password?.[0],
      })
      return
    }

    try {
      const result = await loginMutation.mutate(parsed.data)
      if (result.error) {
        setFieldErrors({ email: result.error })
        return
      }
      await router.invalidate()
    } catch {
      setFieldErrors({ email: 'An unexpected error occurred. Please retry.' })
    }
  }

  const isPending = loginMutation.status === 'pending'

  return (
    <form onSubmit={handleSubmit} aria-busy={isPending}>
      <fieldset disabled={isPending}>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          name="email"
          autoComplete="username"
          aria-invalid={!!fieldErrors.email}
          aria-describedby={fieldErrors.email ? 'email-error' : undefined}
        />
        {fieldErrors.email && (
          <p id="email-error" role="alert">{fieldErrors.email}</p>
        )}

        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          name="password"
          autoComplete="current-password"
          aria-invalid={!!fieldErrors.password}
        />
        {fieldErrors.password && (
          <p role="alert">{fieldErrors.password}</p>
        )}

        <button type="submit">
          {isPending ? 'Logging in\u2026' : 'Login'}
        </button>
      </fieldset>
    </form>
  )
}
```

重點：使用 `<fieldset disabled>` 一次停用所有子元素，而非逐一管理每個 input 的 `disabled` 屬性。搭配 `aria-busy`、`aria-invalid`、`role="alert"` 確保螢幕閱讀器能正確傳達狀態。

### Optimistic Updates

驗證操作中最常見的 optimistic update 場景是登出——使用者按下登出後，UI 應立即反映未登入狀態，不需等待 server round-trip 完成。

```tsx
function UserMenu({ user }: { user: AuthUser }) {
  const router = useRouter()
  const logoutMutation = useServerFn(logoutFn)
  const [optimisticallyLoggedOut, setOptimisticallyLoggedOut] = useState(false)

  const handleLogout = async () => {
    // 立即更新 UI
    setOptimisticallyLoggedOut(true)

    try {
      await logoutMutation.mutate({})
      await router.invalidate()
    } catch {
      // Server 端登出失敗——回滾 optimistic state
      setOptimisticallyLoggedOut(false)
      // 通知使用者重試
    }
  }

  if (optimisticallyLoggedOut) {
    return <LoginPrompt />
  }

  return (
    <div>
      <span>{user.displayName}</span>
      <button onClick={handleLogout}>登出</button>
    </div>
  )
}
```

對於登入操作，不建議使用 optimistic update——登入涉及憑證驗證，在 server 確認前假設成功會引入安全風險，且失敗率不可忽略。

### 並行驗證操作的 Race Condition 處理

多個 tab 同時開啟時，可能發生以下 race condition：

1. **Tab A 登出、Tab B 發送 authenticated request** — Tab B 收到 401，需觸發全域狀態同步
2. **同時觸發多次登入** — 使用者快速點擊或 retry 機制導致重複 mutation
3. **Token refresh 與 API call 並行** — refresh 進行中的請求需排隊或 retry

```tsx
// 使用 AbortController 取消過時的驗證操作
function useAuthMutation<T>(serverFn: ServerFn<T>) {
  const abortControllerRef = useRef<AbortController | null>(null)

  const execute = useCallback(async (data: T) => {
    // 取消前一次尚未完成的操作
    abortControllerRef.current?.abort()
    const controller = new AbortController()
    abortControllerRef.current = controller

    try {
      const result = await serverFn.mutate(data, {
        signal: controller.signal,
      })
      return result
    } catch (error) {
      if (error instanceof DOMException && error.name === 'AbortError') {
        // 被新操作取代——靜默忽略
        return null
      }
      throw error
    }
  }, [serverFn])

  useEffect(() => {
    return () => abortControllerRef.current?.abort()
  }, [])

  return { execute }
}
```

跨 tab 狀態同步可透過 `BroadcastChannel` 實現：

```tsx
const authChannel = new BroadcastChannel('auth-state')

// 登出時通知所有 tab
async function handleLogout() {
  await logoutFn.mutate({})
  authChannel.postMessage({ type: 'LOGGED_OUT' })
}

// 其他 tab 監聽狀態變更
useEffect(() => {
  const channel = new BroadcastChannel('auth-state')
  channel.onmessage = (event) => {
    if (event.data.type === 'LOGGED_OUT') {
      router.invalidate()
    }
  }
  return () => channel.close()
}, [router])
```

### Session 過期的 Graceful UX

Session 過期時，避免直接跳轉到登入頁導致使用者遺失未儲存的工作。

```tsx
function SessionExpiryGuard({ children }: { children: React.ReactNode }) {
  const auth = useAuth()
  const [showReauthDialog, setShowReauthDialog] = useState(false)

  useEffect(() => {
    if (!auth.sessionExpiresAt) return

    const warningMs = auth.sessionExpiresAt - Date.now() - 5 * 60 * 1000
    if (warningMs <= 0) return

    const timer = setTimeout(() => {
      setShowReauthDialog(true)
    }, warningMs)

    return () => clearTimeout(timer)
  }, [auth.sessionExpiresAt])

  return (
    <>
      {children}
      {showReauthDialog && (
        <ReauthDialog
          onReauthenticated={() => setShowReauthDialog(false)}
          onDismiss={() => {
            setShowReauthDialog(false)
            // 使用者選擇忽略——session 過期後由 route guard 處理
          }}
        />
      )}
    </>
  )
}
```

在 server function 層面，偵測到過期 session 時回傳結構化錯誤而非直接 redirect，讓 client 端有機會顯示 re-auth dialog：

```tsx
export function requireAuth() {
  const session = await useAppSession()

  if (!session.data.userId) {
    return { authError: 'SESSION_EXPIRED' as const, redirectTo: '/login' }
  }

  return { user: await getUser(session.data.userId) }
}
```

## 「記住我」功能與安全考量

「記住我」的本質是延長 session cookie 的 `maxAge`。這看似單純，但涉及幾項安全取捨。

### 基本實作

```tsx
export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator(
    z.object({
      email: z.string().email().max(255),
      password: z.string().min(8).max(100),
      rememberMe: z.boolean().default(false),
    }),
  )
  .handler(async ({ data }) => {
    const user = await authenticateUser(data.email, data.password)
    if (!user) return { error: 'Invalid credentials' }

    const session = await useAppSession()

    const maxAge = data.rememberMe
      ? 30 * 24 * 60 * 60  // 30 天
      : undefined            // Session cookie——瀏覽器關閉時清除

    await session.update({ userId: user.id, issuedAt: Date.now() }, { maxAge })

    return { success: true }
  })
```

### 安全考量

**長效 cookie 的風險面增大：**
- 30 天的 `maxAge` 意味著被竊取的 cookie 在 30 天內皆可利用。若裝置遺失或被竊，攻擊窗口顯著擴大。
- 緩解措施：實作 session rotation——每次成功的 authenticated request 後更新 session token（或至少更新 `issuedAt`），使舊 token 失效。

**共用裝置風險：**
- 在公用電腦上勾選「記住我」會導致後續使用者繼承 session。
- 緩解措施：登入頁面在偵測到非私人裝置時（透過 heuristic 或使用者自行勾選「公用電腦」）隱藏或預設不勾選「記住我」。

**Cookie 加密的必要性：**
- 長效 cookie 必須使用 `httpOnly`、`secure`、`sameSite: 'lax'`。若 session 資料包含敏感欄位，應在 server 端加密後寫入 cookie，而非使用明文 JSON。

**Session revocation 機制：**
- 長效 session 需要 server 端的 revocation 能力。僅依賴 cookie `maxAge` 不足以處理「其他裝置登出」的需求。
- 實務做法：在資料庫維護 active session 列表，每次驗證時比對 session ID 是否仍有效。

## 跨框架與跨方案遷移

### 從 Client-side 驗證遷移

若現有系統將 token 存於 `localStorage` 或純 React Context（無 server 端驗證），遷移步驟：

1. **並行運行兩套驗證機制** — 新的 server-side session 與舊的 client-side token 同時存在
2. **遷移 API 端點** — 逐步將 `Authorization: Bearer` header 驗證改為 cookie-based session 驗證
3. **透過 `beforeLoad` guard 取代 client-side redirect** — route 層級保護移至 server
4. **移除 `localStorage` token** — 確認所有端點皆已遷移後清除

**Session fixation 風險：** 遷移期間若同時支援兩種驗證方式，攻擊者可能利用舊的 client-side token 取得新的 server-side session。確保遷移後的登入流程會產生全新的 session ID，而非直接將舊 token 轉換為 session。

### 從 Next.js 遷移

| Next.js | TanStack Start | 注意事項 |
|---------|---------------|---------|
| API Routes / Route Handlers | Server functions | Server function 不使用 `Request`/`Response` 物件，需重構 |
| NextAuth.js `getServerSession` | `useAppSession()` | Session schema 可能不同——需資料遷移或相容層 |
| Middleware-based guards | `beforeLoad` | Middleware 的 edge runtime 限制不再適用 |
| `next-auth` JWT strategy | Cookie-based session | JWT → cookie session 的切換需雙寫過渡期 |

### 從 Remix 遷移

| Remix | TanStack Start | 注意事項 |
|-------|---------------|---------|
| `loader` / `action` | Server functions | 回傳值結構不同——Remix 使用 `json()`/`redirect()` |
| `createCookieSessionStorage` | `useSession` | Session 加密方式可能不同，需重新產生 session |
| `requireUserId` pattern | `beforeLoad` + auth context | 邏輯等效但執行時機不同 |

### 遷移的通用陷阱

**Session 相容性問題：** 不同框架的 session 加密/簽章方式不同。直接切換會導致所有既有使用者的 session 失效，強制登出。

緩解策略：

```tsx
// 過渡期間同時支援舊、新 session 格式
async function resolveSession(): Promise<SessionData | null> {
  // 優先嘗試新格式
  const newSession = await tryNewSession()
  if (newSession) return newSession

  // Fallback 到舊格式
  const legacySession = await tryLegacySession()
  if (legacySession) {
    // 自動升級：用新格式重寫 session
    await migrateSession(legacySession)
    return legacySession
  }

  return null
}
```

**Feature flag 控制遷移範圍：** 使用 feature flag 逐步將流量切換至新驗證系統，而非一次性全面替換。

```tsx
export const loginFn = createServerFn({ method: 'POST' })
  .handler(async ({ data }) => {
    const useNewAuth = await getFeatureFlag('new-auth-system', {
      // 按使用者百分比逐步 rollout
      rolloutPercentage: await getRolloutPercentage(),
    })

    if (useNewAuth) {
      return handleNewLogin(data)
    }
    return handleLegacyLogin(data)
  })
```

## 託管服務 vs. 自建方案：成本與規模分析

這不是二選一的抽象比較——以下用具體數字分析。

### 成本模型

**託管服務（以 Clerk 為例）：**

| 方案 | MAU | 月費 (USD) | 單位成本 |
|------|-----|-----------|---------|
| Free | 10,000 | $0 | $0 |
| Pro | 10,000+ | $25 base + $0.02/MAU | ~$0.02/user |
| Enterprise | 自訂 | 需洽談 | 通常 $0.01–0.05/user |

假設一個 50,000 MAU 的 SaaS：
- Clerk Pro：$25 + (40,000 × $0.02) = **$825/月**
- 年度成本：**~$10,000**

**自建方案的隱性成本：**

| 項目 | 估計成本 |
|------|---------|
| 初始開發（2-4 週工程師時間） | $8,000–$20,000（一次性） |
| 持續維護（每月 2-4 小時） | $400–$800/月 |
| 基礎設施（Redis session store、監控） | $50–$200/月 |
| 安全 audit（年度） | $2,000–$10,000/年 |
| 安全事件應對（如發生） | 不可預測 |

年度持續成本：**$7,000–$22,000**（不含初始開發）

**損益平衡點分析：**
- 低 MAU（< 20,000）：託管服務通常更經濟，初始開發成本由服務商分攤
- 中 MAU（20,000–100,000）：取決於團隊能力與功能需求，成本相近
- 高 MAU（> 100,000）：自建方案的邊際成本趨近於零，託管服務的 per-user 費用成為顯著開支

### 技術決策矩陣

| 考量因素 | 傾向託管 | 傾向自建 |
|---------|---------|---------|
| 需要 SSO / SCIM / SOC 2 | 強烈傾向 | 可行但開發成本高 |
| 驗證流程有深度客製需求 | 受限 | 完全掌控 |
| 團隊無專職安全工程師 | 強烈傾向 | 風險較高 |
| 資料駐留有法規要求 | 需確認供應商合規 | 完全掌控 |
| 需要多租戶 (multi-tenancy) | 多數有原生支援 | 需自行設計 |
| 遷移風險承受度低 | 廠商鎖定風險 | 無鎖定 |

### 混合策略

實務中常見的做法是：初期使用託管服務快速上線，同時在內部驗證層保持抽象：

```tsx
// 抽象層——隔離具體的 auth provider 實作
interface AuthProvider {
  authenticate(credentials: Credentials): Promise<AuthResult>
  getSession(): Promise<SessionData | null>
  revokeSession(sessionId: string): Promise<void>
  listActiveSessions(userId: string): Promise<SessionInfo[]>
}

// 目前使用 Clerk——未來可替換為自建實作
class ClerkAuthProvider implements AuthProvider {
  async authenticate(credentials: Credentials): Promise<AuthResult> {
    // Clerk SDK 呼叫
  }
  // ...
}

// 遷移時只需實作新的 provider
class CustomAuthProvider implements AuthProvider {
  async authenticate(credentials: Credentials): Promise<AuthResult> {
    // 自建邏輯
  }
  // ...
}
```

## Blue-Green 部署與 Session 相容性

驗證系統的部署需特別處理 session 相容性，因為新舊版本會同時處理請求。

### Blue-Green 部署模式

```
                    ┌──────────────┐
                    │  Load        │
                    │  Balancer    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │                         │
       ┌──────▼──────┐          ┌──────▼──────┐
       │  Blue (v1)  │          │  Green (v2) │
       │  舊 session │          │  新 session │
       │  schema     │          │  schema     │
       └─────────────┘          └─────────────┘
```

當 session schema 有 breaking change 時（例如新增必要欄位），需確保兩個版本都能正確處理對方產生的 session：

```tsx
// Session schema 版本化
interface SessionDataV1 {
  version: 1
  userId: string
}

interface SessionDataV2 {
  version: 2
  userId: string
  organizationId: string  // 新增欄位
  permissions: string[]   // 新增欄位
}

type SessionData = SessionDataV1 | SessionDataV2

function normalizeSession(raw: SessionData): NormalizedSession {
  switch (raw.version) {
    case 1:
      return {
        userId: raw.userId,
        organizationId: null,       // V1 session 無此欄位——使用預設值
        permissions: ['read'],      // 預設最小權限
      }
    case 2:
      return {
        userId: raw.userId,
        organizationId: raw.organizationId,
        permissions: raw.permissions,
      }
    default:
      throw new Error(`Unknown session version: ${(raw as any).version}`)
  }
}
```

部署步驟：
1. 部署 Green (v2)——能讀取 V1 與 V2 session，寫入 V2
2. 逐步將流量從 Blue 切至 Green
3. 觀察錯誤率與 session 解析失敗指標
4. 完全切換後，移除 V1 相容邏輯（或保留至所有 V1 session 自然過期）

## 驗證系統的監控與可觀測性

### 核心健康指標

```tsx
// 驗證事件的結構化 logging
interface AuthEvent {
  type: 'LOGIN_SUCCESS' | 'LOGIN_FAILURE' | 'LOGOUT' | 'SESSION_EXPIRED'
        | 'SESSION_REFRESH' | 'PASSWORD_RESET' | 'ACCOUNT_LOCKED'
  userId?: string
  ip: string
  userAgent: string
  timestamp: number
  metadata?: Record<string, unknown>
}

function logAuthEvent(event: AuthEvent): void {
  // 結構化 log——方便 aggregation 與 alerting
  logger.info({
    ...event,
    // 不記錄密碼、token 等敏感資料
    // PII 欄位依法規需求決定是否 hash
    hashedIp: hashForAnalytics(event.ip),
  })
}
```

應監控的指標與對應告警閾值：

| 指標 | 告警條件 | 可能原因 |
|------|---------|---------|
| 登入失敗率 | > 基準值 2 倍 | 暴力攻擊、服務異常 |
| Session 建立延遲 p99 | > 500ms | 資料庫瓶頸、session store 過載 |
| 同時 active session 數 | 異常激增 | Session fixation 攻擊、bot 流量 |
| Session 解析失敗率 | > 0.1% | 部署造成的 schema 不相容 |
| 密碼重設請求率 | 異常激增 | 帳號接管攻擊的前兆 |

### Audit Trail 需求

金融、醫療等受監管產業對 audit trail 有明確法規要求。即使不在受監管產業，audit log 在安全事件調查時不可或缺。

```tsx
// Audit log 應與業務 log 分離——不可變、有 retention policy
interface AuditEntry {
  eventId: string          // UUID v4
  timestamp: string        // ISO 8601
  actor: {
    userId: string
    ip: string
    userAgent: string
    sessionId: string
  }
  action: string           // 例如 'auth.login', 'auth.password_change'
  resource: {
    type: string           // 例如 'user', 'session'
    id: string
  }
  outcome: 'success' | 'failure'
  reason?: string          // 失敗原因
}
```

Audit log 的儲存建議：
- 與主要資料庫分離（避免因 audit log 量大影響效能）
- 設定 retention policy（依法規，通常 1–7 年）
- 確保 append-only——不可修改或刪除已寫入的記錄
- 考慮使用 append-only 服務如 AWS CloudTrail、Datadog Logs 或專用的 audit log SaaS

## 正式環境檢查清單

### 安全性

- [ ] 密碼使用 bcrypt / scrypt / argon2 雜湊（salt rounds >= 12）
- [ ] Session secret 為 32+ 隨機字元，存放於環境變數，不進版控
- [ ] Cookie 設定：`secure: true`、`sameSite: 'lax'`、`httpOnly: true`
- [ ] 全站 HTTPS，HTTP 請求 301 redirect 至 HTTPS
- [ ] 對登入/註冊端點實施 rate limiting（per-IP 與 per-account）
- [ ] 所有 server function 使用 Zod 進行 input validation
- [ ] 錯誤訊息不洩漏系統資訊（email 是否存在、具體失敗原因）
- [ ] Security headers 已設定：
  - `Strict-Transport-Security: max-age=31536000; includeSubDomains`
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY`（防止 clickjacking）
  - `Content-Security-Policy` 限制 script 來源
  - `Referrer-Policy: strict-origin-when-cross-origin`
- [ ] Session rotation：登入成功後產生新 session ID（防止 session fixation）
- [ ] 支援 session revocation（server 端可主動使特定 session 失效）

### 功能性

- [ ] 登入、登出、註冊的完整 happy path 與 error path 皆測試通過
- [ ] 受保護路由正確 redirect 未驗證使用者
- [ ] Session 在頁面 reload 後仍有效
- [ ] Session 在 `maxAge` 後正確過期
- [ ] 密碼重設的完整流程可運作（包含 token 過期處理）
- [ ] 多 tab / 多裝置的 session 狀態一致性
- [ ] 「記住我」功能的 cookie 生命週期正確

### 監控與 Incident Response

- [ ] 驗證事件的結構化 logging（登入、登出、失敗、session 過期）
- [ ] Audit trail 獨立儲存，滿足 retention policy 要求
- [ ] 異常模式告警（同一 IP 大量失敗、session 建立量異常）
- [ ] Session store 健康度監控（Redis 連線數、記憶體使用）
- [ ] Incident response runbook 已建立：
  - Session secret 洩漏：立即 rotate secret、強制所有使用者重新登入
  - 大規模帳號被盜：啟用全域 MFA 強制、鎖定可疑帳號、通知受影響使用者
  - Session store 故障：降級策略（fallback 至 stateless JWT 或維護頁面）
- [ ] 安全聯繫方式公開（`/.well-known/security.txt`）

### 合規性

- [ ] 依 GDPR / CCPA 適用性處理個人資料（告知、同意、資料主體權利）
- [ ] 提供帳號刪除功能（含 session 與 audit log 中 PII 的處理策略）
- [ ] 資料保留政策已記錄並實作自動清理
- [ ] 密碼政策符合 NIST 800-63B 建議（最小 8 字元、支援 64+ 字元、不強制複雜度規則）

## 實作範例

以下 TanStack 官方範例可作為參考實作：

- **[Basic Auth with Prisma](https://github.com/TanStack/router/tree/main/examples/react/start-basic-auth)** — 自建實作，含資料庫與 session 管理
- **[Supabase Integration](https://github.com/TanStack/router/tree/main/examples/react/start-supabase-basic)** — 第三方託管服務整合
- **[Clerk Integration](https://github.com/TanStack/router/tree/main/examples/react/start-clerk-basic)** — 合作夥伴方案，含預建 UI 元件
- **[WorkOS Integration](https://github.com/TanStack/router/tree/main/examples/react/start-workos)** — 企業級驗證（SSO、SCIM）
- **[Client-side Context Auth](https://github.com/TanStack/router/tree/main/examples/react/authenticated-routes)** — 純 client-side 模式（僅供對照，不建議用於 production）

## 重點整理

- 驗證操作的 UX 需處理 pending / success / error 三種狀態，並使用 `<fieldset disabled>` 與 ARIA 屬性確保可存取性
- Optimistic update 適用於登出等低風險操作；登入因涉及憑證驗證，不適合 optimistic 處理
- 多 tab 並行操作的 race condition 需透過 `AbortController` 與 `BroadcastChannel` 處理
- Session 過期應提供 re-auth dialog 而非直接強制跳轉，避免使用者遺失未儲存的工作
- 「記住我」延長了攻擊窗口——需搭配 session rotation、revocation 機制與共用裝置風險緩解
- 跨框架遷移的核心挑戰是 session 相容性——使用版本化 schema 與 feature flag 逐步切換
- 託管 vs. 自建非二選一——分析具體 MAU、團隊能力與合規需求後決策，初期可透過抽象層保留遷移彈性
- Blue-green 部署時需確保新舊版本的 session schema 雙向相容
- Production 檢查清單需涵蓋 security headers、audit trail、incident response runbook 與合規要求

---

本篇是 TanStack Start 驗證教學的最後一章。回到[簡介](./01-introduction.md)以查看完整的目錄。
