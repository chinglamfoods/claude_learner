# 08：安全性最佳實務

> 驗證系統的必備安全措施：密碼雜湊、Session 安全性、速率限制與輸入驗證。

## 為什麼安全性很重要

驗證機制是應用程式的大門。薄弱的實作可能導致：
- 使用者帳號被竊取
- 資料外洩
- 法律責任（違反 GDPR、CCPA 法規）

好消息是：遵循幾個關鍵的實務做法，就能大幅提升驗證系統的安全性。

## 1. 密碼安全

### 使用強雜湊演算法

絕對不要以明文儲存密碼。請使用專為密碼設計的雜湊演算法：

```tsx
import bcrypt from 'bcryptjs'

// 儲存前先對密碼進行雜湊處理
const saltRounds = 12 // 數值越高越安全，但速度越慢
const hashedPassword = await bcrypt.hash(password, saltRounds)

// 登入時驗證密碼
const isValid = await bcrypt.compare(providedPassword, storedHash)
```

**為什麼選擇 `bcrypt`？**
- 刻意設計為低速——使暴力破解攻擊變得不切實際
- 自動加鹽（salting）——即使相同的密碼，每次產生的雜湊值都不同
- 可調整的工作因子——隨著硬體效能提升，可增加 `saltRounds` 的數值

**替代方案：** `scrypt` 和 `argon2` 也是優秀的選擇。Argon2 是密碼雜湊競賽（Password Hashing Competition）的優勝者，被認為是最現代的方案。

### 密碼要求

最低限度應要求：
- 8 個以上的字元（NIST 建議支援最多 64 個以上的字元）
- 比對已知的外洩密碼清單（非必要但建議實作）

避免過於嚴格的規則（必須包含大寫、符號、數字）——研究顯示這類規則反而會導致可預測的模式，例如 `Password1!`。

## 2. Session 安全性

### 正確設定 Cookie

```tsx
export function useAppSession() {
  return useSession({
    name: 'app-session',
    password: process.env.SESSION_SECRET!, // 至少 32 個字元
    cookie: {
      secure: process.env.NODE_ENV === 'production', // 正式環境僅限 HTTPS
      sameSite: 'lax',    // CSRF 防護
      httpOnly: true,      // XSS 防護
      maxAge: 7 * 24 * 60 * 60, // 7 天
    },
  })
}
```

以下是每項設定所防護的攻擊類型：

| 設定 | 攻擊類型 | 防護方式 |
|------|----------|----------|
| `secure: true` | 中間人攻擊 | Cookie 僅透過加密的 HTTPS 連線傳送 |
| `sameSite: 'lax'` | CSRF（跨站請求偽造） | 瀏覽器不會從第三方網站發送 Cookie |
| `httpOnly: true` | XSS（跨站腳本攻擊） | JavaScript 無法透過 `document.cookie` 讀取 Cookie |
| `maxAge` | Session 劫持 | 限制被竊取的 Cookie 有效時間 |

### Session 密鑰

你的 `SESSION_SECRET` 必須：
- **至少 32 個字元**長度
- **隨機產生**（使用密碼產生器）
- **儲存在環境變數中**（絕不提交到 git）
- **在正式環境中定期輪換**

產生方式：
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

## 3. 速率限制

速率限制透過限制登入嘗試次數來防止暴力破解攻擊。

### 簡易記憶體內速率限制器

```tsx
// 簡易速率限制器（正式環境中多伺服器部署請使用 Redis）
const loginAttempts = new Map<string, { count: number; resetTime: number }>()

export const rateLimitLogin = (ip: string): boolean => {
  const now = Date.now()
  const attempts = loginAttempts.get(ip)

  // 首次嘗試或時間窗口已過期——允許並開啟新窗口
  if (!attempts || now > attempts.resetTime) {
    loginAttempts.set(ip, {
      count: 1,
      resetTime: now + 15 * 60 * 1000,  // 15 分鐘的時間窗口
    })
    return true  // 允許
  }

  // 在此時間窗口內嘗試次數過多
  if (attempts.count >= 5) {
    return false  // 封鎖
  }

  // 增加計數並允許
  attempts.count++
  return true
}
```

### 使用速率限制器

```tsx
export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    // 處理登入前先檢查速率限制
    const clientIp = getClientIp()  // 根據你的伺服器架構實作
    if (!rateLimitLogin(clientIp)) {
      return { error: 'Too many login attempts. Please try again later.' }
    }

    // ... 繼續進行驗證流程
  })
```

**重要提醒：** 記憶體內的 `Map` 方式適用於單一伺服器部署。對於多伺服器的正式環境，請使用 Redis 或類似的共享儲存方案。

## 4. 輸入驗證

務必在伺服器端驗證輸入。用戶端驗證是為了改善使用者體驗——它可以被繞過。

### 使用 Zod 進行驗證

```tsx
import { z } from 'zod'

// 為登入資料定義嚴格的結構描述
const loginSchema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(8).max(100),
})

export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data) => loginSchema.parse(data))
  .handler(async ({ data }) => {
    // data 現在保證是有效的：
    // - email 是有效的電子郵件格式，最多 255 個字元
    // - password 為 8-100 個字元
  })
```

### 註冊驗證

```tsx
const registerSchema = z.object({
  email: z.string().email().max(255),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .max(100),
  name: z
    .string()
    .min(1, 'Name is required')
    .max(100)
    .trim(),
})
```

### 為什麼要在伺服器端驗證？

- 用戶端驗證可以透過直接發送請求來繞過（curl、Postman、瀏覽器開發者工具）
- `.max()` 限制可防止透過超長字串進行的阻斷服務攻擊
- `.email()` 確保在查詢資料庫前格式正確
- Zod 提供型別安全的解析輸出——你的 handler 會取得經過驗證且具有型別的資料

## 額外安全措施

### 時序安全比對

檢查密碼時，`bcrypt.compare` 本身已是時序安全的。但對於其他比對（例如 token），請使用：

```tsx
import { timingSafeEqual } from 'crypto'

function safeCompare(a: string, b: string): boolean {
  if (a.length !== b.length) return false
  return timingSafeEqual(Buffer.from(a), Buffer.from(b))
}
```

這可以防止時序攻擊——攻擊者透過測量回應時間來猜測字元。

### 不要洩漏資訊

```tsx
// 不好：透露了該電子郵件存在於你的系統中
return { error: 'Wrong password for user@example.com' }

// 好：使用不會洩漏任何資訊的通用訊息
return { error: 'Invalid credentials' }
```

### 正式環境使用 HTTPS

正式環境務必使用 HTTPS。如果沒有的話：
- Cookie 在傳輸過程中可能被攔截
- 登入憑證會以明文傳送
- 中間人攻擊變得輕而易舉

## 重點整理

- **密碼**：使用 bcrypt 進行雜湊（12 以上的 salt rounds），絕不儲存明文
- **Session**：使用 `httpOnly`、`secure` 和 `sameSite` Cookie 設定
- **速率限制**：限制每個 IP 的登入嘗試次數以防止暴力破解
- **驗證**：務必在伺服器端使用 Zod 等函式庫進行驗證
- **不要洩漏資訊**：使用通用的錯誤訊息，採用時序安全的比對方式
- **HTTPS**：正式環境必備——沒有商量的餘地

---

下一篇：[測試驗證機制](./09-testing-authentication.md)
