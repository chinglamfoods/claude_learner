# 01：TanStack Start 路由系統介紹

> TanStack Start 路由架構概覽——Router 設定、路由樹產生機制、以及巢狀路由的運作方式。

## 路由架構

TanStack Start 的路由系統完全建構於 [TanStack Router](https://tanstack.com/router) 之上。Start 並沒有另造路由機制，而是直接使用 TanStack Router 的全部功能，包含 file-based routing、型別安全的路徑參數、自動 code-splitting、以及 preloading。

核心組成：

| 元件 | 檔案位置 | 功能 |
|------|---------|------|
| Router 設定 | `src/router.tsx` | Router 實例建立與全域行為配置 |
| 根路由 | `src/routes/__root.tsx` | HTML document 結構與應用程式外殼 |
| 路由檔案 | `src/routes/*.tsx` | 各頁面的路由定義 |
| 路由樹 | `src/routeTree.gen.ts` | 自動產生，不需手動編輯 |

## Router 設定

`src/router.tsx` 定義 Router 的全域行為：

```tsx
// src/router.tsx
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

// 必須 export 一個 factory function，每次呼叫回傳新的 router 實例
// SSR 時每個請求需要獨立的 router，避免跨請求狀態污染
export function getRouter() {
  const router = createRouter({
    routeTree,
    scrollRestoration: true,
  })

  return router
}
```

可在此設定的全域選項包含：
- **preloading** — 預設的 preload 策略（`'intent'` 在 hover/focus 時觸發）
- **caching** — loader 資料的 stale time 設定
- **scrollRestoration** — 返回上一頁時恢復捲動位置

## 路由樹自動產生

`routeTree.gen.ts` 由 TanStack Start 的 Vite plugin 在 `npm run dev` 或 `npm run build` 時自動產生。這個檔案包含：

- 從 `src/routes/` 目錄結構推導出的完整路由樹
- TypeScript 型別工具，提供全路由系統的型別推導與自動完成

**不要手動編輯這個檔案。** 每次檔案結構變更時會自動重新產生。

## 巢狀路由

TanStack Router 使用巢狀路由（nested routing）將 URL 路徑對應到元件樹。路由的巢狀關係決定了元件的包裹結構。

以下列路由結構為例：

```
routes/
├── __root.tsx          → <Root>
├── posts.tsx           → <Posts>
├── posts.$postId.tsx   → <Post>
```

當 URL 為 `/posts/123` 時，元件樹為：

```
<Root>
  <Posts>
    <Post />      ← params.postId = "123"
  </Posts>
</Root>
```

每一層路由透過 `<Outlet />` 元件渲染下一層的子路由。如果沒有匹配的子路由，`<Outlet />` 渲染 `null`。

## 根路由的關鍵元件

根路由（`__root.tsx`）中有三個框架提供的必要元件：

| 元件 | 放置位置 | 功能 |
|------|---------|------|
| `<HeadContent />` | `<head>` 內 | 渲染 `head()` 中定義的 meta、title、link 標籤 |
| `<Outlet />` | `<body>` 內 | 渲染匹配的子路由元件 |
| `<Scripts />` | `<body>` 結尾 | 注入 hydration 與 client-side navigation 所需的 JavaScript |

```tsx
// src/routes/__root.tsx
import type { ReactNode } from 'react'
import {
  Outlet,
  createRootRoute,
  HeadContent,
  Scripts,
} from '@tanstack/react-router'

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { title: 'My App' },
    ],
  }),
  component: RootComponent,
})

function RootComponent() {
  return (
    <RootDocument>
      <Outlet />
    </RootDocument>
  )
}

function RootDocument({ children }: Readonly<{ children: ReactNode }>) {
  return (
    <html>
      <head>
        <HeadContent />
      </head>
      <body>
        {children}
        <Scripts />
      </body>
    </html>
  )
}
```

根路由沒有路徑、永遠匹配、永遠渲染。它是放置全域 layout（導覽列、footer）、全域 context provider、以及全域 error boundary 的位置。

## 本教學涵蓋的內容

1. **介紹**（本篇）— 路由架構概覽
2. **File-Based Routing 基礎** — 目錄式、扁平式、混合式路由結構
3. **檔案命名慣例** — 特殊字元 `$`、`_`、`.`、`()` 的意義
4. **路由類型** — 基本路由、Index 路由、動態路徑、Splat 路由、可選參數
5. **Layout 與 Pathless 路由** — Layout routes、pathless layout routes、non-nested routes
6. **路由組織與共置** — 排除檔案、路由群組、檔案共置模式

## 參考來源

- [TanStack Start Routing Guide](https://tanstack.com/start/latest/docs/framework/react/guide/routing)
- [TanStack Router - File-Based Routing](https://tanstack.com/router/latest/docs/routing/file-based-routing)
- [TanStack Router - Routing Concepts](https://tanstack.com/router/latest/docs/routing/routing-concepts)

## 重點整理

- TanStack Start 的路由系統完全由 TanStack Router 驅動
- `router.tsx` 設定全域行為，`__root.tsx` 定義 HTML document 結構
- `routeTree.gen.ts` 自動產生，不需手動編輯
- 巢狀路由透過 `<Outlet />` 層層渲染元件樹
- `<HeadContent />`、`<Outlet />`、`<Scripts />` 是根路由的三個必要元件

---

下一篇：[File-Based Routing 基礎](./02-file-based-routing-basics.md)
