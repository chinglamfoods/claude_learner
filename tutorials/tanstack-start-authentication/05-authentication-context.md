# 05：認證 Context

> 透過 React Context 將認證狀態傳遞至 UI 層，同時釐清 context 與 server 之間的安全職責分界。

## 架構定位

認證 context 在整體架構中的角色需要先釐清：**context 負責 UI 便利性，server 才是安全邊界**。Context 中的 `user` 物件僅用於決定渲染邏輯（顯示哪些按鈕、呈現哪些資訊），絕不應作為權限判斷的依據。任何敏感操作的授權檢查都必須在 server function 或 `beforeLoad` 中執行。

這個分界影響後續所有設計決策——context 的資料可以是 stale 的、可以被 client 竄改，但只要 server 端正確驗證，系統安全性不受影響。

## 實作

```tsx
// contexts/auth.tsx
import {
  createContext,
  useContext,
  useCallback,
  useMemo,
  type ReactNode,
} from 'react'
import { useServerFn } from '@tanstack/react-start'
import { getCurrentUserFn } from '../server/auth'

interface User {
  readonly id: string
  readonly email: string
  readonly role: 'admin' | 'member' | 'viewer'
}

interface AuthContextValue {
  /** null = 未認證，undefined = 尚未解析（SSR 或首次載入） */
  user: User | null
  isLoading: boolean
  /** 重新從 server 取得認證狀態。登入/登出後呼叫。 */
  refetch: () => Promise<void>
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined)

export function AuthProvider({ children }: { children: ReactNode }) {
  const {
    data: user,
    isLoading,
    refetch: _refetch,
  } = useServerFn(getCurrentUserFn)

  const refetch = useCallback(async () => {
    await _refetch()
  }, [_refetch])

  // 避免每次 render 都產生新的 object reference，減少不必要的 consumer re-render
  const value = useMemo<AuthContextValue>(
    () => ({ user: user ?? null, isLoading, refetch }),
    [user, isLoading, refetch],
  )

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
}

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext)
  if (ctx === undefined) {
    throw new Error(
      'useAuth() called outside of <AuthProvider>. ' +
        'Ensure AuthProvider wraps your router/app root.',
    )
  }
  return ctx
}
```

關鍵改動說明：

- `User` 欄位標記 `readonly`，防止 consumer 意外 mutate context 中的物件。
- `role` 使用 union type 而非 `string`，確保型別安全。
- `useMemo` 包裝 `value` 物件——若缺少此步驟，`AuthProvider` 每次 re-render 都會產生新的 object reference，導致所有 consumer 無條件 re-render，即使 `user` 和 `isLoading` 未變更。
- `refetch` 以 `useCallback` 穩定化，配合 `useMemo` 的 dependency array。

## 掛載 Provider

```tsx
// app.tsx
import { AuthProvider } from './contexts/auth'

function App() {
  return (
    <AuthProvider>
      <RouterProvider router={router} />
    </AuthProvider>
  )
}
```

## 使用範例

### 條件式渲染

```tsx
// components/Navbar.tsx
import { useAuth } from '../contexts/auth'

export function Navbar() {
  const { user, isLoading } = useAuth()

  return (
    <nav>
      <a href="/">首頁</a>
      {isLoading ? (
        <span aria-busy="true">載入中</span>
      ) : user ? (
        <>
          <span>{user.email}</span>
          <LogoutButton />
        </>
      ) : (
        <a href="/login">登入</a>
      )}
    </nav>
  )
}
```

### 登入/登出後的 Optimistic UI

登入或登出時，不需要等待 `refetch` 完成才更新 UI。常見做法是在觸發 server function 後立即進行路由導航或本地狀態更新，再讓 `refetch` 在背景同步真實狀態：

```tsx
import { useAuth } from '../contexts/auth'
import { useNavigate } from '@tanstack/react-router'
import { logoutFn } from '../server/auth'

function LogoutButton() {
  const { refetch } = useAuth()
  const navigate = useNavigate()

  const handleLogout = async () => {
    // 立即導航，不阻塞 UI
    void navigate({ to: '/login' })

    await logoutFn()
    // 背景同步 context 狀態
    await refetch()
  }

  return <button onClick={handleLogout}>登出</button>
}
```

注意：optimistic 導航意味著使用者在 `logoutFn` 實際完成前就已離開受保護頁面。若 logout 失敗，需要有 fallback 機制將使用者導回或顯示錯誤。

### 角色檢查

```tsx
function AdminSection() {
  const { user } = useAuth()

  if (user?.role !== 'admin') return null

  return (
    <section>
      <h2>管理區域</h2>
      {/* ... */}
    </section>
  )
}
```

再次強調：這僅控制 UI 可見性。實際的 admin API endpoint 必須在 server 端獨立驗證角色。

## 常見陷阱

### Stale auth state（Token refresh 後的過期狀態）

當 session 被 refresh（例如 token rotation）時，context 中的 `user` 物件不會自動更新。如果你的 session 機制有 sliding expiration 或 token refresh，需要在適當時機呼叫 `refetch()`。常見策略：

- **Window focus event**：使用者從其他 tab 切回時重新驗證。
- **API 401 response**：在 fetch wrapper 中偵測到 401 時觸發 `refetch()`。
- **定時輪詢**：對安全性要求較高的應用程式，可設定間隔重新檢查。

```tsx
import { useEffect } from 'react'

function AuthSyncOnFocus({ children }: { children: ReactNode }) {
  const { refetch } = useAuth()

  useEffect(() => {
    const onFocus = () => void refetch()
    window.addEventListener('focus', onFocus)
    return () => window.removeEventListener('focus', onFocus)
  }, [refetch])

  return <>{children}</>
}
```

### SSR Hydration mismatch

在 TanStack Start 的 SSR 流程中，server render 階段與 client hydration 階段的認證狀態可能不一致。典型情境：

1. Server 端 render 時 session 有效，產出已登入的 HTML。
2. Client hydration 時 `useServerFn` 重新發送請求，但此時 session 已過期（例如 cookie 在傳輸過程中 expire）。
3. Hydration 產出的 DOM 與 server HTML 不匹配，React 拋出 hydration error。

緩解方式：

- 在 `isLoading` 為 `true` 期間，render 與 server 端一致的 skeleton/placeholder，而非根據 `user` 值做條件渲染。
- 避免在首次 render 時就依據 `user` 做出關鍵的 DOM 結構差異——將差異延遲到 `useEffect` 之後。

### Context re-render 效能

所有呼叫 `useAuth()` 的元件在 `value` 變更時都會 re-render。若應用程式中有大量 consumer，考慮以下策略：

- **拆分 context**：將 `user` 和 `isLoading` 分離為獨立 context，讓只關心 `user` 的元件不受 loading 狀態切換影響。
- **Selector pattern**：搭配 `use-context-selector` 等 library，讓 consumer 只訂閱所需的部分值。
- **將頻繁變動的值移出 context**：例如 token expiration countdown 不應放在 auth context 中。

## SSR 與 Client 的狀態處理

在 TanStack Start 中，`useServerFn` 的行為在 SSR 與 client 階段不同：

- **SSR**：server function 在 server 端同步執行，可直接存取 cookie/session。render 結果包含初始認證狀態。
- **Client**：server function 透過 HTTP 呼叫 server 端，回傳結果後更新 context。

這代表從 server render 到 client hydration 存在一個短暫的時間差。在此期間，應避免基於認證狀態觸發 side effect（如 redirect）。等 client 端的 `useServerFn` resolve 後，才反映最新的認證狀態。

## 重點整理

- Auth context 的職責是 UI 狀態分發，不是安全機制——安全驗證一律在 server 端執行
- `useMemo` / `useCallback` 穩定 context value，避免不必要的 consumer re-render
- Token refresh 後 context 中的狀態可能過期，需搭配 window focus、401 偵測等機制主動同步
- SSR hydration mismatch 是常見問題，在 `isLoading` 期間應 render 穩定的 placeholder
- Optimistic UI 提升使用體驗，但需處理 server 操作失敗的 rollback 情境

---

下一篇：[路由保護](./06-route-protection.md)
