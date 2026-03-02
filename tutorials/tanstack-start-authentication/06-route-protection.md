# 06: Route Protection

> Learn how to protect routes so only authenticated users can access certain pages, using TanStack Router's `beforeLoad` hook.

## Why Protect Routes?

Without route protection, anyone could navigate to `/dashboard` or `/admin` by typing the URL directly — even if they're not logged in. Route protection ensures that:

- Unauthenticated users are redirected to the login page
- Authenticated users see the content they're allowed to see
- The check happens **before** the page renders (no flash of unauthorized content)

## The Layout Route Pattern

The most powerful pattern in TanStack Start is using a **layout route** to protect an entire group of pages at once.

### How It Works

In TanStack Router, routes starting with `_` are **layout routes**. They don't add a URL segment — they just wrap their children.

```
routes/
  _authed.tsx              ← layout route (checks auth)
  _authed/
    dashboard.tsx          ← /dashboard (protected)
    settings.tsx           ← /settings (protected)
    admin/
      index.tsx            ← /admin (protected)
  login.tsx                ← /login (public)
  index.tsx                ← / (public)
```

Any route inside the `_authed/` folder is automatically protected because `_authed.tsx` runs first.

### The Auth Check: `beforeLoad`

```tsx
// routes/_authed.tsx — Layout route for all protected pages
import { createFileRoute, redirect } from '@tanstack/react-router'
import { getCurrentUserFn } from '../server/auth'

export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    // Call the server to check if the user is logged in
    const user = await getCurrentUserFn()

    if (!user) {
      // Not logged in → redirect to login page
      // Save the current URL so we can redirect back after login
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      })
    }

    // User is logged in → pass their data to all child routes
    return { user }
  },
})
```

Let's break this down:

1. **`beforeLoad`** runs before the route component renders. This is where you check authentication.
2. **`getCurrentUserFn()`** calls your server function to check the session.
3. **`throw redirect(...)`** sends the user to `/login` if they're not authenticated. The `search: { redirect: location.href }` saves where they were trying to go, so you can redirect them back after login.
4. **`return { user }`** makes the user data available to all child routes via `Route.useRouteContext()`.

### Accessing User Data in Protected Routes

Child routes can access the user data that `_authed.tsx` provides:

```tsx
// routes/_authed/dashboard.tsx — A protected route
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed/dashboard')({
  component: DashboardComponent,
})

function DashboardComponent() {
  // Get the user from the parent layout's context
  const { user } = Route.useRouteContext()

  return (
    <div>
      <h1>Welcome, {user.email}!</h1>
      {/* Dashboard content */}
    </div>
  )
}
```

No need to call `getCurrentUserFn()` again — the parent layout already did it.

## Role-Based Route Protection

You can extend the pattern to check roles:

```tsx
// routes/_authed/admin/index.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_authed/admin/')({
  beforeLoad: async ({ context }) => {
    // context.user comes from the parent _authed layout
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

This creates a **two-level check**:
1. `_authed.tsx` checks if the user is logged in
2. `admin/index.tsx` checks if the user is an admin

## Redirect After Login

Remember the `search: { redirect: location.href }` we saved earlier? Here's how to use it in your login page:

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
    // After successful login, go back to where they were trying to go
    navigate({ to: redirectUrl || '/dashboard' })
  }

  return (
    <form onSubmit={handleLogin}>
      {/* Login form fields */}
    </form>
  )
}
```

## Key Takeaways

- Use **layout routes** (`_authed.tsx`) to protect entire groups of pages at once
- `beforeLoad` runs before rendering — prevents unauthorized content from flashing
- `throw redirect(...)` sends unauthenticated users to the login page
- Pass user data to child routes via `return { user }` in `beforeLoad`
- Save the original URL in `search` params so you can redirect back after login
- For role-based protection, add additional `beforeLoad` checks in child routes

---

Next: [Implementation Patterns](./07-implementation-patterns.md)
