# 03：檔案命名慣例

> 路由檔案名稱中的特殊字元——`$`、`_`、`.`、`()`、`[]`、`-` 各自代表的意義與使用情境。

## 特殊字元一覽

| 字元 | 位置 | 功能 | 範例 |
|------|------|------|------|
| `$` | segment 開頭 | 動態路徑參數 | `$postId.tsx` → `/posts/:postId` |
| `$` | 單獨使用 | Splat / catch-all 路由 | `$.tsx` → 擷取所有剩餘路徑 |
| `_` | segment 前綴 | Pathless layout route（無路徑） | `_layout.tsx` → 不產生 URL segment |
| `_` | segment 後綴 | Non-nested route（脫離巢狀） | `posts_.tsx` → 不繼承 `posts` 的 layout |
| `.` | 檔案名稱中 | 表達路由巢狀層級 | `posts.index.tsx` → `/posts/` |
| `()` | 目錄名稱 | Route group（純組織用） | `(auth)/login.tsx` → `/login` |
| `[]` | segment 中 | 跳脫特殊字元 | `api[.]v1.tsx` → `/api.v1` |
| `-` | 檔案/目錄前綴 | 排除出路由樹 | `-utils.tsx` → 不產生路由 |
| `index` | 檔案名稱 | Index route（精確匹配父路由） | `posts/index.tsx` → `/posts`（精確） |
| `route` | 檔案名稱 | 目錄路由的設定檔 | `posts/route.tsx` → 等同 `posts.tsx` |

## `$` — 動態路徑參數

`$` 後接的標籤名稱會擷取對應的 URL segment 並放入 `params` 物件：

```
routes/
├── posts.$postId.tsx       → /posts/123  → params: { postId: "123" }
├── users.$userId.tsx       → /users/abc  → params: { userId: "abc" }
```

可在路由的 `loader`、`component` 等處存取：

```tsx
// src/routes/posts.$postId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
})

function PostComponent() {
  const { postId } = Route.useParams()
  return <div>Post: {postId}</div>
}
```

動態參數可以出現在路徑的**任何** segment。例如 `/posts/$postId/$revisionId` 會同時擷取 `postId` 與 `revisionId`。

## `$` — Splat（Catch-All）路由

僅包含 `$` 的路由檔案是 splat 路由，會擷取從 `$` 開始到路徑結尾的所有 segment。擷取的值放在 `params._splat` 中：

```
routes/
├── files/
│   └── $.tsx               → /files/docs/readme.md
                              → params: { _splat: "docs/readme.md" }
```

適用於檔案管理器、文件系統瀏覽器等需要匹配任意深度路徑的情境。

## `_` 前綴 — Pathless Layout Route

以 `_` 開頭的路由不會產生 URL segment，但其元件會包裹所有子路由。用於將共用的 layout 或邏輯套用到一組路由上：

```
routes/
├── _authenticated.tsx           → 不產生 URL segment
├── _authenticated.dashboard.tsx → /dashboard（被 _authenticated 包裹）
├── _authenticated.settings.tsx  → /settings（被 _authenticated 包裹）
├── login.tsx                    → /login（不被 _authenticated 包裹）
```

`_` 後面的名稱（如 `authenticated`）是路由的 ID，用於 TypeScript 型別識別與自動完成，但不會出現在 URL 中。

**限制：** Pathless layout route 不支援動態參數作為路徑的一部分：

```
routes/
├── _$postId/     ← ❌ 不允許
├── _postLayout/  ← ✅ 正確做法
```

## `_` 後綴 — Non-Nested Route

在 segment 結尾加上 `_` 可以讓路由**脫離**其父路由的巢狀結構，渲染獨立的元件樹：

```
routes/
├── posts.tsx                 → /posts         → <Posts>
├── posts.$postId.tsx         → /posts/123     → <Posts><Post>
├── posts_.$postId.edit.tsx   → /posts/123/edit → <PostEditor>
                                                  ↑ 不包裹在 <Posts> 中
```

典型用途：編輯頁面需要全螢幕 layout，不需要父路由的側邊欄或列表視圖。

## `.` — 巢狀分隔符號

檔案名稱中的 `.` 等同於目錄分隔符，用於表達路由巢狀：

```
posts.index.tsx     ≡  posts/index.tsx      → /posts（精確匹配）
posts.$postId.tsx   ≡  posts/$postId.tsx    → /posts/:postId
```

兩者可自由混用。選擇依據：路由數量少用 `.`（減少目錄），路由數量多用目錄（更好的組織性）。

## `()` — Route Group

以 `()` 包裹的目錄名稱是純組織用途，不影響路由路徑或元件樹：

```
routes/
├── (marketing)/
│   ├── pricing.tsx       → /pricing
│   ├── features.tsx      → /features
├── (dashboard)/
│   ├── analytics.tsx     → /analytics
│   ├── reports.tsx       → /reports
```

`(marketing)` 和 `(dashboard)` 目錄不會出現在 URL 中，僅用於檔案系統層級的分組。

## `[]` — 跳脫特殊字元

當路由路徑中需要包含 `.` 等在命名慣例中有特殊意義的字元時，使用 `[]` 跳脫：

```
routes/
├── script[.]js.tsx       → /script.js
├── api[.]v1.tsx          → /api.v1
```

## `-` 前綴 — 排除檔案

以 `-` 開頭的檔案或目錄會被排除出路由樹，不會出現在 `routeTree.gen.ts` 中。這讓你可以在路由目錄中**共置**輔助程式碼：

```
routes/
├── posts.tsx              → /posts（會產生路由）
├── -posts-table.tsx       → 排除（不產生路由）
├── -components/           → 排除（整個目錄）
│   ├── header.tsx
│   └── footer.tsx
```

```tsx
// src/routes/posts.tsx
import { PostsTable } from './-posts-table'
import { PostsHeader } from './-components/header'
// 從排除的檔案中 import，正常使用
```

## `{-$paramName}` — 可選路徑參數

可選路徑參數使用 `{-$paramName}` 語法，匹配時參數可能存在或不存在：

```
routes/
├── posts.{-$category}.tsx    → /posts 或 /posts/tech
```

```tsx
// src/routes/posts.{-$category}.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/{-$category}')({
  component: PostsComponent,
})

function PostsComponent() {
  const { category } = Route.useParams()
  // category: string | undefined
  return <div>{category ? `${category} 分類` : '所有文章'}</div>
}
```

可選參數的優先順序低於精確匹配。例如 `/posts/featured` 會先匹配 `posts.featured.tsx`（如果存在），才會匹配 `posts.{-$category}.tsx`。

## 重點整理

- `$` 擷取動態路徑參數，單獨使用時為 splat（catch-all）路由
- `_` 前綴建立 pathless layout route，`_` 後綴建立 non-nested route
- `.` 在檔案名稱中等同目錄分隔符，兩者可混用
- `()` 目錄是純組織用途，`-` 前綴的檔案被排除出路由樹
- `[]` 用於跳脫特殊字元，`{-$param}` 定義可選路徑參數
- 每個特殊字元都有明確的語義，組合使用時需注意順序（如 `posts_.$postId.edit.tsx`）

---

下一篇：[路由類型](./04-route-types.md)
