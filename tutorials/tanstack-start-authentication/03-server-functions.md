# 03：用於驗證的 Server Functions

> 透過 `createServerFn` 將驗證邏輯隔離在伺服器端，涵蓋輸入驗證、session 建立、重新導向控制，以及常見的序列化與錯誤處理陷阱。

## `createServerFn` API 結構

`createServerFn` 接受一個 HTTP method 設定，回傳一個 builder，可鏈式呼叫 `.inputValidator()` 與 `.handler()`。框架在 build 時自動將 handler 抽離至伺服器端 bundle，用戶端僅保留一個 RPC 呼叫代理。

```tsx
import { createServerFn } from '@tanstack/react-start'

const myServerFunction = createServerFn({ method: 'POST' })
  .inputValidator((data: unknown) => {
    // 務必在此處做 runtime 驗證，不要只依賴 TypeScript 型別
    const parsed = mySchema.parse(data)
    return parsed
  })
  .handler(async ({ data }) => {
    // 僅在伺服器端執行
    return { result: 'something' }
  })
```

幾個設計上的要點：

- **`method`** — `'POST'` 用於寫入操作（登入、登出、修改資料），`'GET'` 用於純讀取。GET server functions 的回傳值可被 TanStack Router 的 loader 快取；POST 則不會。
- **`.inputValidator()`** — 這是 runtime boundary，接收的參數型別是 `unknown`。只做 TypeScript 型別標註而不做 runtime 驗證等同於沒有驗證——用戶端可以傳送任意 payload。建議搭配 Zod 或 Valibot。
- **`.handler()`** — 實際的伺服器端邏輯。回傳值必須是 JSON-serializable（詳見下方序列化限制）。

## 登入 Server Function

```tsx
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'
import { z } from 'zod'

const loginInputSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
})

export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: unknown) => loginInputSchema.parse(data))
  .handler(async ({ data }) => {
    const user = await authenticateUser(data.email, data.password)

    if (!user) {
      // 回傳結構化錯誤，讓用戶端能精確顯示
      return {
        error: true as const,
        code: 'INVALID_CREDENTIALS',
        message: '電子郵件或密碼不正確',
      }
    }

    const session = await useAppSession()
    await session.update({
      userId: user.id,
      email: user.email,
    })

    // 必須用 throw，不能用 return（見下方說明）
    throw redirect({ to: '/dashboard' })
  })
```

### `throw redirect` vs `return redirect`

`redirect()` 回傳的是一個特殊物件。TanStack Start 透過例外機制（throw）來中斷 handler 執行流程並觸發 HTTP 302 回應。如果你用 `return redirect(...)` 而非 `throw redirect(...)`，框架不會執行重新導向——回傳值會被當成普通的 JSON 資料送回用戶端，TypeScript 也不會警告你。

**規則：凡是在 server function handler 內要觸發導向，一律使用 `throw`。**

## 登出 Server Function

```tsx
export const logoutFn = createServerFn({ method: 'POST' }).handler(
  async () => {
    const session = await useAppSession()
    await session.clear()
    throw redirect({ to: '/' })
  }
)
```

登出不需要 `.inputValidator()`——沒有用戶端輸入。但在正式環境中，考慮加入 CSRF token 驗證：

```tsx
export const logoutFn = createServerFn({ method: 'POST' })
  .inputValidator((data: unknown) => {
    const parsed = z.object({ csrfToken: z.string() }).parse(data)
    return parsed
  })
  .handler(async ({ data }) => {
    await verifyCsrfToken(data.csrfToken)
    const session = await useAppSession()
    await session.clear()
    throw redirect({ to: '/' })
  })
```

## 取得目前使用者

```tsx
export const getCurrentUserFn = createServerFn({ method: 'GET' }).handler(
  async () => {
    const session = await useAppSession()
    const userId = session.data.userId

    if (!userId) {
      return null
    }

    const user = await getUserById(userId)

    // 防禦性處理：session 中存在 userId 但資料庫中使用者已被刪除
    if (!user) {
      const session = await useAppSession()
      await session.clear()
      return null
    }

    // 僅回傳用戶端需要的欄位，避免洩漏敏感資料
    return {
      id: user.id,
      email: user.email,
      role: user.role,
    }
  }
)
```

使用 `method: 'GET'` 是因為此函式不產生副作用。當搭配 TanStack Router 的 `loader` 時，GET server functions 的結果會被 deduplication 與快取機制處理——同一次導覽中多個元件呼叫 `getCurrentUserFn()` 只會產生一次實際請求。

## 常見陷阱與邊界情況

### 序列化限制

Server function 的輸入與輸出都透過 HTTP 傳輸，必須是 JSON-serializable。以下型別**無法**正確傳遞：

- `Date` 物件（會變成字串，接收端不會自動轉回 `Date`）
- `Map`、`Set`
- `undefined` 值（JSON 中不存在，會被靜默移除）
- 函式、class instances

若需要傳遞日期，使用 ISO 8601 字串，並在接收端明確轉換：

```tsx
// 輸出端
return { createdAt: user.createdAt.toISOString() }

// 接收端
const createdAt = new Date(result.createdAt)
```

### 網路失敗處理

用戶端呼叫 server function 本質上是一次 HTTP 請求。網路中斷、伺服器 5xx 回應等情況下，呼叫會拋出例外。用戶端應做防禦性處理：

```tsx
async function handleLogin(email: string, password: string) {
  try {
    const result = await loginFn({ data: { email, password } })
    if (result?.error) {
      // 業務邏輯錯誤（憑證錯誤等）
      setError(result.message)
      return
    }
    // 成功時 loginFn 會 throw redirect，不會走到這裡
  } catch (err) {
    // redirect 也是透過 throw 傳遞，TanStack Start 會自動處理
    // 這裡只會捕獲到真正的網路或序列化錯誤
    if (isRedirect(err)) throw err
    setError('網路錯誤，請稍後再試')
  }
}
```

### Race Conditions

使用者快速連續點擊登入按鈕會觸發多次 `loginFn` 呼叫。處理方式：

1. **用戶端：** 點擊後立即 disable 按鈕，或使用 `useMutation` 的 `isPending` 狀態
2. **伺服器端：** `session.update()` 是 idempotent 的——多次寫入相同資料不會造成問題，但如果 `authenticateUser` 有 rate limiting 邏輯，需確保不會將合法使用者鎖定

### Input Validation 的位置

`.inputValidator()` 在 handler **之前**執行。如果驗證失敗，handler 不會被呼叫。但注意：`.inputValidator()` 拋出的錯誤會直接以 500 狀態碼回傳用戶端。若需要回傳結構化的驗證錯誤訊息（如欄位級別的錯誤），有兩種策略：

```tsx
// 策略 A：在 inputValidator 中做基本型別檢查，在 handler 中做業務驗證
.inputValidator((data: unknown) => z.object({ email: z.string(), password: z.string() }).parse(data))
.handler(async ({ data }) => {
  if (!isValidEmail(data.email)) {
    return { error: true, fields: { email: '無效的電子郵件格式' } }
  }
  // ...
})

// 策略 B：在 inputValidator 中完整驗證，用 try-catch 包裝回傳友善錯誤
.inputValidator((data: unknown) => {
  const result = loginInputSchema.safeParse(data)
  if (!result.success) {
    throw new Error(result.error.issues.map(i => i.message).join(', '))
  }
  return result.data
})
```

### Middleware 模式

當多個 server functions 共享相同的前置邏輯（例如驗證使用者已登入），可抽出 middleware 函式：

```tsx
async function requireAuth() {
  const session = await useAppSession()
  if (!session.data.userId) {
    throw redirect({ to: '/login' })
  }
  return session.data
}

export const updateProfileFn = createServerFn({ method: 'POST' })
  .inputValidator((data: unknown) => updateProfileSchema.parse(data))
  .handler(async ({ data }) => {
    const sessionData = await requireAuth()
    return await updateUser(sessionData.userId, data)
  })
```

這比在每個 handler 中重複 session 檢查更可維護。`requireAuth` 在驗證失敗時直接 `throw redirect`，後續程式碼不會執行。

## 重點整理

- `createServerFn` 將函式隔離在伺服器端 bundle，用戶端僅持有 RPC 代理
- `.inputValidator()` 是 runtime boundary——務必做 runtime 驗證，不要只依賴 TypeScript 型別標註
- `throw redirect(...)` 才能觸發重新導向，`return redirect(...)` 不會生效
- 回傳值必須是 JSON-serializable；`Date`、`Map`、`Set`、`undefined` 均不適用
- 用戶端須處理網路失敗，並注意 `throw redirect` 與真正的錯誤例外會在同一個 catch 區塊中出現
- 使用 middleware 函式（如 `requireAuth()`）抽出共用的驗證邏輯，避免在每個 handler 中重複

---

下一篇：[Session 管理](./04-session-management.md)
