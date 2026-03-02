# 04: Session Management

> Learn how to set up and configure secure sessions using TanStack Start's built-in session management with HTTP-only cookies.

## What is a Session?

A **session** is how your server remembers who a user is across multiple requests. Here's the flow:

1. User logs in successfully
2. Server creates a session and stores it in an **HTTP-only cookie**
3. Browser automatically sends this cookie with every subsequent request
4. Server reads the cookie to know who the user is

Think of it like a wristband at a concert — once you get it at the gate (login), you can access all the areas (pages) without showing your ticket again.

## Setting Up Sessions with `useSession`

TanStack Start provides a `useSession` function from `@tanstack/react-start/server`. Here's how to set it up:

```tsx
// utils/session.ts
import { useSession } from '@tanstack/react-start/server'

// Define the shape of your session data
type SessionData = {
  userId?: string
  email?: string
  role?: string
}

// Create a reusable function to access the session
export function useAppSession() {
  return useSession<SessionData>({
    // The name of the cookie
    name: 'app-session',

    // A secret key used to encrypt the cookie (at least 32 characters!)
    password: process.env.SESSION_SECRET!,

    // Cookie security settings
    cookie: {
      secure: process.env.NODE_ENV === 'production', // HTTPS only in production
      sameSite: 'lax',    // Protects against CSRF attacks
      httpOnly: true,      // Cannot be read by JavaScript (XSS protection)
    },
  })
}
```

Let's break down each setting:

### `name`
The name of the cookie stored in the browser. You can call it whatever you want — `'app-session'` is a common choice.

### `password`
A secret string used to **encrypt** the session data. This ensures that even though the cookie is stored in the browser, its contents can't be read or tampered with.

> **Important:** This must be at least 32 characters long. Store it in an environment variable, never in your code!

Example `.env` file:
```
SESSION_SECRET=at-least-32-characters-long-random-string-here
```

### `cookie` options

| Option | What it does |
|--------|-------------|
| `secure: true` | Cookie only sent over HTTPS (set to `true` in production) |
| `sameSite: 'lax'` | Browser only sends the cookie for same-site requests and top-level navigations — prevents CSRF attacks |
| `httpOnly: true` | Cookie cannot be accessed by JavaScript (`document.cookie`) — prevents XSS attacks |

## Reading Session Data

Once a session exists, you can read from it in any server function:

```tsx
const session = await useAppSession()

// Access session data
const userId = session.data.userId    // string | undefined
const email = session.data.email      // string | undefined
const role = session.data.role        // string | undefined
```

## Writing Session Data

To store data in the session (e.g., after login):

```tsx
const session = await useAppSession()

// Update session with user info
await session.update({
  userId: user.id,
  email: user.email,
  role: user.role,
})
```

The `update` method replaces the session data with what you provide. The encrypted cookie is automatically sent back to the browser.

## Clearing a Session

To log a user out, clear the session:

```tsx
const session = await useAppSession()
await session.clear()
```

This removes all session data and effectively logs the user out.

## Session Expiration

You can control how long a session lasts with `maxAge`:

```tsx
export function useAppSession() {
  return useSession<SessionData>({
    name: 'app-session',
    password: process.env.SESSION_SECRET!,
    cookie: {
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      httpOnly: true,
      maxAge: 7 * 24 * 60 * 60, // 7 days in seconds
    },
  })
}
```

If you don't set `maxAge`, the cookie will be a **session cookie** — it gets deleted when the user closes their browser.

## Complete Example: Session in Action

Here's how everything connects — from login to accessing a protected page:

```tsx
// server/auth.ts
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'
import { useAppSession } from '../utils/session'

// Login: create a session
export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    const user = await authenticateUser(data.email, data.password)
    if (!user) return { error: 'Invalid credentials' }

    const session = await useAppSession()
    await session.update({ userId: user.id, email: user.email })

    throw redirect({ to: '/dashboard' })
  })

// Check auth: read the session
export const getCurrentUserFn = createServerFn({ method: 'GET' }).handler(
  async () => {
    const session = await useAppSession()
    if (!session.data.userId) return null
    return await getUserById(session.data.userId)
  }
)

// Logout: clear the session
export const logoutFn = createServerFn({ method: 'POST' }).handler(
  async () => {
    const session = await useAppSession()
    await session.clear()
    throw redirect({ to: '/' })
  }
)
```

## Key Takeaways

- Sessions let the server remember who a user is across requests
- TanStack Start uses encrypted HTTP-only cookies — secure by default
- `useSession` from `@tanstack/react-start/server` is the core API
- Always store your `SESSION_SECRET` in an environment variable (32+ characters)
- Use `session.update()` to save data, `session.data` to read, and `session.clear()` to log out
- Configure `cookie` options (`secure`, `sameSite`, `httpOnly`) for proper security

---

Next: [Authentication Context](./05-authentication-context.md)
