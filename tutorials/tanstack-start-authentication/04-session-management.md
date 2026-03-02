# 04：Session 管理

> 使用 TanStack Start 的 `useSession` API 實作加密 cookie-based session，涵蓋安全設定、secret rotation 以及水平擴展下的 session 策略。

## 使用 `useSession` 設定 Session

TanStack Start 透過 `@tanstack/react-start/server` 提供的 `useSession` 函式，以加密 cookie 實現 session 管理。以下是標準的封裝方式：

```tsx
// utils/session.ts
import { useSession } from '@tanstack/react-start/server'

type SessionData = {
  userId?: string
  email?: string
  role?: string
}

export function useAppSession() {
  return useSession<SessionData>({
    name: 'app-session',
    password: process.env.SESSION_SECRET!,
    cookie: {
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      httpOnly: true,
      maxAge: 7 * 24 * 60 * 60, // 7 天
    },
  })
}
```

### 設定項說明

**`name`** — cookie 名稱。避免使用預設值或容易猜測的名稱（如 `session`），以降低 targeted attack 的風險。

**`password`** — 用於加密 cookie 內容的 secret。長度必須至少 32 個字元，務必透過環境變數注入：

```
SESSION_SECRET=at-least-32-characters-long-random-string-here
```

> **Session secret rotation：** 在正式環境中，定期輪換 secret 是必要的。`useSession` 的 `password` 參數接受陣列格式——將新 secret 放在第一個位置，舊 secret 保留在後方。框架會嘗試以陣列中的每個 secret 依序解密，但新 session 一律使用第一個 secret 加密。這確保了 rotation 期間既有 session 不會立即失效：
>
> ```tsx
> password: [
>   process.env.SESSION_SECRET_CURRENT!,
>   process.env.SESSION_SECRET_PREVIOUS!,
> ]
> ```

**`cookie` 選項：**

| 選項 | 說明 |
|------|------|
| `secure: true` | 僅允許透過 HTTPS 傳送。正式環境必須啟用 |
| `sameSite: 'lax'` | 允許同站請求與頂層導覽攜帶 cookie，阻擋跨站 POST——有效緩解 CSRF |
| `httpOnly: true` | 禁止 `document.cookie` 存取，防止 XSS 竊取 session |
| `maxAge` | Cookie 存活時間（秒）。未設定時為 session cookie，瀏覽器關閉即失效 |

## 讀取 Session 資料

在任何 server function 中取得 session：

```tsx
const session = await useAppSession()

const userId = session.data.userId    // string | undefined
const email = session.data.email      // string | undefined
const role = session.data.role        // string | undefined
```

## 寫入 Session 資料

`session.update()` 以提供的物件**完整覆寫**現有 session 內容，加密後的 cookie 會自動附帶在 response 中：

```tsx
const session = await useAppSession()

await session.update({
  userId: user.id,
  email: user.email,
  role: user.role,
})
```

> **注意事項：Session fixation 防護。** 在使用者驗證成功後寫入 session 時，應確保先清除原有 session 再建立新的。這可防止攻擊者預先植入 session ID、等待受害者登入後劫持該 session。實務做法是在 login handler 中先呼叫 `session.clear()` 再執行 `session.update()`。

## 清除 Session

```tsx
const session = await useAppSession()
await session.clear()
```

這會移除所有 session 資料並使 cookie 失效。

## 完整範例：登入、驗證、登出

```tsx
// server/auth.ts
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'
import { useAppSession } from '../utils/session'

// 登入
export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    const user = await authenticateUser(data.email, data.password)
    if (!user) return { error: 'Invalid credentials' }

    const session = await useAppSession()
    // 先清除再寫入，防止 session fixation
    await session.clear()
    await session.update({ userId: user.id, email: user.email, role: user.role })

    throw redirect({ to: '/dashboard' })
  })

// 驗證身份
export const getCurrentUserFn = createServerFn({ method: 'GET' }).handler(
  async () => {
    const session = await useAppSession()
    if (!session.data.userId) return null
    return await getUserById(session.data.userId)
  }
)

// 登出
export const logoutFn = createServerFn({ method: 'POST' }).handler(
  async () => {
    const session = await useAppSession()
    await session.clear()
    throw redirect({ to: '/' })
  }
)
```

## 注意事項與 Edge Cases

### Cookie 大小限制

瀏覽器對每個 cookie 的大小限制約為 **4KB**。由於 `useSession` 將 session 資料加密後直接存入 cookie（而非僅存 session ID），payload 過大會導致 cookie 被靜默截斷或被瀏覽器拒絕。應遵循以下原則：

- Session 中僅存放識別資訊（`userId`、`role`）
- 避免將完整 user profile、permission list 或其他大型結構塞入 session
- 需要的資料應在 server function 中透過 `userId` 從資料庫查詢

### 並行 Session 更新（Concurrent Updates）

Cookie-based session 的本質是 last-write-wins。若同一使用者在多個 tab 或並行請求中同時觸發 `session.update()`，後寫入的 response cookie 會覆蓋先前的值。對於需要原子性更新的場景（例如 session 中的計數器），應將該狀態移至資料庫層級管理。

### 水平擴展與 Session 儲存後端

TanStack Start 預設的 cookie-based session 是 stateless 的——session 資料完整存放在 cookie 中，伺服器端不需要共享狀態。這在多伺服器部署下天然支援水平擴展，無需額外的 session store。

但當需求超出 cookie-based 方案的限制時（例如需要伺服器端 session 撤銷、超過 4KB 的 session 資料、或即時 session 失效），應考慮切換至 server-side session store：

- **Redis** 是最常見的選擇——低延遲、支援 TTL 自動過期、叢集模式下可水平擴展
- **資料庫（PostgreSQL / MySQL）** 適用於已有 DB infra 且 session 吞吐量不高的場景
- 此時 cookie 僅存放 session ID，實際資料存於後端

### Session 失效模式（Invalidation Patterns）

Cookie-based session 的限制之一是無法從伺服器端主動撤銷——cookie 由瀏覽器持有，伺服器只能在下次收到該 cookie 時拒絕它。常見的解決方案：

- **資料庫層級的 revocation list：** 在 DB 中維護已撤銷的 session 或 user 記錄，每次驗證 session 時進行比對
- **短 `maxAge` 搭配 refresh 機制：** 縮短 cookie 有效期（例如 15 分鐘），搭配前端自動 refresh，在 refresh 時檢查是否該 session 應被撤銷
- **Server-side session store：** 如上述，直接在 Redis / DB 中刪除 session record 即可立即失效

## 重點整理

- `useSession`（`@tanstack/react-start/server`）以加密 cookie 實現 stateless session
- `SESSION_SECRET` 存放於環境變數，正式環境應定期 rotate（password 支援陣列格式）
- Cookie 選項 `secure`、`sameSite: 'lax'`、`httpOnly` 三者缺一不可
- 登入時先 `clear()` 再 `update()` 以防止 session fixation
- Session 中僅存放最小必要資料——cookie 大小限制約 4KB
- 並行更新場景下 cookie-based session 採 last-write-wins 語意
- 需要伺服器端撤銷或大型 session 時，應改用 Redis / DB-backed session store

---

下一篇：[Authentication Context](./05-authentication-context.md)
