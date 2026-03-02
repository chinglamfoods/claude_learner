# 09: Testing Authentication

> How to write unit tests and integration tests for your authentication system.

## Why Test Authentication?

Authentication is critical infrastructure — if it breaks, users can't log in, or worse, unauthorized users might gain access. Testing ensures:

- Login works with valid credentials
- Login rejects invalid credentials
- Protected routes redirect unauthenticated users
- Session management works correctly
- Edge cases are handled (expired tokens, duplicate emails, etc.)

## Unit Testing Server Functions

Unit tests verify individual pieces of logic in isolation.

### Setting Up with Vitest

```tsx
// __tests__/auth.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { loginFn } from '../server/auth'

describe('Authentication', () => {
  // Reset the test database before each test
  beforeEach(async () => {
    await setupTestDatabase()
  })

  it('should login with valid credentials', async () => {
    // Arrange: create a test user first
    // (setupTestDatabase should seed this)

    // Act: attempt to login
    const result = await loginFn({
      data: { email: 'test@example.com', password: 'password123' },
    })

    // Assert: no error, user data returned
    expect(result.error).toBeUndefined()
    expect(result.user).toBeDefined()
  })

  it('should reject invalid credentials', async () => {
    const result = await loginFn({
      data: { email: 'test@example.com', password: 'wrongpassword' },
    })

    expect(result.error).toBe('Invalid credentials')
  })
})
```

### What to Test in Unit Tests

| Test case | What you're verifying |
|-----------|----------------------|
| Valid login | Correct credentials return user data |
| Invalid password | Wrong password returns error |
| Non-existent user | Unknown email returns error |
| Registration | New user is created and session started |
| Duplicate registration | Existing email returns error |
| Password hashing | Passwords are hashed, not stored plain |
| Session creation | Session data is set after login |
| Session clearing | Logout clears session data |

## Integration Testing

Integration tests verify that multiple parts of the system work together — routes, auth checks, redirects, and rendering.

### Testing Auth Flow with React Testing Library

```tsx
// __tests__/auth-flow.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import { RouterProvider, createMemoryHistory } from '@tanstack/react-router'
import { router } from '../router'

describe('Authentication Flow', () => {
  it('should redirect to login when accessing protected route', async () => {
    // Create an in-memory history starting at a protected route
    const history = createMemoryHistory()
    history.push('/dashboard')  // This is a protected route

    // Render the app
    render(<RouterProvider router={router} history={history} />)

    // The user should be redirected to the login page
    await waitFor(() => {
      expect(screen.getByText('Login')).toBeInTheDocument()
    })
  })

  it('should show dashboard after successful login', async () => {
    const history = createMemoryHistory()
    history.push('/login')

    render(<RouterProvider router={router} history={history} />)

    // Fill in and submit the login form
    fireEvent.change(screen.getByLabelText('Email'), {
      target: { value: 'test@example.com' },
    })
    fireEvent.change(screen.getByLabelText('Password'), {
      target: { value: 'password123' },
    })
    fireEvent.click(screen.getByText('Login'))

    // After login, should redirect to dashboard
    await waitFor(() => {
      expect(screen.getByText('Welcome')).toBeInTheDocument()
    })
  })
})
```

### What to Test in Integration Tests

| Test case | What you're verifying |
|-----------|----------------------|
| Protected route redirect | Unauthenticated → login page |
| Login → redirect | Successful login → protected page |
| Logout → redirect | Logout → public page |
| Role-based redirect | Non-admin → unauthorized page |
| Persist across navigation | Session stays active between pages |

## Testing Tips

### 1. Use a Test Database

Never run tests against your production or development database. Use an in-memory database (like SQLite) or a separate test database.

### 2. Seed Test Data

Create helper functions to set up known test users:

```tsx
async function setupTestDatabase() {
  // Clear all data
  await db.user.deleteMany()

  // Create a test user with a known password
  await db.user.create({
    data: {
      email: 'test@example.com',
      password: await bcrypt.hash('password123', 12),
      name: 'Test User',
      role: 'user',
    },
  })
}
```

### 3. Mock External Services

If you use OAuth providers, mock them in tests:

```tsx
// Mock the OAuth token exchange
vi.mock('../server/oauth', () => ({
  exchangeCodeForToken: vi.fn().mockResolvedValue({
    access_token: 'mock-token',
    user: { email: 'oauth@example.com' },
  }),
}))
```

### 4. Test Edge Cases

Don't just test the happy path. Test:
- Expired sessions
- Malformed input
- Concurrent login attempts
- Network errors during auth

## Key Takeaways

- Unit tests verify individual auth functions (login, register, session management)
- Integration tests verify the full flow (navigate → redirect → login → access)
- Always use a separate test database with seeded test data
- Mock external services (OAuth providers, email sending)
- Test both happy paths and edge cases (expired tokens, invalid input)
- Use `vitest` + `@testing-library/react` as your testing stack

---

Next: [Common Patterns and Production](./10-common-patterns-and-production.md)
