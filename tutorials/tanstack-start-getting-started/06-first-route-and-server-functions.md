# 06：第一個路由與 Server Function

> 建立首頁路由，使用 `createServerFn` 實作伺服器端讀寫邏輯，完成一個可運作的全端計數器。

## 目標

建立一個首頁路由 (`/`)，功能包括：
- 從伺服器端的檔案讀取計數值
- 在頁面上顯示一個按鈕，點擊後透過 server function 將計數值 +1
- 完整示範 loader、server function、以及 client-server 互動的流程

[完成後的效果](https://stackblitz.com/github/tanstack/router/tree/main/examples/react/start-counter)

## 建立首頁路由

在 `src/routes/` 目錄下建立 `index.tsx`，對應 `/` 路徑：

```tsx
// src/routes/index.tsx
import * as fs from 'node:fs'
import { createFileRoute, useRouter } from '@tanstack/react-router'
import { createServerFn } from '@tanstack/react-start'

const filePath = 'count.txt'

async function readCount() {
  return parseInt(
    await fs.promises.readFile(filePath, 'utf-8').catch(() => '0'),
  )
}

const getCount = createServerFn({
  method: 'GET',
}).handler(() => {
  return readCount()
})

const updateCount = createServerFn({ method: 'POST' })
  .inputValidator((d: number) => d)
  .handler(async ({ data }) => {
    const count = await readCount()
    await fs.promises.writeFile(filePath, `${count + data}`)
  })

export const Route = createFileRoute('/')({
  component: Home,
  loader: async () => await getCount(),
})

function Home() {
  const router = useRouter()
  const state = Route.useLoaderData()

  return (
    <button
      type="button"
      onClick={() => {
        updateCount({ data: 1 }).then(() => {
          router.invalidate()
        })
      }}
    >
      Add 1 to {state}?
    </button>
  )
}
```

## 程式碼拆解

### Server Functions

`createServerFn` 建立只在伺服器端執行的函式。框架會自動將它轉換為 API endpoint，客戶端呼叫時透過 HTTP 請求傳遞資料。

```tsx
// GET — 無副作用的讀取操作
const getCount = createServerFn({
  method: 'GET',
}).handler(() => {
  // 這段程式碼只在伺服器端執行
  // 可以安全使用 Node.js API（fs、crypto 等）
  return readCount()
})
```

```tsx
// POST — 有副作用的寫入操作
const updateCount = createServerFn({ method: 'POST' })
  .inputValidator((d: number) => d)  // 驗證並定義輸入型別
  .handler(async ({ data }) => {
    // data 的型別由 inputValidator 推導為 number
    const count = await readCount()
    await fs.promises.writeFile(filePath, `${count + data}`)
  })
```

**`method` 的影響：**
- `GET` — 結果可被快取，瀏覽器可在 prefetch 時發送
- `POST` — 不會被快取，適合寫入操作

**`inputValidator` 的作用：**
- 驗證客戶端傳入的資料
- 同時作為 TypeScript 的型別定義來源 — `handler` 中 `data` 的型別由 validator 的回傳型別推導
- 在生產環境中可搭配 Zod 等函式庫做更嚴格的驗證

### File-based Routing

```tsx
export const Route = createFileRoute('/')({
  component: Home,
  loader: async () => await getCount(),
})
```

- `createFileRoute('/')` — 路徑字串 `'/'` 對應檔案位置 `src/routes/index.tsx`
- `loader` — 在路由匹配時自動執行，回傳值可透過 `Route.useLoaderData()` 取得
- `component` — 該路由要渲染的 React 元件

**Loader 的執行時機：**
- SSR 時在伺服器端執行
- Client-side navigation 時在客戶端執行（但 server function 的 handler 仍在伺服器端）
- 支援 preloading — 當使用者 hover 連結時可提前觸發

### Client 端互動

```tsx
function Home() {
  const router = useRouter()
  const state = Route.useLoaderData()

  return (
    <button
      type="button"
      onClick={() => {
        updateCount({ data: 1 }).then(() => {
          router.invalidate()
        })
      }}
    >
      Add 1 to {state}?
    </button>
  )
}
```

- `Route.useLoaderData()` — 取得 `loader` 的回傳值，型別自動推導
- `router.invalidate()` — 標記所有 loader 資料為 stale，觸發重新載入。這是 mutation 後更新 UI 的標準模式
- `updateCount({ data: 1 })` — 在客戶端呼叫 server function，框架自動發送 HTTP POST 請求

### Node.js API 的使用

```tsx
import * as fs from 'node:fs'
```

在 route 檔案中直接 import Node.js 模組是安全的。`tanstackStart()` plugin 的 code splitting 會確保 server function 的 handler（以及其 import 的 Node.js 模組）不會進入 client bundle。

但要注意：只有 `createServerFn` 的 `handler` 內的程式碼會被隔離。如果在 handler 外部直接使用 `fs`，該程式碼可能會被打包到 client 端。

## 啟動開發伺服器

```bash
npm run dev
```

開啟 `http://localhost:3000`，你會看到一個按鈕顯示目前的計數值。點擊按鈕後，計數值會透過 server function 寫入 `count.txt`，然後 `router.invalidate()` 觸發 loader 重新讀取最新值。

## 完成後的完整檔案結構

```
myApp/
├── src/
│   ├── routes/
│   │   ├── __root.tsx
│   │   └── index.tsx
│   ├── router.tsx
│   └── routeTree.gen.ts     ← 自動產生
├── vite.config.ts
├── package.json
├── tsconfig.json
└── count.txt                 ← 執行後自動建立
```

## 下一步

專案已可運作。接下來建議閱讀官方的 [Routing 指南](https://tanstack.com/start/latest/docs/framework/react/guide/routing) 學習：
- 巢狀路由與 layout routes
- 動態路徑參數（`$postId`）
- Search params 的型別安全處理
- 路由層級的資料載入與錯誤處理

## 重點整理

- `createServerFn` 將伺服器邏輯封裝為可從客戶端直接呼叫的函式，框架處理 RPC 序列化
- `method: 'GET'` 用於讀取、`method: 'POST'` 用於寫入，影響快取行為
- `inputValidator` 同時處理資料驗證與 TypeScript 型別推導
- `loader` 在路由匹配時自動執行，回傳值透過 `Route.useLoaderData()` 取得
- `router.invalidate()` 是 mutation 後更新 UI 的標準模式
- Server function 的 handler 內可安全使用 Node.js API，不會洩漏到 client bundle
