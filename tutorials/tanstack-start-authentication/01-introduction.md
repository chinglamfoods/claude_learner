# 01: Introduction to Authentication in TanStack Start

> A beginner-friendly overview of what TanStack Start offers for authentication, what you can build with it, and what you need to get started.

## What is TanStack Start?

TanStack Start is a full-stack React framework built on top of TanStack Router. It gives you server-side rendering (SSR), server functions, and a powerful routing system — all with end-to-end type safety.

When it comes to **authentication** (verifying who a user is) and **authorization** (controlling what they can do), TanStack Start provides the building blocks you need to implement secure, production-ready auth systems.

## What Can You Build?

With TanStack Start's authentication tools, you can:

- **Email/password login systems** — traditional sign-up and sign-in flows
- **Social logins** — "Sign in with Google/GitHub" via OAuth
- **Role-based access control (RBAC)** — restrict pages and actions based on user roles (admin, moderator, user)
- **Protected routes** — pages that require login to access
- **Session management** — secure cookie-based sessions that persist across requests
- **Password reset flows** — secure token-based password recovery

You can also integrate with third-party authentication providers like Clerk, WorkOS, Auth.js, Better Auth, or Supabase Auth instead of building everything from scratch.

## Authentication Approaches

TanStack Start supports several approaches:

| Approach | Best For | Trade-off |
|----------|----------|-----------|
| **DIY (build your own)** | Full control, custom requirements | More work, you handle security |
| **Partner solutions (Clerk, WorkOS)** | Enterprise features, fast setup | Vendor dependency, per-user pricing |
| **OSS libraries (Better Auth, Auth.js)** | Community-driven, self-hosted | Less support than hosted options |
| **Hosted services (Supabase, Auth0, Firebase)** | Managed infrastructure | Less customization |

## Prerequisites

Before starting this tutorial, you should have:

- **Basic React knowledge** — components, hooks, JSX
- **Node.js installed** (v18 or later recommended)
- **A TanStack Start project** — follow the [Quick Start guide](https://tanstack.com/start/latest/docs/framework/react/quick-start) if you don't have one yet
- **Basic understanding of HTTP** — requests, responses, cookies (helpful but not required)

## What This Tutorial Covers

This tutorial walks through the TanStack Start authentication documentation step by step:

1. **Introduction** (this file) — overview and prerequisites
2. **Core Concepts** — authentication vs. authorization and the architecture model
3. **Server Functions** — handling auth logic securely on the server
4. **Session Management** — setting up secure cookie-based sessions
5. **Authentication Context** — sharing auth state across your React app
6. **Route Protection** — blocking unauthorized access to pages
7. **Implementation Patterns** — email/password, RBAC, OAuth, password reset
8. **Security Best Practices** — password hashing, rate limiting, input validation
9. **Testing Authentication** — unit and integration tests
10. **Common Patterns and Production** — loading states, remember me, migration tips

## Source Reference

This tutorial is based on the official TanStack Start documentation:
- [Authentication Overview](https://tanstack.com/start/latest/docs/framework/react/guide/authentication-overview)
- [Authentication Guide](https://tanstack.com/start/latest/docs/framework/react/guide/authentication)

## Key Takeaways

- TanStack Start is a full-stack React framework with built-in tools for authentication
- You can build your own auth system or integrate with third-party providers
- The framework provides server functions, sessions, and route protection as core building blocks
- This tutorial will walk you through each of these concepts step by step

---

Next: [Core Concepts](./02-core-concepts.md)
