# 02: Core Concepts

> Understand the difference between authentication and authorization, and learn the architecture behind TanStack Start's auth model.

## Authentication vs. Authorization

These two terms come up constantly in web development, so let's be clear about what each means:

- **Authentication** answers: *"Who is this user?"*
  - Logging in and logging out
  - Verifying identity (email/password, social login, etc.)

- **Authorization** answers: *"What can this user do?"*
  - Permissions and roles (admin, editor, viewer)
  - Access control to specific pages or actions

TanStack Start provides tools for **both** through server functions, sessions, and route protection. You'll typically implement authentication first, then layer authorization on top.

## Full-Stack Architecture Model

TanStack Start uses a full-stack architecture where authentication logic is split across three layers:

### Server-Side (Secure)

This is where sensitive operations happen. The server is the **only** place you should:

- Store and validate sessions
- Verify user credentials (passwords, tokens)
- Perform database operations (look up users, save sessions)
- Generate and verify tokens
- Run protected API endpoints

> **Why server-only?** Because client-side code is visible to anyone using browser DevTools. Passwords, database queries, and secret keys must never run in the browser.

### Client-Side (Public)

The client (browser) handles the user-facing parts:

- Managing authentication state in React (logged in or not)
- Showing/hiding UI based on auth status
- Login and logout forms
- Redirecting users to the right pages

### Isomorphic (Both)

Some code runs on both the server and client:

- Route loaders that check auth state (these run server-side during SSR, and may re-run client-side during navigation)
- Shared validation logic
- User profile data access

## Session Management Patterns

There are several ways to manage user sessions. Here's a comparison:

### HTTP-Only Cookies (Recommended)

```
Browser sends cookie automatically with every request
    ↓
Server reads cookie → validates session → returns response
```

- **Most secure** — cookies with `httpOnly: true` can't be read by JavaScript (protects against XSS attacks)
- **Automatic** — the browser handles sending cookies with every request
- **Built-in CSRF protection** with the `sameSite` attribute
- **Best for** traditional web applications (which is most apps!)

TanStack Start uses this approach by default.

### JWT Tokens

- Stateless — the token contains the user's info, no server-side storage needed
- Good for API-first applications
- **Caution:** storing JWTs in localStorage is vulnerable to XSS attacks
- Consider refresh token rotation for better security

### Server-Side Sessions

- Session data stored in a database or Redis
- Easy to revoke (just delete the session record)
- Requires a session store
- Good when you need to immediately invalidate sessions (e.g., "log out all devices")

## Route Protection Architecture

TanStack Start offers several patterns for protecting routes:

### Layout Route Pattern (Recommended)

Protect entire groups of pages by wrapping them in a **layout route** that checks authentication:

```
routes/
  _authed.tsx          ← checks if user is logged in
  _authed/
    dashboard.tsx      ← protected (child of _authed)
    settings.tsx       ← protected (child of _authed)
    admin/
      index.tsx        ← protected (child of _authed)
  login.tsx            ← public
  index.tsx            ← public
```

Any route inside `_authed/` automatically requires authentication because the parent layout checks it first.

### Component-Level Protection

Show or hide parts of a page based on auth status. Useful when a single page has both public and private content.

### Server Function Guards

Validate authentication on the server before executing sensitive operations. This is essential even if you have route-level protection — always verify on the server.

## Key Takeaways

- **Authentication** = who is the user; **Authorization** = what can they do
- Sensitive logic (passwords, sessions, DB queries) belongs on the **server only**
- HTTP-only cookies are the recommended session management approach
- Layout routes are the cleanest way to protect groups of pages
- Always validate auth on the server, even if the client also checks

---

Next: [Server Functions for Authentication](./03-server-functions.md)
