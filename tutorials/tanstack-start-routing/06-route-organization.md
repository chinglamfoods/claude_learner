# 06：路由組織與共置

> 使用 route groups、排除檔案、以及共置模式組織大型專案的路由結構。

## Route Group 目錄

以 `()` 包裹的目錄名稱是純組織用途，不影響路由路徑或元件樹。適合將大量路由分類管理：

```
routes/
├── index.tsx                    → /
├── (marketing)/
│   ├── pricing.tsx              → /pricing
│   ├── features.tsx             → /features
│   └── about.tsx                → /about
├── (dashboard)/
│   ├── analytics.tsx            → /analytics
│   ├── reports.tsx              → /reports
│   └── settings.tsx             → /settings
├── (auth)/
│   ├── login.tsx                → /login
│   └── register.tsx             → /register
```

`(marketing)`、`(dashboard)`、`(auth)` 不會出現在 URL 中。它們的唯一作用是讓開發者在檔案系統中快速辨識路由的功能分組。

**與 pathless layout route 的差異：** Route group 純粹是檔案系統層級的組織，不會產生任何 layout 元件或路由邏輯。Pathless layout route（`_` 前綴）則會建立實際的路由節點，可以附加 layout 元件、loader、`beforeLoad` 等。

## 排除檔案（`-` 前綴）

以 `-` 開頭的檔案和目錄會被路由產生器忽略，不會出現在 `routeTree.gen.ts` 中。這讓你可以在路由目錄中**共置**與路由相關的輔助程式碼：

```
routes/
├── posts.tsx
├── -posts-table.tsx              ← 不產生路由
├── -posts-table.test.tsx         ← 不產生路由
├── -components/                  ← 整個目錄被排除
│   ├── post-card.tsx
│   ├── post-filters.tsx
│   └── pagination.tsx
├── -hooks/
│   └── use-post-search.ts
```

在路由檔案中正常 import：

```tsx
// src/routes/posts.tsx
import { createFileRoute } from '@tanstack/react-router'
import { PostsTable } from './-posts-table'
import { PostCard } from './-components/post-card'
import { usePostSearch } from './-hooks/use-post-search'

export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
  component: PostsPage,
})

function PostsPage() {
  const posts = Route.useLoaderData()
  const { filtered, setQuery } = usePostSearch(posts)

  return (
    <div>
      <input onChange={(e) => setQuery(e.target.value)} />
      <PostsTable posts={filtered} />
    </div>
  )
}
```

### 共置的優勢

將路由專屬的元件、hooks、工具函式放在路由旁邊而非集中在 `src/components/` 中：

- **就近原則** — 修改路由時，相關程式碼就在旁邊，不需在目錄間跳轉
- **刪除路由時一併清理** — 刪除路由目錄就能移除所有相關程式碼，不會遺留孤兒檔案
- **作用域明確** — 只有該路由使用的元件就放在該路由旁邊，共用元件則放在 `src/components/`

## 大型專案的路由組織策略

### 按功能模組分組

結合 route group 與目錄式路由：

```
routes/
├── __root.tsx
├── index.tsx
├── (public)/                        ← 公開頁面
│   ├── pricing.tsx
│   ├── docs.tsx
│   └── docs.$slug.tsx
├── _authenticated.tsx               ← 認證 layout（pathless）
├── _authenticated/
│   ├── (workspace)/                 ← 工作區相關
│   │   ├── projects.tsx
│   │   ├── projects/
│   │   │   ├── index.tsx
│   │   │   ├── $projectId.tsx
│   │   │   └── -components/
│   │   │       └── project-card.tsx
│   │   └── teams.tsx
│   └── (settings)/                  ← 設定相關
│       ├── account.tsx
│       ├── billing.tsx
│       └── integrations.tsx
```

### 混合策略

對於中型專案，通常不需要完整的分組策略。混合使用即可：

```
routes/
├── __root.tsx
├── index.tsx
├── about.tsx
├── posts.tsx
├── posts/
│   ├── index.tsx
│   ├── $postId.tsx
│   └── -components/
│       └── post-card.tsx
├── _admin.tsx
├── _admin.users.tsx
├── _admin.analytics.tsx
├── settings.tsx
├── settings.profile.tsx
└── settings.notifications.tsx
```

簡單的路由用扁平式，有子路由的用目錄，需要共用 layout 的用 pathless route。不需要強迫所有路由都採用同一種模式。

## `[]` 跳脫字元的實務用途

當路由路徑需要包含 `.`、`$` 等特殊字元時使用 `[]` 跳脫：

```
routes/
├── script[.]js.tsx        → /script.js
├── api[.]v1.tsx           → /api.v1
├── feed[.]xml.tsx         → /feed.xml
├── sitemap[.]xml.tsx      → /sitemap.xml
```

常見於 SEO 相關路由（sitemap、RSS feed）或需要模擬靜態檔案路徑的場景。

## 路由檔案的結構慣例

每個路由檔案的標準結構：

```tsx
// 1. Imports
import { createFileRoute } from '@tanstack/react-router'
import { createServerFn } from '@tanstack/react-start'

// 2. Server functions（如果有）
const getData = createServerFn({ method: 'GET' })
  .handler(async () => { /* ... */ })

// 3. Route 定義（export const Route）
export const Route = createFileRoute('/path')({
  loader: () => getData(),
  component: PageComponent,
  head: () => ({
    meta: [{ title: 'Page Title' }],
  }),
})

// 4. Component 定義
function PageComponent() {
  const data = Route.useLoaderData()
  return <div>{/* ... */}</div>
}

// 5. 輔助元件（該路由專屬的小元件）
function SubComponent() {
  return <div>{/* ... */}</div>
}
```

將 `Route` export 放在元件定義之前是慣例，讓閱讀者先看到路由的設定（路徑、loader、head），再看實作細節。

## 重點整理

- `()` route group 是純檔案系統組織，不影響 URL 或元件樹
- `-` 前綴的檔案和目錄被排除出路由樹，用於共置路由專屬的輔助程式碼
- 共置策略讓路由相關的元件、hooks 就近管理，便於維護與清理
- 大型專案可組合使用 route group、pathless layout、目錄式路由來建構清晰的結構
- `[]` 跳脫字元用於路徑中包含 `.` 等特殊字元的場景
- 路由檔案的慣例結構：imports → server functions → Route export → components
