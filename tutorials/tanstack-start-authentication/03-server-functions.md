# 03: Server Functions for Authentication

> Learn how to use TanStack Start's server functions to handle login, logout, and user retrieval securely on the server.

## What Are Server Functions?

Server functions are functions that **only run on the server**, even though you call them from your React components. TanStack Start makes this seamless — you write the function, and the framework handles the client-server communication.

This is critical for authentication because:
- Passwords and credentials must never be exposed to the browser
- Database queries should only run server-side
- Session management needs server-side access to HTTP cookies

## Creating Server Functions with `createServerFn`

TanStack Start provides `createServerFn` to define server-only functions. Here's the basic pattern:

```tsx
import { createServerFn } from '@tanstack/react-start'

const myServerFunction = createServerFn({ method: 'POST' })
  .inputValidator((data: { someField: string }) => data)
  .handler(async ({ data }) => {
    // This code ONLY runs on the server
    // Safe to access databases, secrets, etc.
    return { result: 'something' }
  })
```

Let's break this down:
- **`method: 'POST'`** — the HTTP method used when the client calls this function (use POST for mutations like login/logout, GET for reads)
- **`.inputValidator()`** — validates and types the input data
- **`.handler()`** — the actual server-side logic

## Login Server Function

Here's a complete login function:

```tsx
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'

export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    // Step 1: Verify the user's credentials against your database
    const user = await authenticateUser(data.email, data.password)

    // Step 2: If credentials are wrong, return an error
    if (!user) {
      return { error: 'Invalid credentials' }
    }

    // Step 3: Create a session (stores user info in a secure cookie)
    const session = await useAppSession()
    await session.update({
      userId: user.id,
      email: user.email,
    })

    // Step 4: Redirect to the protected area
    throw redirect({ to: '/dashboard' })
  })
```

**Key points:**
- `authenticateUser` is your own function that checks the database (we'll build this in a later section)
- `useAppSession()` gives you access to the session (covered in the next section)
- `throw redirect(...)` sends the user to a new page after successful login — note it uses `throw`, not `return`

## Logout Server Function

Logging out is simpler — just clear the session:

```tsx
export const logoutFn = createServerFn({ method: 'POST' }).handler(
  async () => {
    // Clear all session data
    const session = await useAppSession()
    await session.clear()

    // Redirect to the home page
    throw redirect({ to: '/' })
  }
)
```

No input validation needed — there's nothing to validate when logging out.

## Get Current User Function

To check if someone is logged in and get their info:

```tsx
export const getCurrentUserFn = createServerFn({ method: 'GET' }).handler(
  async () => {
    // Read the session
    const session = await useAppSession()
    const userId = session.data.userId

    // If no userId in session, user is not logged in
    if (!userId) {
      return null
    }

    // Look up the full user record from the database
    return await getUserById(userId)
  }
)
```

**Why `method: 'GET'`?** This function doesn't modify anything — it just reads data. GET is appropriate for read-only operations.

## How It All Fits Together

```
User clicks "Login"
    ↓
React component calls loginFn({ email, password })
    ↓
TanStack Start sends a POST request to the server
    ↓
Server: validates credentials → creates session → redirects
    ↓
Browser: receives redirect → navigates to /dashboard
    ↓
Dashboard route: calls getCurrentUserFn() → gets user data → renders page
```

## Key Takeaways

- **Server functions** run only on the server, keeping credentials and database access secure
- Use `createServerFn` with `method: 'POST'` for login/logout (mutations) and `method: 'GET'` for reading user data
- `.inputValidator()` provides type-safe input validation
- Use `throw redirect(...)` to send users to a new page after auth actions
- `useAppSession()` provides access to the session store (next section covers this in detail)

---

Next: [Session Management](./04-session-management.md)
