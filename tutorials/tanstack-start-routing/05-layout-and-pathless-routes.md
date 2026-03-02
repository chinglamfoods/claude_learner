# 05：Layout 與 Pathless 路由

> Layout routes 包裹子路由提供共用 UI，pathless layout routes 提供不影響 URL 的 layout 包裹，non-nested routes 脫離父路由的巢狀結構。

## Layout Routes

Layout route 有自己的 URL segment，其元件透過 `<Outlet />` 渲染子路由。典型用途包含共用導覽列、側邊欄、以及資料載入邏輯：

```
routes/
├── app.tsx                  ← /app 的 layout route
├── app.dashboard.tsx        ← /app/dashboard
├── app.settings.tsx         ← /app/settings
```

```tsx
// src/routes/app.tsx
import { Outlet, createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/app')({
  component: AppLayoutComponent,
})

function AppLayoutComponent() {
  return (
    <div className="app-shell">
      <nav>
        <Link to="/app/dashboard">Dashboard</Link>
        <Link to="/app/settings">Settings</Link>
      </nav>
      <main>
        <Outlet />
      </main>
    </div>
  )
}
```

| URL | 渲染結果 |
|-----|---------|
| `/app` | `<AppLayout>`（`<Outlet />` 渲染 `null` 或 index route）|
| `/app/dashboard` | `<AppLayout><Dashboard>` |
| `/app/settings` | `<AppLayout><Settings>` |

也可以用目錄結構表達：

```
routes/
├── app/
│   ├── route.tsx            ← /app 的 layout 設定
│   ├── dashboard.tsx        ← /app/dashboard
│   └── settings.tsx         ← /app/settings
```

### Layout Route 搭配動態參數

Layout route 也支援動態參數 segment，常用於需要在多個子頁面共享特定資源的場景：

```
routes/
├── app/users/
│   ├── $userId/
│   │   ├── route.tsx        ← /app/users/:userId 的 layout
│   │   ├── index.tsx        ← /app/users/:userId（精確匹配）
│   │   └── edit.tsx         ← /app/users/:userId/edit
```

```tsx
// src/routes/app/users/$userId/route.tsx
import { Outlet, createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/app/users/$userId')({
  // 在 layout 層級載入使用者資料，子路由不需重複載入
  loader: ({ params }) => fetchUser(params.userId),
  component: UserLayoutComponent,
})

function UserLayoutComponent() {
  const user = Route.useLoaderData()
  return (
    <div>
      <h2>{user.name}</h2>
      <Outlet />
    </div>
  )
}
```

Layout route 的 `loader` 會在任何子路由匹配時執行，確保子路由可以安全存取父路由已載入的資料。

## Pathless Layout Routes

Pathless layout routes 以 `_` 前綴命名，**不產生 URL segment**，但其元件會包裹所有子路由。用於將共用的 layout 或邏輯套用到一組不同路徑的路由上：

```
routes/
├── _authenticated.tsx              ← 不產生 URL segment
├── _authenticated.dashboard.tsx    ← /dashboard
├── _authenticated.settings.tsx     ← /settings
├── login.tsx                       ← /login
```

```tsx
// src/routes/_authenticated.tsx
import { Outlet, createFileRoute, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_authenticated')({
  // 在所有子路由載入前檢查認證狀態
  beforeLoad: async ({ context }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({ to: '/login' })
    }
  },
  component: AuthenticatedLayout,
})

function AuthenticatedLayout() {
  return (
    <div className="authenticated-shell">
      <Sidebar />
      <main>
        <Outlet />
      </main>
    </div>
  )
}
```

| URL | 渲染結果 |
|-----|---------|
| `/dashboard` | `<AuthenticatedLayout><Dashboard>` |
| `/settings` | `<AuthenticatedLayout><Settings>` |
| `/login` | `<Login>`（不經過 `_authenticated`）|

**關鍵差異：** `/dashboard` 的 URL 中沒有 `authenticated` 這個 segment。pathless layout route 只影響元件樹和路由邏輯，不影響 URL 結構。

### `_` 後面的名稱

`_` 後面的名稱是路由的唯一 ID，用於：
- TypeScript 型別識別
- 開發者工具中的路由辨識
- 程式碼中的路由引用

目錄形式也可以使用：

```
routes/
├── _authenticated/
│   ├── route.tsx
│   ├── dashboard.tsx
│   └── settings.tsx
```

### 限制

Pathless layout route **不能**包含動態參數作為其名稱的一部分：

```
routes/
├── _$postId/       ← ❌ 不允許
├── _postLayout/    ← ✅ 正確做法
```

因為 pathless route 不匹配 URL path segment，所以沒有可擷取動態參數的位置。

## Non-Nested Routes

在 segment 名稱的結尾加上 `_` 後綴，可以讓路由脫離其父路由的巢狀結構，渲染獨立的元件樹：

```
routes/
├── posts.tsx                    ← /posts 的 layout
├── posts.$postId.tsx            ← /posts/:postId（巢狀在 posts 下）
├── posts_.$postId.edit.tsx      ← /posts/:postId/edit（脫離 posts 的 layout）
```

| URL | 渲染結果 |
|-----|---------|
| `/posts` | `<Posts>` |
| `/posts/123` | `<Posts><Post>` |
| `/posts/123/edit` | `<PostEditor>`（直接在 `<Root>` 下，不包裹 `<Posts>`）|

```tsx
// src/routes/posts_.$postId.edit.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId/edit')({
  component: PostEditorComponent,
})

function PostEditorComponent() {
  const { postId } = Route.useParams()
  // 全螢幕編輯器，不需要 posts 列表的側邊欄
  return <FullscreenEditor postId={postId} />
}
```

**典型用途：**
- 編輯頁面需要全螢幕 layout
- 列印預覽頁面不需要導覽列
- Modal 或 overlay 需要獨立的元件樹

**命名模式拆解：** `posts_.$postId.edit.tsx`
- `posts_` — `posts` segment 加上 `_` 後綴，表示脫離 `posts.tsx` 的巢狀
- `.$postId` — 動態參數 segment
- `.edit` — 靜態 segment

URL 路徑仍然是 `/posts/:postId/edit`，只是元件樹不經過 `posts.tsx` 的 layout。

## 三者比較

| 類型 | 影響 URL | 包裹子路由 | 使用情境 |
|------|---------|-----------|---------|
| Layout route | 是 | 是 | 共用 UI 與邏輯的路由群組 |
| Pathless layout route | 否 | 是 | 認證檢查、主題切換等不影響 URL 的 layout |
| Non-nested route | 是（路徑不變） | 否（脫離父 layout） | 全螢幕編輯、列印預覽等需要獨立 layout 的頁面 |

## 重點整理

- Layout route 有 URL segment，透過 `<Outlet />` 渲染子路由，子路由繼承其 layout 與 loader 資料
- Pathless layout route 以 `_` 前綴命名，不影響 URL，用於跨路徑的共用 layout 與邏輯
- Non-nested route 以 `_` 後綴脫離父路由巢狀，渲染獨立元件樹但 URL 路徑不變
- Pathless layout route 不能使用動態參數作為名稱的一部分
- 三種路由類型可以組合使用，建構出靈活的元件樹與 URL 結構

---

下一篇：[路由組織與共置](./06-route-organization.md)
