# 09：測試認證系統

> 認證測試的實務策略——涵蓋 middleware 隔離測試、OAuth contract testing、token 過期驗證、速率限制行為，以及 CI/CD pipeline 中的注意事項。

## 測試策略概覽

認證測試可分為四個層級，各自對應不同的故障模式：

| 層級 | 範圍 | 典型工具 | 偵測目標 |
|------|------|----------|----------|
| Unit | 單一函式 / middleware | Vitest | 邏輯錯誤、邊界條件 |
| Integration | 路由 + session + 資料庫 | Vitest + Testing Library | 元件協作失敗 |
| Contract | OAuth provider 回應格式 | Pact / MSW | 第三方 API 變更 |
| Load | 登入端點吞吐量 | k6 / Artillery | 併發瓶頸、rate limiter 行為 |

以下依序展開各層級的實作細節。

## Test Factories 與 Fixtures

在進入各測試層級之前，先建立可重用的 test factory。避免在每個測試檔案中重複建立使用者資料：

```tsx
// test/factories/user.ts
import { faker } from '@faker-js/faker'
import bcrypt from 'bcryptjs'

interface CreateUserOptions {
  email?: string
  password?: string
  role?: 'user' | 'admin'
  emailVerified?: boolean
}

const DEFAULT_PASSWORD = 'Test!Passw0rd'

export async function createUserFixture(
  db: PrismaClient,
  overrides: CreateUserOptions = {},
) {
  const plainPassword = overrides.password ?? DEFAULT_PASSWORD
  return db.user.create({
    data: {
      email: overrides.email ?? faker.internet.email(),
      password: await bcrypt.hash(plainPassword, 4), // cost=4 加速測試
      name: faker.person.fullName(),
      role: overrides.role ?? 'user',
      emailVerified: overrides.emailVerified ?? true,
      createdAt: new Date(),
    },
  })
}

export function createSessionFixture(userId: string, options?: {
  expiresAt?: Date
  csrfToken?: string
}) {
  return {
    userId,
    expiresAt: options?.expiresAt ?? new Date(Date.now() + 86_400_000),
    csrfToken: options?.csrfToken ?? crypto.randomUUID(),
  }
}
```

注意 bcrypt cost factor 在測試環境中降為 4——正式環境使用 12 以上，但在測試中不需要密碼學強度，只需要雜湊行為正確。

## Auth Middleware 隔離測試

Middleware 是認證邏輯的核心路徑，必須能獨立測試，不依賴完整的 HTTP server 或 router：

```tsx
// __tests__/middleware/auth-middleware.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { authMiddleware } from '../../server/middleware/auth'
import { createSessionFixture } from '../factories/user'

function createMockRequest(overrides: Partial<Request> = {}): Request {
  return {
    headers: new Headers({ cookie: '' }),
    url: 'http://localhost:3000/api/protected',
    method: 'GET',
    ...overrides,
  } as Request
}

function createMockContext() {
  return {
    next: vi.fn().mockResolvedValue(new Response('OK', { status: 200 })),
    redirect: vi.fn((url: string) => new Response(null, {
      status: 302,
      headers: { Location: url },
    })),
  }
}

describe('authMiddleware', () => {
  let getSession: ReturnType<typeof vi.fn>

  beforeEach(() => {
    getSession = vi.fn()
    vi.clearAllMocks()
  })

  it('呼叫 next() 當 session 有效', async () => {
    const session = createSessionFixture('user-1')
    getSession.mockResolvedValue(session)

    const ctx = createMockContext()
    await authMiddleware({ getSession })(createMockRequest(), ctx)

    expect(ctx.next).toHaveBeenCalledOnce()
    expect(ctx.redirect).not.toHaveBeenCalled()
  })

  it('回傳 302 到 /login 當 session 不存在', async () => {
    getSession.mockResolvedValue(null)

    const ctx = createMockContext()
    const response = await authMiddleware({ getSession })(createMockRequest(), ctx)

    expect(response.status).toBe(302)
    expect(response.headers.get('Location')).toBe('/login')
  })

  it('回傳 302 當 session 已過期', async () => {
    const expiredSession = createSessionFixture('user-1', {
      expiresAt: new Date(Date.now() - 1000), // 1 秒前已過期
    })
    getSession.mockResolvedValue(expiredSession)

    const ctx = createMockContext()
    const response = await authMiddleware({ getSession })(createMockRequest(), ctx)

    expect(response.status).toBe(302)
  })
})
```

關鍵：middleware 接受 `getSession` 作為依賴注入，而非直接 import session 模組。這讓測試可以完全控制 session 行為，無需 mock 模組層級的副作用。

## Snapshot Testing：Auth Redirect 行為

對認證重新導向的完整 response 進行 snapshot testing，確保 redirect URL、status code、headers 的結構不會無意間改變：

```tsx
// __tests__/redirects/auth-redirects.test.ts
import { describe, it, expect } from 'vitest'
import { buildAuthRedirect } from '../../server/auth/redirects'

describe('auth redirect responses', () => {
  it('未認證導向保留原始 URL 為 redirect param', () => {
    const response = buildAuthRedirect({
      reason: 'unauthenticated',
      originalUrl: '/dashboard/settings?tab=billing',
    })

    expect({
      status: response.status,
      location: response.headers.get('Location'),
      cacheControl: response.headers.get('Cache-Control'),
    }).toMatchInlineSnapshot(`
      {
        "status": 302,
        "location": "/login?redirect=%2Fdashboard%2Fsettings%3Ftab%3Dbilling",
        "cacheControl": "no-store",
      }
    `)
  })

  it('權限不足導向至 403 頁面', () => {
    const response = buildAuthRedirect({
      reason: 'forbidden',
      originalUrl: '/admin/users',
    })

    expect({
      status: response.status,
      location: response.headers.get('Location'),
    }).toMatchInlineSnapshot(`
      {
        "status": 302,
        "location": "/403",
      }
    `)
  })
})
```

使用 `toMatchInlineSnapshot` 而非 `toMatchSnapshot`——inline snapshot 讓 PR review 時可以直接看到預期值，不需要另開 `.snap` 檔案。

## CSRF Protection 測試

CSRF token 驗證是容易被忽略的測試項目。確保 state-changing 端點拒絕缺少或無效 CSRF token 的請求：

```tsx
// __tests__/security/csrf.test.ts
import { describe, it, expect } from 'vitest'
import { validateCsrfToken } from '../../server/security/csrf'

describe('CSRF protection', () => {
  const validToken = 'abc123-valid-csrf-token'
  const sessionCsrfToken = 'abc123-valid-csrf-token'

  it('接受匹配的 CSRF token', () => {
    expect(validateCsrfToken(validToken, sessionCsrfToken)).toBe(true)
  })

  it('拒絕缺少的 CSRF token', () => {
    expect(validateCsrfToken(undefined, sessionCsrfToken)).toBe(false)
  })

  it('拒絕不匹配的 CSRF token', () => {
    expect(validateCsrfToken('wrong-token', sessionCsrfToken)).toBe(false)
  })

  it('使用 timing-safe comparison 防止 timing attack', () => {
    // 確認實作使用 crypto.timingSafeEqual 而非 ===
    // 這裡透過測量不同長度 token 的比較時間來間接驗證
    const start1 = performance.now()
    for (let i = 0; i < 10_000; i++) {
      validateCsrfToken('a', sessionCsrfToken)
    }
    const time1 = performance.now() - start1

    const start2 = performance.now()
    for (let i = 0; i < 10_000; i++) {
      validateCsrfToken('a'.repeat(1000), sessionCsrfToken)
    }
    const time2 = performance.now() - start2

    // timing-safe comparison 的時間差應該很小
    // 這是一個粗略的 heuristic，非決定性斷言
    expect(Math.abs(time2 - time1)).toBeLessThan(50)
  })
})
```

## 時間相關 Token 測試

Token 過期測試是常見的 flaky test 來源。使用 `vi.useFakeTimers()` 控制時間：

```tsx
// __tests__/session/token-expiry.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { isSessionValid, refreshSession } from '../../server/session'
import { createSessionFixture } from '../factories/user'

describe('token expiry', () => {
  beforeEach(() => {
    vi.useFakeTimers()
    vi.setSystemTime(new Date('2026-01-15T10:00:00Z'))
  })

  afterEach(() => {
    vi.useRealTimers()
  })

  it('session 在 expiresAt 之前為有效', () => {
    const session = createSessionFixture('user-1', {
      expiresAt: new Date('2026-01-16T10:00:00Z'), // 24 小時後
    })

    expect(isSessionValid(session)).toBe(true)
  })

  it('session 在 expiresAt 之後為無效', () => {
    const session = createSessionFixture('user-1', {
      expiresAt: new Date('2026-01-15T09:00:00Z'), // 1 小時前
    })

    expect(isSessionValid(session)).toBe(false)
  })

  it('session 在剛好過期的瞬間為無效', () => {
    const session = createSessionFixture('user-1', {
      expiresAt: new Date('2026-01-15T10:00:00Z'), // 正好是現在
    })

    // 邊界條件：expiresAt === now 應視為過期
    expect(isSessionValid(session)).toBe(false)
  })

  it('refresh 延長 session 存活時間', async () => {
    const session = createSessionFixture('user-1', {
      expiresAt: new Date('2026-01-15T10:30:00Z'), // 30 分鐘後過期
    })

    const refreshed = await refreshSession(session)

    // refresh 後的 expiresAt 應從現在起算重新設定
    expect(refreshed.expiresAt.getTime()).toBeGreaterThan(
      new Date('2026-01-15T10:30:00Z').getTime()
    )
  })
})
```

## 正確 Mock `useSession`

在 component 測試中 mock `useSession` 是高頻操作，但 mock 方式不正確會導致 test pollution。以下是推薦的模式：

```tsx
// test/helpers/auth-context.tsx
import { vi } from 'vitest'
import type { Session } from '../../server/session'

// 集中管理 mock，避免每個測試檔案各自 vi.mock
const mockSession = vi.hoisted(() => ({
  current: null as Session | null,
}))

vi.mock('../../hooks/useSession', () => ({
  useSession: () => mockSession.current,
}))

export function setMockSession(session: Session | null) {
  mockSession.current = session
}

export function clearMockSession() {
  mockSession.current = null
}
```

```tsx
// __tests__/components/protected-page.test.tsx
import { describe, it, expect, beforeEach } from 'vitest'
import { render, screen } from '@testing-library/react'
import { setMockSession, clearMockSession } from '../helpers/auth-context'
import { ProtectedPage } from '../../components/ProtectedPage'

describe('ProtectedPage', () => {
  beforeEach(() => {
    clearMockSession() // 每個測試前重設，防止 test pollution
  })

  it('已認證時渲染內容', () => {
    setMockSession({
      userId: 'user-1',
      expiresAt: new Date(Date.now() + 86_400_000),
      csrfToken: 'token',
    })

    render(<ProtectedPage />)
    expect(screen.getByTestId('protected-content')).toBeInTheDocument()
  })

  it('未認證時渲染 redirect 或 fallback', () => {
    // mockSession 已被 clearMockSession 重設為 null
    render(<ProtectedPage />)
    expect(screen.queryByTestId('protected-content')).not.toBeInTheDocument()
  })
})
```

**陷阱**：使用 `vi.hoisted` 確保 mock 變數在模組載入之前就存在。若將 mock state 放在 `vi.mock` 外部的一般變數中，module hoisting 順序可能導致 `undefined`。

## 防止 Test Pollution：共享 Session State

當測試以 `--pool=threads`（Vitest 預設）並行執行時，共享的 in-memory session store 會造成 test pollution：

```tsx
// vitest.config.ts — 認證測試的建議設定
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    // 認證測試使用 forks 而非 threads，確保 process-level 隔離
    pool: 'forks',
    poolOptions: {
      forks: {
        isolate: true,
      },
    },
    // 或者使用 workspace 將 auth 測試獨立分組
    // 見下方 CI/CD 段落
    setupFiles: ['./test/setup/auth-setup.ts'],
  },
})
```

```tsx
// test/setup/auth-setup.ts
import { beforeEach, afterEach } from 'vitest'
import { prisma } from '../../server/db'

// 使用 transaction rollback 確保每個測試的資料庫狀態獨立
beforeEach(async () => {
  // Prisma interactive transactions 作為 savepoint
  await prisma.$executeRaw`BEGIN`
})

afterEach(async () => {
  await prisma.$executeRaw`ROLLBACK`
})
```

## OAuth Provider Contract Testing

OAuth provider 的回應格式可能在無預警的情況下變更。Contract test 在 CI 中定期驗證回應結構是否符合預期 schema：

```tsx
// __tests__/contracts/oauth-provider.test.ts
import { describe, it, expect } from 'vitest'
import { z } from 'zod'
import { http, HttpResponse } from 'msw'
import { setupServer } from 'msw/node'

// 定義預期的 OAuth token response schema
const OAuthTokenResponseSchema = z.object({
  access_token: z.string(),
  token_type: z.literal('Bearer'),
  expires_in: z.number().positive(),
  refresh_token: z.string().optional(),
  scope: z.string(),
})

const OAuthUserInfoSchema = z.object({
  sub: z.string(),
  email: z.string().email(),
  email_verified: z.boolean(),
  name: z.string().optional(),
  picture: z.string().url().optional(),
})

// MSW handler 模擬 provider 回應
const handlers = [
  http.post('https://oauth.provider.com/token', () => {
    return HttpResponse.json({
      access_token: 'mock-access-token',
      token_type: 'Bearer',
      expires_in: 3600,
      refresh_token: 'mock-refresh-token',
      scope: 'openid email profile',
    })
  }),
  http.get('https://oauth.provider.com/userinfo', () => {
    return HttpResponse.json({
      sub: '12345',
      email: 'user@provider.com',
      email_verified: true,
      name: 'Test User',
      picture: 'https://provider.com/avatar.jpg',
    })
  }),
]

const server = setupServer(...handlers)

describe('OAuth provider contract', () => {
  beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
  afterEach(() => server.resetHandlers())
  afterAll(() => server.close())

  it('token exchange 回應符合預期 schema', async () => {
    const response = await fetch('https://oauth.provider.com/token', {
      method: 'POST',
      body: new URLSearchParams({ code: 'auth-code', grant_type: 'authorization_code' }),
    })
    const data = await response.json()
    expect(() => OAuthTokenResponseSchema.parse(data)).not.toThrow()
  })

  it('userinfo 回應符合預期 schema', async () => {
    const response = await fetch('https://oauth.provider.com/userinfo', {
      headers: { Authorization: 'Bearer mock-access-token' },
    })
    const data = await response.json()
    expect(() => OAuthUserInfoSchema.parse(data)).not.toThrow()
  })

  it('處理 provider 回傳非預期欄位時不崩潰', async () => {
    server.use(
      http.get('https://oauth.provider.com/userinfo', () => {
        return HttpResponse.json({
          sub: '12345',
          email: 'user@provider.com',
          email_verified: true,
          unexpected_field: 'should be ignored',
        })
      })
    )

    const response = await fetch('https://oauth.provider.com/userinfo', {
      headers: { Authorization: 'Bearer mock-access-token' },
    })
    const data = await response.json()
    // Zod 的 .parse 在 strict mode 下會拒絕多餘欄位
    // 使用 .passthrough() 允許未知欄位
    expect(() => OAuthUserInfoSchema.passthrough().parse(data)).not.toThrow()
  })
})
```

進階做法：在 CI 中設定一個 scheduled job，以真實的 test credentials 對 OAuth provider 的 discovery endpoint (`/.well-known/openid-configuration`) 發出請求，驗證 issuer、jwks_uri 等欄位沒有變動。

## Load Testing 登入端點

認證端點是 brute-force attack 的主要目標，也是效能瓶頸的常見位置（bcrypt 雜湊的 CPU 成本）。使用 k6 進行 load testing：

```javascript
// load-tests/login-endpoint.js
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate } from 'k6/metrics'

const failRate = new Rate('failed_logins')

export const options = {
  scenarios: {
    sustained_load: {
      executor: 'constant-arrival-rate',
      rate: 50,          // 每秒 50 個請求
      timeUnit: '1s',
      duration: '2m',
      preAllocatedVUs: 100,
      maxVUs: 200,
    },
    spike: {
      executor: 'ramping-arrival-rate',
      startRate: 10,
      timeUnit: '1s',
      stages: [
        { duration: '30s', target: 200 },  // 急速攀升
        { duration: '1m', target: 200 },   // 維持高峰
        { duration: '30s', target: 10 },   // 回落
      ],
      preAllocatedVUs: 300,
      maxVUs: 500,
      startTime: '2m',   // sustained_load 結束後開始
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1500'],
    failed_logins: ['rate<0.01'],   // 合法登入的失敗率應 < 1%
  },
}

export default function () {
  const payload = JSON.stringify({
    email: `loadtest+${__VU}@example.com`,
    password: 'LoadTest!Pass123',
  })

  const res = http.post('http://localhost:3000/api/auth/login', payload, {
    headers: { 'Content-Type': 'application/json' },
  })

  check(res, {
    'status is 200 or 429': (r) => r.status === 200 || r.status === 429,
    'response time < 500ms': (r) => r.timings.duration < 500,
  })

  failRate.add(res.status !== 200 && res.status !== 429)
  sleep(0.1)
}
```

注意 `status === 429` 被視為正常——rate limiter 拒絕過多請求是預期行為。

## 測試 Rate Limiting 行為

Rate limiter 本身也需要被測試。確保它在正確的閾值觸發，且不會影響正常使用者：

```tsx
// __tests__/security/rate-limit.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { createRateLimiter } from '../../server/security/rate-limit'

describe('login rate limiter', () => {
  beforeEach(() => {
    vi.useFakeTimers()
    vi.setSystemTime(new Date('2026-01-15T10:00:00Z'))
  })

  afterEach(() => {
    vi.useRealTimers()
  })

  it('允許閾值內的請求', () => {
    const limiter = createRateLimiter({ maxAttempts: 5, windowMs: 60_000 })
    const ip = '192.168.1.1'

    for (let i = 0; i < 5; i++) {
      expect(limiter.isAllowed(ip)).toBe(true)
    }
  })

  it('超過閾值後拒絕請求', () => {
    const limiter = createRateLimiter({ maxAttempts: 5, windowMs: 60_000 })
    const ip = '192.168.1.1'

    for (let i = 0; i < 5; i++) limiter.consume(ip)

    expect(limiter.isAllowed(ip)).toBe(false)
  })

  it('時間窗口過後重設計數器', () => {
    const limiter = createRateLimiter({ maxAttempts: 5, windowMs: 60_000 })
    const ip = '192.168.1.1'

    for (let i = 0; i < 5; i++) limiter.consume(ip)
    expect(limiter.isAllowed(ip)).toBe(false)

    // 快轉 61 秒
    vi.advanceTimersByTime(61_000)

    expect(limiter.isAllowed(ip)).toBe(true)
  })

  it('不同 IP 各自獨立計數', () => {
    const limiter = createRateLimiter({ maxAttempts: 5, windowMs: 60_000 })

    for (let i = 0; i < 5; i++) limiter.consume('192.168.1.1')

    expect(limiter.isAllowed('192.168.1.1')).toBe(false)
    expect(limiter.isAllowed('192.168.1.2')).toBe(true)
  })
})
```

## Integration Test：完整認證流程

整合測試驗證從路由進入到認證檢查再到畫面渲染的完整路徑：

```tsx
// __tests__/integration/auth-flow.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { RouterProvider, createMemoryHistory } from '@tanstack/react-router'
import { createTestRouter } from '../helpers/router'
import { createUserFixture } from '../factories/user'
import { prisma } from '../../server/db'

describe('認證流程整合測試', () => {
  let testUser: Awaited<ReturnType<typeof createUserFixture>>

  beforeEach(async () => {
    await prisma.$executeRaw`BEGIN`
    testUser = await createUserFixture(prisma, {
      email: 'integration@test.com',
      password: 'Test!Passw0rd',
    })
  })

  afterEach(async () => {
    await prisma.$executeRaw`ROLLBACK`
  })

  it('未認證存取受保護路由 → 重新導向 /login → 登入成功 → 回到原路由', async () => {
    const user = userEvent.setup()
    const router = createTestRouter()
    const history = createMemoryHistory({ initialEntries: ['/dashboard'] })

    render(<RouterProvider router={router} history={history} />)

    // 應被重新導向到 /login，且 URL 包含 redirect param
    await waitFor(() => {
      expect(screen.getByRole('heading', { name: /login/i })).toBeInTheDocument()
    })

    // 填寫表單
    await user.type(screen.getByLabelText(/email/i), 'integration@test.com')
    await user.type(screen.getByLabelText(/password/i), 'Test!Passw0rd')
    await user.click(screen.getByRole('button', { name: /login/i }))

    // 登入後應回到原先嘗試存取的 /dashboard
    await waitFor(() => {
      expect(screen.getByTestId('dashboard-content')).toBeInTheDocument()
    })
  })

  it('登出後無法再存取受保護路由', async () => {
    const user = userEvent.setup()
    const router = createTestRouter({ initialSession: { userId: testUser.id } })
    const history = createMemoryHistory({ initialEntries: ['/dashboard'] })

    render(<RouterProvider router={router} history={history} />)

    await waitFor(() => {
      expect(screen.getByTestId('dashboard-content')).toBeInTheDocument()
    })

    await user.click(screen.getByRole('button', { name: /logout/i }))

    await waitFor(() => {
      expect(screen.getByRole('heading', { name: /login/i })).toBeInTheDocument()
    })
  })
})
```

使用 `userEvent` 而非 `fireEvent`——前者模擬真實使用者行為（focus、keyboard events 的完整序列），更能捕捉實際的互動問題。

## CI/CD 注意事項

### Vitest Workspace 分離認證測試

```tsx
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config'

export default defineWorkspace([
  {
    extends: './vitest.config.ts',
    test: {
      name: 'auth-unit',
      include: ['__tests__/middleware/**', '__tests__/security/**', '__tests__/session/**'],
      pool: 'threads',
    },
  },
  {
    extends: './vitest.config.ts',
    test: {
      name: 'auth-integration',
      include: ['__tests__/integration/**'],
      pool: 'forks',          // process 隔離，防止 session state 汙染
      poolOptions: { forks: { isolate: true } },
      testTimeout: 15_000,    // 整合測試給予較長 timeout
    },
  },
  {
    extends: './vitest.config.ts',
    test: {
      name: 'auth-contracts',
      include: ['__tests__/contracts/**'],
      pool: 'threads',
    },
  },
])
```

### CI Pipeline 建議

- **Unit + Contract tests**：每個 PR 必跑，執行時間控制在 30 秒內
- **Integration tests**：每個 PR 必跑，使用獨立的 test database（ephemeral container）
- **Load tests**：僅在 `main` branch 的 nightly build 中執行，避免阻塞日常 PR flow
- **OAuth contract tests against real providers**：scheduled job（每週一次），使用 test credentials，失敗時觸發 Slack alert 而非阻斷 pipeline

### 環境變數管理

```bash
# .env.test — 不納入版本控制
DATABASE_URL="postgresql://test:test@localhost:5433/auth_test"
SESSION_SECRET="test-only-not-a-real-secret"
OAUTH_CLIENT_ID="test-client-id"
OAUTH_CLIENT_SECRET="test-client-secret"
```

在 CI 中使用 service container 提供 test database：

```yaml
# .github/workflows/auth-tests.yml（片段）
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_DB: auth_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - 5433:5432
    options: >-
      --health-cmd="pg_isready"
      --health-interval=5s
      --health-timeout=3s
      --health-retries=5
```

## 重點整理

- 建立 test factory 集中管理使用者 fixture，避免測試間的重複程式碼，並使用低 cost factor 的 bcrypt 加速測試執行
- Auth middleware 透過依賴注入 `getSession` 實現隔離測試，不需啟動完整 HTTP server
- 使用 `vi.useFakeTimers()` 測試 token 過期與 session refresh，避免時間相關的 flaky tests
- 以 `vi.hoisted` 搭配集中式 mock helper 管理 `useSession`，在 `beforeEach` 中重設以防止 test pollution
- 並行測試需要 process-level 隔離（`pool: 'forks'`）或 transaction rollback，防止共享 session state 互相干擾
- Contract test 以 Zod schema + MSW 驗證 OAuth provider 回應格式，及早偵測第三方 API breaking change
- Load test 以 k6 驗證登入端點在高併發下的 latency 與 rate limiter 行為
- CI pipeline 區分測試層級：unit/contract 每 PR 必跑、integration 搭配 ephemeral database、load test 僅 nightly

---

下一篇：[常見模式與正式環境部署](./10-common-patterns-and-production.md)
