# 05：Router 設定與根元件

> 建立 `router.tsx` 與 `__root.tsx` — TanStack Start 專案的兩個必要檔案。

## 專案檔案結構

完成這一步後的檔案結構：

```
myApp/
├── src/
│   ├── routes/
│   │   └── __root.tsx      ← 應用程式根元件
│   ├── router.tsx           ← Router 設定
│   └── routeTree.gen.ts     ← 自動產生（首次執行後出現）
├── vite.config.ts
├── package.json
└── tsconfig.json
```

`routeTree.gen.ts` 是由 `tanstackStart()` plugin 自動產生的，不需手動建立。首次執行 `npm run dev` 時會自動生成。

## Router 設定（router.tsx）

```tsx
// src/router.tsx
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

export function getRouter() {
  const router = createRouter({
    routeTree,
    scrollRestoration: true,
  })

  return router
}
```

**為什麼用 `getRouter()` factory function 而不是直接 export router 實例？**

每個 SSR 請求都需要獨立的 router 實例，避免跨請求的狀態污染。如果直接 export 一個 singleton，所有請求會共用同一個 router 狀態（包括 loader 資料、navigation 狀態），導致資料洩漏。

### createRouter 常用選項

```tsx
const router = createRouter({
  routeTree,
  // 啟用捲動位置恢復（返回上一頁時回到原本的捲動位置）
  scrollRestoration: true,
  // 預設的 preload 策略：'intent' 在 hover/focus 時預載入
  defaultPreload: 'intent',
  // loader 資料的預設 stale time（毫秒），超過後會重新載入
  defaultPreloadStaleTime: 30_000,
})
```

## 根元件（__root.tsx）

`__root.tsx` 是整個應用程式的最外層元件，所有路由都會被包裹在其中。它負責：
- 定義 HTML document 結構（`<html>`、`<head>`、`<body>`）
- 設定全域 `<meta>` 標籤
- 提供 `<Outlet />` 作為子路由的渲染位置

```tsx
// src/routes/__root.tsx
/// <reference types="vite/client" />
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
      {
        charSet: 'utf-8',
      },
      {
        name: 'viewport',
        content: 'width=device-width, initial-scale=1',
      },
      {
        title: 'TanStack Start Starter',
      },
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

### 關鍵元件說明

| 元件 | 功能 |
|------|------|
| `<HeadContent />` | 渲染 `head()` 中定義的 `<meta>` 標籤。各路由可透過自己的 `head()` 覆寫或追加 |
| `<Scripts />` | 注入 hydration 與 client-side navigation 所需的 JavaScript。**必須放在 `<body>` 結尾處** |
| `<Outlet />` | 渲染當前匹配的子路由元件 |

### head() 的運作方式

`head()` 回傳的物件定義了該路由的 `<meta>` 標籤。在巢狀路由中，每一層路由都可以定義自己的 `head()`，TanStack Start 會自動合併：

- `meta` 陣列中的項目會由子路由覆寫（根據 `name` 或 `property` 屬性 dedup）
- `title` 由最深層的匹配路由決定
- `links`、`scripts` 等會累加

### `/// <reference types="vite/client" />`

這行 triple-slash directive 引入 Vite 的 client 型別定義，讓 TypeScript 認得 `import.meta.env` 等 Vite 特有的 API。

## 重點整理

- TanStack Start 有兩個必要檔案：`src/router.tsx`（router 設定）與 `src/routes/__root.tsx`（根元件）
- `getRouter()` 使用 factory function 模式，確保 SSR 時每個請求有獨立的 router 實例
- `routeTree.gen.ts` 由 plugin 自動產生，不需手動建立或編輯
- `__root.tsx` 定義了 HTML document 結構，`<HeadContent />` 與 `<Scripts />` 是必要元件
- `head()` 支援巢狀路由合併，子路由可覆寫父路由的 meta 定義

---

下一篇：[第一個路由與 Server Function](./06-first-route-and-server-functions.md)
