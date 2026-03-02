# 04：路由類型

> 基本路由、Index 路由、動態路由、Splat 路由、可選參數路由的定義方式與匹配行為。

## 基本路由（Static Routes）

匹配固定路徑，最直觀的路由類型：

```tsx
// src/routes/about.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/about')({
  component: AboutComponent,
})

function AboutComponent() {
  return <div>About</div>
}
```

| 檔案名稱 | 匹配路徑 |
|---------|---------|
| `about.tsx` | `/about` |
| `contact.tsx` | `/contact` |
| `settings.profile.tsx` | `/settings/profile` |

## Index 路由

Index 路由在父路由**精確匹配**且沒有子路由匹配時渲染。用於提供父路由的「預設內容」：

```tsx
// src/routes/posts.index.tsx
import { createFileRoute } from '@tanstack/react-router'

// 注意 createFileRoute 路徑的尾部斜線，用於標記 index route
export const Route = createFileRoute('/posts/')({
  component: PostsIndexComponent,
})

function PostsIndexComponent() {
  return <div>請選擇一篇文章</div>
}
```

**匹配行為：**

| URL | 匹配的路由 | 渲染結果 |
|-----|----------|---------|
| `/posts` | `posts.tsx` + `posts.index.tsx` | `<Posts><PostsIndex>` |
| `/posts/123` | `posts.tsx` + `posts.$postId.tsx` | `<Posts><Post>` |

當 URL 為 `/posts` 時，`posts.tsx`（layout）渲染 `<Posts>`，其 `<Outlet />` 渲染 `posts.index.tsx` 的元件。當 URL 為 `/posts/123` 時，`<Outlet />` 改為渲染 `posts.$postId.tsx`。

## 動態路由（Dynamic Routes）

以 `$` 開頭的 segment 會擷取 URL 中對應位置的值：

```tsx
// src/routes/posts.$postId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  // loader 接收 params，可用於資料載入
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
})

function PostComponent() {
  const { postId } = Route.useParams()
  const post = Route.useLoaderData()
  return (
    <article>
      <h1>{post.title}</h1>
      <p>ID: {postId}</p>
    </article>
  )
}
```

多層動態參數：

```
routes/
├── orgs.$orgId.repos.$repoId.tsx
```

URL `/orgs/acme/repos/widget` 產生 `params: { orgId: "acme", repoId: "widget" }`。

**注意：** 動態參數的值永遠是 `string`。如果需要其他型別（如 `number`），需在 loader 或元件中自行轉換與驗證。

## Splat 路由（Catch-All）

僅包含 `$` 的路由檔案擷取剩餘的所有路徑段落：

```tsx
// src/routes/files.$.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/files/$')({
  component: FileBrowserComponent,
})

function FileBrowserComponent() {
  const { _splat } = Route.useParams()
  // URL: /files/documents/2024/report.pdf
  // _splat: "documents/2024/report.pdf"

  const segments = _splat?.split('/') ?? []
  return (
    <div>
      <nav>
        {segments.map((seg, i) => (
          <span key={i}>{seg} / </span>
        ))}
      </nav>
    </div>
  )
}
```

Splat 路由通常用於：
- 檔案瀏覽器
- CMS 頁面路由（URL 結構不固定）
- 404 fallback 頁面

## 可選參數路由

使用 `{-$paramName}` 語法定義可有可無的路徑段落：

```tsx
// src/routes/posts.{-$category}.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/{-$category}')({
  component: PostsComponent,
  loader: ({ params }) => {
    // category 可能是 undefined
    return params.category
      ? fetchPostsByCategory(params.category)
      : fetchAllPosts()
  },
})

function PostsComponent() {
  const { category } = Route.useParams()
  const posts = Route.useLoaderData()

  return (
    <div>
      <h1>{category ? `${category} 分類` : '所有文章'}</h1>
      {/* ... */}
    </div>
  )
}
```

| URL | params |
|-----|--------|
| `/posts` | `{ category: undefined }` |
| `/posts/tech` | `{ category: "tech" }` |

多個可選參數可串接：`posts.{-$category}.{-$slug}.tsx` 匹配 `/posts`、`/posts/tech`、`/posts/tech/hello-world`。

**優先順序：** 可選參數路由的排序低於精確匹配。`/posts/featured` 優先匹配 `posts.featured.tsx`（如果存在），之後才考慮 `posts.{-$category}.tsx`。

## 路由匹配順序

當多個路由可能匹配同一個 URL 時，TanStack Router 依以下優先順序決定：

1. **精確匹配的靜態路由**（`/posts/featured`）
2. **動態參數路由**（`/posts/$postId`）
3. **可選參數路由**（`/posts/{-$category}`）
4. **Splat 路由**（`/posts/$`）

這個順序確保最具體的路由優先匹配，避免過於寬泛的路由攔截更精確的路徑。

## 重點整理

- 基本路由匹配固定路徑，Index 路由匹配父路由的精確路徑
- 動態參數用 `$paramName` 定義，值永遠是 `string`
- Splat 路由（`$.tsx`）擷取所有剩餘路徑至 `params._splat`
- 可選參數用 `{-$paramName}` 定義，匹配時參數可能為 `undefined`
- 匹配優先順序：靜態 > 動態 > 可選 > Splat

---

下一篇：[Layout 與 Pathless 路由](./05-layout-and-pathless-routes.md)
