# 08: Security Best Practices

> Essential security measures for your authentication system: password hashing, session security, rate limiting, and input validation.

## Why Security Matters

Authentication is the front door to your application. A weak implementation can lead to:
- Stolen user accounts
- Data breaches
- Legal liability (GDPR, CCPA violations)

The good news: following a few key practices makes your auth system significantly more secure.

## 1. Password Security

### Use Strong Hashing

Never store passwords as plain text. Use a purpose-built hashing algorithm:

```tsx
import bcrypt from 'bcryptjs'

// Hash a password before storing it
const saltRounds = 12 // Higher = more secure but slower
const hashedPassword = await bcrypt.hash(password, saltRounds)

// Verify a password during login
const isValid = await bcrypt.compare(providedPassword, storedHash)
```

**Why `bcrypt`?**
- Intentionally slow — makes brute-force attacks impractical
- Automatic salting — each hash is unique, even for the same password
- Configurable work factor — increase `saltRounds` as hardware gets faster

**Alternatives:** `scrypt` and `argon2` are also excellent choices. Argon2 is the winner of the Password Hashing Competition and is considered the most modern option.

### Password Requirements

At minimum, require:
- 8+ characters (NIST recommends supporting up to 64+)
- Check against known breached passwords (optional but recommended)

Avoid overly strict rules (must include uppercase, symbol, number) — research shows they lead to predictable patterns like `Password1!`.

## 2. Session Security

### Configure Cookies Correctly

```tsx
export function useAppSession() {
  return useSession({
    name: 'app-session',
    password: process.env.SESSION_SECRET!, // 32+ characters
    cookie: {
      secure: process.env.NODE_ENV === 'production', // HTTPS only in production
      sameSite: 'lax',    // CSRF protection
      httpOnly: true,      // XSS protection
      maxAge: 7 * 24 * 60 * 60, // 7 days
    },
  })
}
```

Here's what each setting protects against:

| Setting | Attack | How it helps |
|---------|--------|-------------|
| `secure: true` | Man-in-the-middle | Cookie only sent over encrypted HTTPS connections |
| `sameSite: 'lax'` | CSRF (Cross-Site Request Forgery) | Browser won't send cookie from third-party sites |
| `httpOnly: true` | XSS (Cross-Site Scripting) | JavaScript can't read the cookie via `document.cookie` |
| `maxAge` | Session hijacking | Limits how long a stolen cookie remains valid |

### Session Secret

Your `SESSION_SECRET` must be:
- **At least 32 characters** long
- **Randomly generated** (use a password generator)
- **Stored in environment variables** (never committed to git)
- **Rotated periodically** in production

Generate one with:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

## 3. Rate Limiting

Rate limiting prevents brute-force attacks by limiting how many login attempts can be made.

### Simple In-Memory Rate Limiter

```tsx
// Simple rate limiter (use Redis in production for multi-server setups)
const loginAttempts = new Map<string, { count: number; resetTime: number }>()

export const rateLimitLogin = (ip: string): boolean => {
  const now = Date.now()
  const attempts = loginAttempts.get(ip)

  // First attempt or window expired — allow and start new window
  if (!attempts || now > attempts.resetTime) {
    loginAttempts.set(ip, {
      count: 1,
      resetTime: now + 15 * 60 * 1000,  // 15-minute window
    })
    return true  // allowed
  }

  // Too many attempts in this window
  if (attempts.count >= 5) {
    return false  // blocked
  }

  // Increment and allow
  attempts.count++
  return true
}
```

### Using the Rate Limiter

```tsx
export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    // Check rate limit before processing login
    const clientIp = getClientIp()  // implement based on your server setup
    if (!rateLimitLogin(clientIp)) {
      return { error: 'Too many login attempts. Please try again later.' }
    }

    // ... proceed with authentication
  })
```

**Important:** The in-memory `Map` approach works for single-server deployments. For production systems with multiple servers, use Redis or a similar shared store.

## 4. Input Validation

Always validate input on the server. Client-side validation is for UX — it can be bypassed.

### Using Zod for Validation

```tsx
import { z } from 'zod'

// Define a strict schema for login data
const loginSchema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(8).max(100),
})

export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data) => loginSchema.parse(data))
  .handler(async ({ data }) => {
    // data is now guaranteed to be valid:
    // - email is a valid email format, max 255 chars
    // - password is 8-100 characters
  })
```

### Registration Validation

```tsx
const registerSchema = z.object({
  email: z.string().email().max(255),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .max(100),
  name: z
    .string()
    .min(1, 'Name is required')
    .max(100)
    .trim(),
})
```

### Why Validate on the Server?

- Client-side validation can be bypassed by sending requests directly (curl, Postman, browser DevTools)
- `.max()` limits prevent denial-of-service via extremely long strings
- `.email()` ensures proper format before querying the database
- Zod provides type-safe parsed output — your handler gets validated, typed data

## Additional Security Measures

### Timing-Safe Comparison

When checking passwords, `bcrypt.compare` is already timing-safe. But for other comparisons (like tokens), use:

```tsx
import { timingSafeEqual } from 'crypto'

function safeCompare(a: string, b: string): boolean {
  if (a.length !== b.length) return false
  return timingSafeEqual(Buffer.from(a), Buffer.from(b))
}
```

This prevents timing attacks where an attacker measures response time to guess characters.

### Don't Leak Information

```tsx
// BAD: reveals that the email exists in your system
return { error: 'Wrong password for user@example.com' }

// GOOD: generic message that doesn't reveal anything
return { error: 'Invalid credentials' }
```

### HTTPS in Production

Always use HTTPS in production. Without it:
- Cookies can be intercepted in transit
- Login credentials are sent in plain text
- Man-in-the-middle attacks become trivial

## Key Takeaways

- **Passwords**: Hash with bcrypt (12+ salt rounds), never store plain text
- **Sessions**: Use `httpOnly`, `secure`, and `sameSite` cookie settings
- **Rate limiting**: Limit login attempts per IP to prevent brute force
- **Validation**: Always validate on the server with a library like Zod
- **Don't leak info**: Use generic error messages, timing-safe comparisons
- **HTTPS**: Always in production — non-negotiable

---

Next: [Testing Authentication](./09-testing-authentication.md)
