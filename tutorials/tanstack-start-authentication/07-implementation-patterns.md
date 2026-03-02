# 07: Implementation Patterns

> Practical implementations for email/password auth, role-based access control, social login, and password reset flows.

## Basic Email/Password Authentication

This is the most common authentication pattern. Let's build it step by step.

### User Registration

```tsx
// server/auth.ts
import bcrypt from 'bcryptjs'
import { createServerFn } from '@tanstack/react-start'

export const registerFn = createServerFn({ method: 'POST' })
  .inputValidator(
    (data: { email: string; password: string; name: string }) => data,
  )
  .handler(async ({ data }) => {
    // Step 1: Check if a user with this email already exists
    const existingUser = await getUserByEmail(data.email)
    if (existingUser) {
      return { error: 'User already exists' }
    }

    // Step 2: Hash the password before storing it
    // NEVER store plain-text passwords!
    // The "12" is the salt rounds — higher = more secure but slower
    const hashedPassword = await bcrypt.hash(data.password, 12)

    // Step 3: Save the user to your database
    const user = await createUser({
      email: data.email,
      password: hashedPassword,
      name: data.name,
    })

    // Step 4: Automatically log them in by creating a session
    const session = await useAppSession()
    await session.update({ userId: user.id })

    return { success: true, user: { id: user.id, email: user.email } }
  })
```

**Why `bcrypt`?**
- It's a one-way hash — you can't reverse it to get the original password
- The salt rounds (12) make brute-force attacks extremely slow
- Each password gets a unique random salt, so identical passwords produce different hashes

### User Login (Credential Verification)

```tsx
async function authenticateUser(email: string, password: string) {
  // Look up the user by email
  const user = await getUserByEmail(email)
  if (!user) return null

  // Compare the provided password with the stored hash
  const isValid = await bcrypt.compare(password, user.password)
  return isValid ? user : null
}
```

`bcrypt.compare` handles the salt automatically — you don't need to extract or store it separately.

## Role-Based Access Control (RBAC)

RBAC lets you control what different types of users can do.

### Defining Roles and Permissions

```tsx
// utils/auth.ts

// Define your roles as constants
export const roles = {
  USER: 'user',
  ADMIN: 'admin',
  MODERATOR: 'moderator',
} as const

type Role = (typeof roles)[keyof typeof roles]

// Check if a user's role meets the required level
export function hasPermission(userRole: Role, requiredRole: Role): boolean {
  // Define a hierarchy: higher number = more permissions
  const hierarchy = {
    [roles.USER]: 0,
    [roles.MODERATOR]: 1,
    [roles.ADMIN]: 2,
  }

  return hierarchy[userRole] >= hierarchy[requiredRole]
}
```

### Using RBAC in Route Protection

```tsx
// routes/_authed/admin/index.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'
import { hasPermission, roles } from '../../utils/auth'

export const Route = createFileRoute('/_authed/admin/')({
  beforeLoad: async ({ context }) => {
    // context.user is available from the _authed layout
    if (!hasPermission(context.user.role, roles.ADMIN)) {
      throw redirect({ to: '/unauthorized' })
    }
  },
})
```

## Social Authentication (OAuth)

OAuth lets users sign in with existing accounts (Google, GitHub, etc.).

### Setting Up OAuth Providers

```tsx
// server/oauth.ts
export const authProviders = {
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID!,
    redirectUri: `${process.env.APP_URL}/auth/google/callback`,
  },
  github: {
    clientId: process.env.GITHUB_CLIENT_ID!,
    redirectUri: `${process.env.APP_URL}/auth/github/callback`,
  },
}
```

### Initiating the OAuth Flow

```tsx
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'

export const initiateOAuthFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { provider: 'google' | 'github' }) => data)
  .handler(async ({ data }) => {
    const provider = authProviders[data.provider]

    // Generate a random state parameter for CSRF protection
    const state = generateRandomState()

    // Save the state in the session so we can verify it later
    const session = await useAppSession()
    await session.update({ oauthState: state })

    // Build the OAuth authorization URL and redirect the user
    const authUrl = generateOAuthUrl(provider, state)
    throw redirect({ href: authUrl })
  })
```

**The OAuth flow in brief:**
1. Your app redirects the user to Google/GitHub's login page
2. User logs in there and grants permission
3. Google/GitHub redirects back to your callback URL with a code
4. Your server exchanges the code for user info
5. You create/update the user in your database and start a session

### Why the `state` parameter?

The `state` is a random string you generate before redirecting. When the OAuth provider redirects back, you verify the `state` matches what you stored. This prevents CSRF attacks where someone could trick a user into logging in as the attacker.

## Password Reset Flow

A secure password reset involves two steps: requesting the reset and confirming it.

### Step 1: Request Password Reset

```tsx
export const requestPasswordResetFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string }) => data)
  .handler(async ({ data }) => {
    const user = await getUserByEmail(data.email)
    if (!user) {
      // IMPORTANT: Don't reveal whether the email exists!
      // Always return the same response to prevent email enumeration
      return { success: true }
    }

    // Generate a secure random token
    const token = generateSecureToken()

    // Token expires in 1 hour
    const expires = new Date(Date.now() + 60 * 60 * 1000)

    // Save the token in your database
    await savePasswordResetToken(user.id, token, expires)

    // Send the reset email with a link containing the token
    await sendPasswordResetEmail(user.email, token)

    return { success: true }
  })
```

### Step 2: Reset the Password

```tsx
export const resetPasswordFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { token: string; newPassword: string }) => data)
  .handler(async ({ data }) => {
    // Look up the token
    const resetToken = await getPasswordResetToken(data.token)

    // Check if token is valid and not expired
    if (!resetToken || resetToken.expires < new Date()) {
      return { error: 'Invalid or expired token' }
    }

    // Hash the new password and update the user
    const hashedPassword = await bcrypt.hash(data.newPassword, 12)
    await updateUserPassword(resetToken.userId, hashedPassword)

    // Delete the used token so it can't be reused
    await deletePasswordResetToken(data.token)

    return { success: true }
  })
```

**Security notes:**
- Tokens should be cryptographically random (use `crypto.randomBytes`)
- Always set an expiration time (1 hour is typical)
- Delete tokens after use
- Never reveal whether an email exists in your system

## Key Takeaways

- Always hash passwords with `bcrypt` (salt rounds of 12 is a good default)
- RBAC uses a role hierarchy to check permissions — simple and effective
- OAuth requires CSRF protection via the `state` parameter
- Password reset tokens should be random, time-limited, and single-use
- Never reveal whether an email exists in error messages

---

Next: [Security Best Practices](./08-security-best-practices.md)
