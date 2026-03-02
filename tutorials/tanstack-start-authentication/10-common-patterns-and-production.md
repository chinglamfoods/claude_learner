# 10: Common Patterns and Production Considerations

> Loading states, "Remember Me" functionality, migration tips, and production readiness checklist.

## Loading States

Users need feedback while authentication operations are in progress. Without loading states, the UI feels broken.

### Login Form with Loading State

```tsx
function LoginForm() {
  const [isLoading, setIsLoading] = useState(false)
  const loginMutation = useServerFn(loginFn)

  const handleSubmit = async (data: LoginData) => {
    setIsLoading(true)
    try {
      await loginMutation.mutate(data)
    } catch (error) {
      // Handle error (show message to user)
    } finally {
      // Always reset loading state, even on error
      setIsLoading(false)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Email
        <input type="email" name="email" disabled={isLoading} />
      </label>

      <label>
        Password
        <input type="password" name="password" disabled={isLoading} />
      </label>

      <button disabled={isLoading}>
        {isLoading ? 'Logging in...' : 'Login'}
      </button>
    </form>
  )
}
```

**Best practices for loading states:**
- Disable form inputs and the submit button while loading
- Show a clear indicator (text change, spinner, etc.)
- Always reset loading state in `finally` (handles both success and error)

## Remember Me Functionality

"Remember Me" extends the session duration so users don't have to log in as frequently.

```tsx
export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator(
    (data: { email: string; password: string; rememberMe?: boolean }) => data,
  )
  .handler(async ({ data }) => {
    const user = await authenticateUser(data.email, data.password)
    if (!user) return { error: 'Invalid credentials' }

    const session = await useAppSession()
    await session.update(
      { userId: user.id },
      {
        // If "Remember Me" is checked: 30 days
        // If not: session cookie (deleted when browser closes)
        maxAge: data.rememberMe ? 30 * 24 * 60 * 60 : undefined,
      },
    )

    return { success: true }
  })
```

### On the Login Form

```tsx
<label>
  <input type="checkbox" name="rememberMe" />
  Remember me for 30 days
</label>
```

## Migration from Other Solutions

### From Client-Side Auth (localStorage, Context-only)

If you're currently storing auth state in `localStorage` or React Context without server-side validation:

1. **Move auth logic to server functions** — credential verification, token generation
2. **Replace localStorage with server sessions** — HTTP-only cookies are more secure
3. **Update route protection to use `beforeLoad`** — instead of client-side redirect logic
4. **Add CSRF protection** — `sameSite: 'lax'` cookies handle this

### From Next.js

- Replace **API routes** with TanStack Start's **server functions**
- Migrate **NextAuth.js sessions** to TanStack Start's `useSession`
- Update **middleware-based** route protection to `beforeLoad`

### From Remix

- Convert **loaders and actions** to server functions
- Adapt **session patterns** (Remix sessions → TanStack Start sessions)
- Update **route protection** patterns

## Hosted vs. DIY Decision Guide

Choosing between building your own auth and using a hosted service:

### Choose Hosted (Clerk, WorkOS) When:

- You need **enterprise features** fast (SSO, SCIM, compliance)
- You want **pre-built UI components** (login forms, user management)
- You need **managed security updates** and monitoring
- Your budget allows per-user or subscription pricing
- You're a small team that can't dedicate time to auth maintenance

### Choose DIY When:

- You need **complete control** over the auth flow and user data
- You have **custom business logic** requirements
- You want to **avoid vendor lock-in**
- You have a team that can maintain and monitor the auth system
- **Cost control** is important (no per-user pricing)

## Production Checklist

Before deploying your authentication system to production, verify:

### Security
- [ ] Passwords hashed with bcrypt/scrypt/argon2 (12+ salt rounds)
- [ ] Session secret is 32+ random characters, stored in env variable
- [ ] Cookies set with `secure: true`, `sameSite: 'lax'`, `httpOnly: true`
- [ ] HTTPS enabled (non-negotiable for production)
- [ ] Rate limiting on login endpoints
- [ ] Input validation on all server functions (use Zod)
- [ ] Generic error messages (don't reveal if emails exist)

### Functionality
- [ ] Login, logout, and registration work correctly
- [ ] Protected routes redirect unauthenticated users
- [ ] Session persists across page reloads
- [ ] Session expires after the configured `maxAge`
- [ ] Password reset flow works end-to-end

### Monitoring
- [ ] Log authentication events (login, logout, failed attempts)
- [ ] Monitor for unusual patterns (many failed logins from one IP)
- [ ] Set up alerts for auth system errors

### Compliance
- [ ] Handle personal data per GDPR/CCPA if applicable
- [ ] Provide account deletion if required
- [ ] Document your data retention policies

## Working Examples

Study these official TanStack examples for reference implementations:

- **[Basic Auth with Prisma](https://github.com/TanStack/router/tree/main/examples/react/start-basic-auth)** — Complete DIY implementation with database and sessions
- **[Supabase Integration](https://github.com/TanStack/router/tree/main/examples/react/start-supabase-basic)** — Third-party service integration
- **[Clerk Integration](https://github.com/TanStack/router/tree/main/examples/react/start-clerk-basic)** — Partner solution with pre-built UI
- **[WorkOS Integration](https://github.com/TanStack/router/tree/main/examples/react/start-workos)** — Enterprise authentication
- **[Client-side Context Auth](https://github.com/TanStack/router/tree/main/examples/react/authenticated-routes)** — Client-only patterns

## Key Takeaways

- Always show loading states during auth operations — disable forms and show feedback
- "Remember Me" is simply a longer `maxAge` on the session cookie
- Migrating from other frameworks mainly involves moving logic to server functions
- Choose hosted auth for speed and enterprise features; DIY for control and cost savings
- Use the production checklist before deploying — it covers security, functionality, and compliance
- Study the official examples to see complete, working implementations

---

This concludes the TanStack Start Authentication tutorial! Go back to the [Introduction](./01-introduction.md) to review the full table of contents.
