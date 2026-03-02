# 06：路由保護

> 透過 TanStack Router 的 `beforeLoad` hook 實作路由層級的存取控制，涵蓋 layout route 模式、角色型權限、redirect loop 防護及 SSR 時序考量。

## Layout Route 集中式保護

TanStack Router 以 `_` 前綴命名的路由為 **layout route**，不產生 URL segment，僅作為子路由的邏輯容器。將驗證邏輯集中於 layout route，所有子路由即自動受保護。

```
routes/
  _authed.tsx              ← layout route（驗證閘道）
  _authed/
    dashboard.tsx          ← /dashboard（受保護）
    settings.tsx           ← /settings（受保護）
    admin/
      _adminOnly.tsx       ← 巢狀 layout route（角色閘道）
      _adminOnly/
        users.tsx          ← /admin/users（需 admin 角色）
        audit-log.tsx      ← /admin/audit-log（需 admin 角色）
  login.tsx                ← /login（公開）
  index.tsx                ← /（公開）
```

### `beforeLoad` 驗證閘道

```tsx
// routes/_authed.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'
import { getCurrentUserFn } from '../server/auth'

export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    const user = await getCurrentUserFn()

    if (!user) {
      throw redirect({
        to: '/login',
        search: {
          redirect: location.href,
        },
      })
    }

    // 將 user 注入 route context，子路由透過 useRouteContext() 取得
    return { user }
  },
})
```

`beforeLoad` 在元件渲染前執行，確保未授權內容不會短暫出現在畫面上。透過 `return { user }`，所有子路由可直接從 context 取得使用者資料，無需重複呼叫 server function。

### 子路由消費 Context

```tsx
// routes/_authed/dashboard.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed/dashboard')({
  component: DashboardComponent,
})

function DashboardComponent() {
  const { user } = Route.useRouteContext()

  return (
    <div>
      <h1>Welcome, {user.email}</h1>
      {/* dashboard content */}
    </div>
  )
}
```

## 深層巢狀角色型保護

實務上常見多層權限需求：先驗證身份，再驗證角色，甚至針對特定資源驗證 permission。透過巢狀 layout route 逐層堆疊 `beforeLoad`，可組合出任意深度的存取控制鏈。

```tsx
// routes/_authed/admin/_adminOnly.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed/admin/_adminOnly')({
  beforeLoad: async ({ context }) => {
    // context.user 由父層 _authed layout 注入
    if (!context.user.roles?.includes('admin')) {
      throw redirect({ to: '/forbidden' })
    }

    // 可進一步載入 admin 專屬權限
    const permissions = await getAdminPermissionsFn(context.user.id)
    return { permissions }
  },
})
```

```tsx
// routes/_authed/admin/_adminOnly/users.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed/admin/_adminOnly/users')({
  beforeLoad: async ({ context }) => {
    // 第三層檢查：特定資源權限
    if (!context.permissions.includes('manage_users')) {
      throw redirect({ to: '/forbidden' })
    }
  },
  component: UserManagement,
})
```

執行順序為 `_authed.beforeLoad` -> `_adminOnly.beforeLoad` -> `users.beforeLoad`。任一層 throw redirect 即中斷後續鏈路。

## Redirect Loop 防護

當 `/login` 頁面本身也被保護，或驗證狀態判定存在 race condition 時，容易產生無限 redirect loop。防護策略：

```tsx
// routes/_authed.tsx
export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    const user = await getCurrentUserFn()

    if (!user) {
      // 防止 redirect loop：若目標已是 /login 則不再 redirect
      if (location.pathname === '/login') {
        return { user: null }
      }

      throw redirect({
        to: '/login',
        search: {
          redirect: location.href,
        },
      })
    }

    return { user }
  },
})
```

登入頁接收 redirect 參數時，應驗證其合法性：

```tsx
// routes/login.tsx
import { createFileRoute, useNavigate } from '@tanstack/react-router'
import { z } from 'zod'

const loginSearchSchema = z.object({
  redirect: z.string().optional(),
})

export const Route = createFileRoute('/login')({
  validateSearch: loginSearchSchema,
  component: LoginPage,
})

function LoginPage() {
  const navigate = useNavigate()
  const { redirect: redirectUrl } = Route.useSearch()

  const handleLogin = async (data: LoginFormData) => {
    await loginFn({ data })

    // 僅允許同源相對路徑，防止 open redirect 攻擊
    const safeRedirect =
      redirectUrl && redirectUrl.startsWith('/') ? redirectUrl : '/dashboard'

    navigate({ to: safeRedirect })
  }

  return (
    <form onSubmit={handleLogin}>
      {/* login form fields */}
    </form>
  )
}
```

## 跨 Auth Redirect 保留表單狀態

使用者填寫表單途中 session 過期、觸發 redirect 至登入頁後，若直接跳轉將遺失所有輸入資料。可將表單狀態序列化後暫存，登入後還原：

```tsx
// utils/form-state-persistence.ts
const FORM_STATE_KEY = 'pending_form_state'

export function persistFormState(formData: Record<string, unknown>) {
  try {
    sessionStorage.setItem(FORM_STATE_KEY, JSON.stringify(formData))
  } catch {
    // sessionStorage 不可用（如 SSR 或 private browsing 限制）時靜默失敗
  }
}

export function restoreFormState<T>(): T | null {
  try {
    const stored = sessionStorage.getItem(FORM_STATE_KEY)
    if (!stored) return null
    sessionStorage.removeItem(FORM_STATE_KEY)
    return JSON.parse(stored) as T
  } catch {
    return null
  }
}
```

在 `beforeLoad` redirect 前觸發保存：

```tsx
// 於表單元件中偵測 session 過期
const submitWithAuthCheck = async (formData: CreatePostData) => {
  try {
    await createPostFn({ data: formData })
  } catch (error) {
    if (isAuthError(error)) {
      persistFormState(formData)
      // beforeLoad 的 redirect 會自然觸發，或手動導向
      navigate({ to: '/login', search: { redirect: location.href } })
    }
    throw error
  }
}
```

## 處理導航途中 Session 過期

使用者在應用程式內導航時，session 可能恰好過期。`beforeLoad` 每次導航都會執行，因此能即時攔截。但需注意 stale closure 問題：

```tsx
// routes/_authed.tsx
export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location, cause }) => {
    const user = await getCurrentUserFn()

    if (!user) {
      // cause 區分初次載入與 client-side 導航
      // 'enter' = 初次進入或 hard refresh
      // 'stay'  = 同路由 search/hash 變更
      if (cause === 'enter') {
        throw redirect({
          to: '/login',
          search: { redirect: location.href },
        })
      }

      // client-side 導航時，可選擇顯示 toast 而非硬跳轉
      throw redirect({
        to: '/login',
        search: {
          redirect: location.href,
          reason: 'session_expired',
        },
      })
    }

    return { user }
  },
})
```

## SSR 與 Client-Side `beforeLoad` 時序

在 TanStack Start 的 SSR 模式下，`beforeLoad` 的行為有關鍵差異：

**SSR 階段（server）**：`beforeLoad` 在 server 上執行。`redirect` 會產生 HTTP 302 response，瀏覽器收到後直接跳轉。此時 `getCurrentUserFn()` 直接存取 server-side session，無需網路往返。

**Client-Side 導航（hydration 後）**：`beforeLoad` 在 browser 中執行。`getCurrentUserFn()` 透過 RPC 呼叫 server function，存在網路延遲。`redirect` 由 TanStack Router 在 client 端處理。

需注意的陷阱：

```tsx
// 錯誤示範：在 beforeLoad 中使用 browser-only API
export const Route = createFileRoute('/_authed')({
  beforeLoad: async () => {
    // localStorage 在 SSR 階段不存在，會拋出 ReferenceError
    const token = localStorage.getItem('auth_token')
    // ...
  },
})

// 正確做法：始終透過 server function 驗證
export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    // server function 在 SSR 時直接執行，在 client 時透過 RPC
    const user = await getCurrentUserFn()
    if (!user) {
      throw redirect({ to: '/login', search: { redirect: location.href } })
    }
    return { user }
  },
})
```

## Concurrent Navigation 的 Race Condition

當使用者快速連續點擊多個受保護連結時，多個 `beforeLoad` 可能同時執行。TanStack Router 內部會取消先前的 pending navigation，但 `beforeLoad` 中已發出的 async 請求不會自動中止。若 `getCurrentUserFn()` 回應緩慢，可能出現：

1. 使用者點擊 `/dashboard`，`beforeLoad` 發出驗證請求 A
2. 使用者立即點擊 `/settings`，`beforeLoad` 發出驗證請求 B
3. 請求 A 的結果被丟棄，但 server 仍處理了該請求

對於大多數驗證場景這不構成問題（GET 請求且冪等），但若 `beforeLoad` 中包含副作用（如 audit log 寫入），需自行透過 `AbortController` 管理：

```tsx
export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ abortController, location }) => {
    const user = await getCurrentUserFn({
      signal: abortController.signal,
    })

    if (!user) {
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      })
    }

    return { user }
  },
})
```

## 重點整理

- Layout route（`_authed.tsx`）集中管理驗證邏輯，子路由自動受保護
- `beforeLoad` 於元件渲染前執行，`throw redirect(...)` 阻斷未授權存取
- 巢狀 layout route 可堆疊多層權限檢查（身份 -> 角色 -> 資源權限）
- Redirect URL 必須驗證為同源相對路徑，防止 open redirect 攻擊
- SSR 階段 `beforeLoad` 在 server 執行，不可使用 browser-only API
- Concurrent navigation 下，多個 `beforeLoad` 可能同時觸發，含副作用的操作應搭配 `AbortController`
- Session 過期的處理應區分初次載入與 client-side 導航，提供適當的 UX 回饋
- 跨 auth redirect 的表單狀態可透過 `sessionStorage` 暫存與還原

---

下一篇：[Implementation Patterns](./07-implementation-patterns.md)
