# 05: Authentication Context

> Learn how to share authentication state across your entire React application using React Context.

## Why Do You Need an Auth Context?

Once you have server functions for login/logout and session management, you need a way to share the user's authentication state with your React components. For example:

- A navbar that shows "Login" or "Welcome, Jane!" depending on auth status
- A settings page that displays the user's email
- Conditional rendering based on user roles

React Context provides a clean way to make this data available to any component without passing props through every level.

## Creating the Auth Context

Here's the full implementation:

```tsx
// contexts/auth.tsx
import { createContext, useContext, ReactNode } from 'react'
import { useServerFn } from '@tanstack/react-start'
import { getCurrentUserFn } from '../server/auth'

// Define the shape of a user object
type User = {
  id: string
  email: string
  role: string
}

// Define what the context provides
type AuthContextType = {
  user: User | null     // null means not logged in
  isLoading: boolean    // true while fetching user data
  refetch: () => void   // function to re-check auth status
}

// Create the context with undefined as default
// (we'll throw an error if used outside the provider)
const AuthContext = createContext<AuthContextType | undefined>(undefined)

// The Provider component wraps your app and provides auth state
export function AuthProvider({ children }: { children: ReactNode }) {
  // useServerFn calls getCurrentUserFn and manages loading/error states
  const { data: user, isLoading, refetch } = useServerFn(getCurrentUserFn)

  return (
    <AuthContext.Provider value={{ user, isLoading, refetch }}>
      {children}
    </AuthContext.Provider>
  )
}

// Custom hook to access auth state from any component
export function useAuth() {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}
```

Let's walk through each piece:

### The `User` type

Defines what user data your app works with. Customize this based on your needs — you might add `name`, `avatarUrl`, etc.

### The `AuthContextType`

The shape of data the context provides:
- `user` — the current user, or `null` if not logged in
- `isLoading` — `true` while the server function is running
- `refetch` — a function you can call to re-check auth status (useful after login/logout)

### The `AuthProvider`

This component:
1. Calls `getCurrentUserFn` (your server function from the previous section)
2. Provides the result to all child components via context

### The `useAuth` hook

A convenience hook that:
1. Reads the context value
2. Throws a helpful error if someone forgets to wrap their app with `AuthProvider`

## Using the Auth Context

### Wrapping Your App

Add `AuthProvider` at the top level of your app:

```tsx
// app.tsx or root layout
import { AuthProvider } from './contexts/auth'

function App() {
  return (
    <AuthProvider>
      {/* All your routes and components go here */}
      <RouterProvider router={router} />
    </AuthProvider>
  )
}
```

### Reading Auth State in Components

Any component inside `AuthProvider` can use the `useAuth` hook:

```tsx
// components/Navbar.tsx
import { useAuth } from '../contexts/auth'

function Navbar() {
  const { user, isLoading } = useAuth()

  if (isLoading) {
    return <nav>Loading...</nav>
  }

  return (
    <nav>
      <a href="/">Home</a>
      {user ? (
        <>
          <span>Welcome, {user.email}!</span>
          <button onClick={handleLogout}>Logout</button>
        </>
      ) : (
        <a href="/login">Login</a>
      )}
    </nav>
  )
}
```

### Checking Roles

```tsx
function AdminPanel() {
  const { user } = useAuth()

  if (user?.role !== 'admin') {
    return <p>You don't have permission to view this page.</p>
  }

  return (
    <div>
      <h1>Admin Panel</h1>
      {/* Admin-only content */}
    </div>
  )
}
```

## Important Note

The Auth Context approach works well for client-side state management, but **it should not be your only line of defense**. Always also protect routes on the server using `beforeLoad` (covered in the next section) and validate auth in server functions.

Context is for **UI convenience** — showing the right buttons, displaying user info. The server is where you enforce **actual security**.

## Key Takeaways

- React Context lets you share auth state with any component in your app
- `AuthProvider` wraps your app and fetches the current user on load
- `useAuth()` gives components access to `user`, `isLoading`, and `refetch`
- Context is for UI rendering — always enforce security on the server too
- Call `refetch()` after login or logout to update the UI

---

Next: [Route Protection](./06-route-protection.md)
