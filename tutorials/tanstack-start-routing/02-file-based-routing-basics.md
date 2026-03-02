# 02：File-Based Routing 基礎

> 三種路由組織方式——目錄式、扁平式、混合式，以及檔案路徑與 URL 的對應關係。

## File-Based Routing 的運作方式

TanStack Router 掃描 `src/routes/` 目錄中的檔案，依據檔案名稱與目錄結構自動產生路由樹。這帶來幾項關鍵優勢：

- **自動 code-splitting** — 每個路由檔案獨立打包，只有匹配的路由才會被載入
- **完整型別推導** — 路由路徑、參數、loader 資料全程型別安全
- **與 URL 結構一致** — 檔案結構直觀反映應用程式的 URL 階層

路由以 `.` 或 `/`（目錄）表示巢狀關係，兩者可自由混用。

## 目錄式路由（Directory Routes）

使用目錄結構表達路由階層，適合路由數量較多且有明確分組需求的情境：

| 檔案名稱 | 路由路徑 | 元件輸出 |
|---------|---------|---------|
| `__root.tsx` | | `<Root>` |
| `index.tsx` | `/`（精確匹配）| `<Root><RootIndex>` |
| `about.tsx` | `/about` | `<Root><About>` |
| `posts.tsx` | `/posts` | `<Root><Posts>` |
| `posts/index.tsx` | `/posts`（精確匹配）| `<Root><Posts><PostsIndex>` |
| `posts/$postId.tsx` | `/posts/$postId` | `<Root><Posts><Post>` |
| `settings.tsx` | `/settings` | `<Root><Settings>` |
| `settings/profile.tsx` | `/settings/profile` | `<Root><Settings><Profile>` |
| `settings/notifications.tsx` | `/settings/notifications` | `<Root><Settings><Notifications>` |

目錄結構讓具有共同父路由的子路由集中管理，檔案名稱也較短。

## 扁平式路由（Flat Routes）

使用 `.` 分隔符號在檔案名稱中表達巢狀關係，適合深度巢狀但每層只有少數路由的情境：

| 檔案名稱 | 路由路徑 | 元件輸出 |
|---------|---------|---------|
| `__root.tsx` | | `<Root>` |
| `index.tsx` | `/`（精確匹配）| `<Root><RootIndex>` |
| `about.tsx` | `/about` | `<Root><About>` |
| `posts.tsx` | `/posts` | `<Root><Posts>` |
| `posts.index.tsx` | `/posts`（精確匹配）| `<Root><Posts><PostsIndex>` |
| `posts.$postId.tsx` | `/posts/$postId` | `<Root><Posts><Post>` |
| `settings.tsx` | `/settings` | `<Root><Settings>` |
| `settings.profile.tsx` | `/settings/profile` | `<Root><Settings><Profile>` |
| `settings.notifications.tsx` | `/settings/notifications` | `<Root><Settings><Notifications>` |

所有路由檔案都在同一層目錄，適合小型專案或需要快速瀏覽所有路由的場景。

## 混合式路由（Mixed Routes）

實務上最常用的方式。對於路由數量較多的群組使用目錄，對於獨立路由使用扁平式：

```
routes/
├── __root.tsx
├── index.tsx
├── about.tsx
├── posts.tsx
├── posts/                        ← 目錄式：posts 下有多個子路由
│   ├── index.tsx
│   ├── $postId.tsx
│   └── $postId.edit.tsx
├── settings.tsx                  ← 扁平式：settings 下的路由較少
├── settings.profile.tsx
└── settings.notifications.tsx
```

混合式路由讓你在同一個專案中根據各群組的複雜度選擇最適合的組織方式。

## 建立路由檔案

路由檔案使用 `createFileRoute` 函式定義，並 export 為 `Route`：

```tsx
// src/routes/posts.$postId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  component: PostComponent,
})

function PostComponent() {
  const { postId } = Route.useParams()
  return <div>Post: {postId}</div>
}
```

**關於路徑字串：** `createFileRoute('/posts/$postId')` 中的路徑字串由 TanStack Router 的 Vite plugin **自動管理**。當你建立、移動或重新命名路由檔案時，這個字串會自動更新。你不需要手動維護它。

這個看似冗餘的設計是為了 TypeScript 型別推導——沒有這個路徑字串，TypeScript 無法得知目前的檔案對應哪個路由，也就無法推導出正確的 params、search params 等型別。

## `route.tsx` 目錄路由

當使用目錄組織路由時，可以用 `route.tsx` 作為該目錄的路由設定檔：

```
routes/
├── account/
│   ├── route.tsx            ← /account 的路由設定（等同 account.tsx）
│   ├── overview.tsx         ← /account/overview
│   └── settings.tsx         ← /account/settings
```

`account/route.tsx` 與根目錄的 `account.tsx` 效果相同，但將路由設定與其子路由放在同一個目錄中，組織更清晰。

## Virtual File Routes

如果預設的 file-based routing 結構無法滿足需求，可以使用 Virtual File Routes 自訂路由來源，同時保留 file-based routing 的效能優勢（自動 code-splitting、型別產生）。這在需要非標準檔案結構或從外部來源動態產生路由時特別有用。

## 重點整理

- File-based routing 有三種組織方式：目錄式、扁平式、混合式
- `.` 分隔符號在檔案名稱中表達巢狀關係，等同於目錄結構
- 混合式路由是實務上最常用的方式，根據各群組複雜度選擇最適合的結構
- `createFileRoute` 的路徑字串由 plugin 自動管理，用於 TypeScript 型別推導
- `route.tsx` 可用於目錄中的路由設定，替代同名的扁平路由檔案

---

下一篇：[檔案命名慣例](./03-file-naming-conventions.md)
