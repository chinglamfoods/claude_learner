# 05：認證 Context

> 學習如何使用 React Context 在整個 React 應用程式中共享認證狀態。

## 為什麼需要 Auth Context？

當你已經有了處理登入/登出和 session 管理的 server function 之後，你還需要一種方式將使用者的認證狀態分享給各個 React 元件。例如：

- 導覽列根據認證狀態顯示「登入」或「歡迎，Jane！」
- 設定頁面顯示使用者的電子郵件
- 根據使用者角色進行條件式渲染

React Context 提供了一種乾淨的方式，讓任何元件都能存取這些資料，而不需要透過每一層傳遞 props。

## 建立 Auth Context

以下是完整的實作：

```tsx
// contexts/auth.tsx
import { createContext, useContext, ReactNode } from 'react'
import { useServerFn } from '@tanstack/react-start'
import { getCurrentUserFn } from '../server/auth'

// 定義使用者物件的結構
type User = {
  id: string
  email: string
  role: string
}

// 定義 context 提供的內容
type AuthContextType = {
  user: User | null     // null 表示未登入
  isLoading: boolean    // 正在取得使用者資料時為 true
  refetch: () => void   // 重新檢查認證狀態的函式
}

// 以 undefined 作為預設值建立 context
// （如果在 provider 之外使用，會拋出錯誤）
const AuthContext = createContext<AuthContextType | undefined>(undefined)

// Provider 元件包裹你的應用程式，並提供認證狀態
export function AuthProvider({ children }: { children: ReactNode }) {
  // useServerFn 呼叫 getCurrentUserFn 並管理載入/錯誤狀態
  const { data: user, isLoading, refetch } = useServerFn(getCurrentUserFn)

  return (
    <AuthContext.Provider value={{ user, isLoading, refetch }}>
      {children}
    </AuthContext.Provider>
  )
}

// 自訂 hook，讓任何元件都能存取認證狀態
export function useAuth() {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}
```

讓我們逐一說明每個部分：

### `User` 型別

定義你的應用程式使用哪些使用者資料。你可以根據需求自訂，例如加入 `name`、`avatarUrl` 等欄位。

### `AuthContextType`

Context 提供的資料結構：
- `user` — 目前的使用者，未登入時為 `null`
- `isLoading` — server function 執行中時為 `true`
- `refetch` — 可以呼叫此函式重新檢查認證狀態（在登入/登出後很實用）

### `AuthProvider`

這個元件的功能：
1. 呼叫 `getCurrentUserFn`（你在上一節建立的 server function）
2. 透過 context 將結果提供給所有子元件

### `useAuth` hook

一個便利的 hook，功能如下：
1. 讀取 context 的值
2. 如果有人忘記用 `AuthProvider` 包裹應用程式，會拋出一個有幫助的錯誤訊息

## 使用 Auth Context

### 包裹你的應用程式

在應用程式的最頂層加入 `AuthProvider`：

```tsx
// app.tsx 或根佈局
import { AuthProvider } from './contexts/auth'

function App() {
  return (
    <AuthProvider>
      {/* 所有路由和元件放在這裡 */}
      <RouterProvider router={router} />
    </AuthProvider>
  )
}
```

### 在元件中讀取認證狀態

任何在 `AuthProvider` 內部的元件都可以使用 `useAuth` hook：

```tsx
// components/Navbar.tsx
import { useAuth } from '../contexts/auth'

function Navbar() {
  const { user, isLoading } = useAuth()

  if (isLoading) {
    return <nav>載入中...</nav>
  }

  return (
    <nav>
      <a href="/">首頁</a>
      {user ? (
        <>
          <span>歡迎，{user.email}！</span>
          <button onClick={handleLogout}>登出</button>
        </>
      ) : (
        <a href="/login">登入</a>
      )}
    </nav>
  )
}
```

### 檢查角色

```tsx
function AdminPanel() {
  const { user } = useAuth()

  if (user?.role !== 'admin') {
    return <p>你沒有權限檢視此頁面。</p>
  }

  return (
    <div>
      <h1>管理員面板</h1>
      {/* 僅限管理員的內容 */}
    </div>
  )
}
```

## 重要提醒

Auth Context 的做法很適合用於用戶端的狀態管理，但**不應該是你唯一的防線**。你也必須在伺服器端使用 `beforeLoad` 保護路由（下一節會介紹），並在 server function 中驗證認證。

Context 是為了**介面上的方便** — 顯示正確的按鈕、呈現使用者資訊。伺服器才是你執行**實際安全防護**的地方。

## 重點整理

- React Context 讓你可以與應用程式中的任何元件共享認證狀態
- `AuthProvider` 包裹你的應用程式，並在載入時取得目前的使用者
- `useAuth()` 讓元件可以存取 `user`、`isLoading` 和 `refetch`
- Context 是用於介面渲染的 — 務必在伺服器端也要執行安全防護
- 在登入或登出後呼叫 `refetch()` 來更新介面

---

下一篇：[路由保護](./06-route-protection.md)
