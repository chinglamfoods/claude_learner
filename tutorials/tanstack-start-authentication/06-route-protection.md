# 06：路由保護

> 學習如何保護路由，讓只有已驗證的使用者才能存取特定頁面，使用 TanStack Router 的 `beforeLoad` hook。

## 為什麼需要保護路由？

如果沒有路由保護，任何人都可以直接在瀏覽器輸入 `/dashboard` 或 `/admin` 的 URL 來存取頁面——即使他們尚未登入。路由保護能確保：

- 未驗證的使用者會被導向登入頁面
- 已驗證的使用者只能看到他們被授權的內容
- 檢查在頁面渲染**之前**就會執行（不會出現未授權內容的閃爍畫面）

## Layout Route 模式

TanStack Start 中最強大的模式之一，就是使用 **layout route** 來一次保護一整組頁面。

### 運作原理

在 TanStack Router 中，以 `_` 開頭的路由是 **layout route**。它們不會增加 URL 路徑片段——只是用來包裝其子路由。

```
routes/
  _authed.tsx              ← layout route（檢查驗證狀態）
  _authed/
    dashboard.tsx          ← /dashboard（受保護）
    settings.tsx           ← /settings（受保護）
    admin/
      index.tsx            ← /admin（受保護）
  login.tsx                ← /login（公開）
  index.tsx                ← /（公開）
```

任何在 `_authed/` 資料夾內的路由都會自動受到保護，因為 `_authed.tsx` 會先執行。

### 驗證檢查：`beforeLoad`

```tsx
// routes/_authed.tsx — 所有受保護頁面的 layout route
import { createFileRoute, redirect } from '@tanstack/react-router'
import { getCurrentUserFn } from '../server/auth'

export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    // 呼叫伺服器檢查使用者是否已登入
    const user = await getCurrentUserFn()

    if (!user) {
      // 未登入 → 導向登入頁面
      // 儲存目前的 URL，以便登入後可以導回原頁面
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      })
    }

    // 使用者已登入 → 將使用者資料傳遞給所有子路由
    return { user }
  },
})
```

讓我們逐步拆解：

1. **`beforeLoad`** 會在路由元件渲染之前執行。這裡就是檢查驗證狀態的地方。
2. **`getCurrentUserFn()`** 呼叫你的伺服器函式來檢查 session。
3. **`throw redirect(...)`** 會在使用者未驗證時將其導向 `/login`。`search: { redirect: location.href }` 會儲存使用者原本要前往的位置，以便登入後能導回。
4. **`return { user }`** 讓使用者資料可以透過 `Route.useRouteContext()` 在所有子路由中取得。

### 在受保護的路由中存取使用者資料

子路由可以存取 `_authed.tsx` 提供的使用者資料：

```tsx
// routes/_authed/dashboard.tsx — 一個受保護的路由
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed/dashboard')({
  component: DashboardComponent,
})

function DashboardComponent() {
  // 從父層 layout 的 context 取得使用者資料
  const { user } = Route.useRouteContext()

  return (
    <div>
      <h1>Welcome, {user.email}!</h1>
      {/* 儀表板內容 */}
    </div>
  )
}
```

不需要再呼叫一次 `getCurrentUserFn()`——父層 layout 已經處理好了。

## 基於角色的路由保護

你可以擴展此模式來檢查角色：

```tsx
// routes/_authed/admin/index.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed/admin/')({
  beforeLoad: async ({ context }) => {
    // context.user 來自父層的 _authed layout
    if (context.user.role !== 'admin') {
      throw redirect({ to: '/unauthorized' })
    }
  },
  component: AdminPage,
})

function AdminPage() {
  return <h1>Admin Dashboard</h1>
}
```

這建立了一個**兩層式檢查**：
1. `_authed.tsx` 檢查使用者是否已登入
2. `admin/index.tsx` 檢查使用者是否為管理員

## 登入後導回原頁面

還記得我們先前儲存的 `search: { redirect: location.href }` 嗎？以下是如何在登入頁面中使用它：

```tsx
// routes/login.tsx
import { createFileRoute, useNavigate, useSearch } from '@tanstack/react-router'

export const Route = createFileRoute('/login')({
  component: LoginPage,
})

function LoginPage() {
  const navigate = useNavigate()
  const { redirect: redirectUrl } = useSearch({ from: '/login' })

  const handleLogin = async (data) => {
    await loginFn({ data })
    // 登入成功後，導回使用者原本要前往的頁面
    navigate({ to: redirectUrl || '/dashboard' })
  }

  return (
    <form onSubmit={handleLogin}>
      {/* 登入表單欄位 */}
    </form>
  )
}
```

## 重點整理

- 使用 **layout route**（`_authed.tsx`）來一次保護一整組頁面
- `beforeLoad` 在渲染之前執行——防止未授權內容的閃爍畫面
- `throw redirect(...)` 將未驗證的使用者導向登入頁面
- 在 `beforeLoad` 中透過 `return { user }` 將使用者資料傳遞給子路由
- 將原始 URL 儲存在 `search` 參數中，以便登入後能導回原頁面
- 若需要基於角色的保護，可在子路由中加入額外的 `beforeLoad` 檢查

---

下一篇：[Implementation Patterns](./07-implementation-patterns.md)
