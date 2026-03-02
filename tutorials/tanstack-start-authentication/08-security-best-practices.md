# 08：安全性最佳實務

> 驗證系統的生產級安全措施：密碼雜湊策略、Session 強化、分散式速率限制、輸入驗證陷阱，以及 HTTP security headers 的縱深防禦。

## 1. 密碼雜湊

### 演算法選擇

```tsx
import bcrypt from 'bcryptjs'

const SALT_ROUNDS = 12

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS)
}

export async function verifyPassword(
  candidatePassword: string,
  storedHash: string,
): Promise<boolean> {
  return bcrypt.compare(candidatePassword, storedHash)
}
```

`bcrypt` 透過刻意的計算成本與自動加鹽，使離線暴力破解在經濟上不可行。`saltRounds = 12` 在目前硬體上約產生 250ms 的延遲——足以阻擋攻擊者，又不至於影響使用者體驗。

替代方案：`argon2id`（Password Hashing Competition 優勝者）提供可調整的記憶體成本參數，能有效抵抗 GPU/ASIC 加速攻擊。若團隊無歷史包袱，建議優先採用 argon2id。

### 密碼政策

依據 NIST SP 800-63B：

- 最少 8 字元，支援至少 64 字元
- 比對已知洩漏資料庫（Have I Been Pwned API 的 k-Anonymity 模型可在不洩漏完整密碼的前提下查詢）
- 不強制字元組合規則——研究顯示強制大小寫＋符號只會產出 `Password1!` 這類可預測模式

## 2. Session 安全性

### Cookie 設定

```tsx
export function useAppSession() {
  return useSession({
    name: '__Host-session', // __Host- prefix 強制 secure + path=/
    password: process.env.SESSION_SECRET!,
    cookie: {
      secure: true,
      sameSite: 'lax' as const,
      httpOnly: true,
      path: '/',
      maxAge: 60 * 60 * 24 * 7, // 7 天（秒）
    },
  })
}
```

| 設定 | 防護目標 | 備註 |
|------|----------|------|
| `secure: true` | 中間人攻擊 | Cookie 僅透過 HTTPS 傳送 |
| `sameSite: 'lax'` | CSRF | 阻止跨站 POST 請求自動附帶 Cookie |
| `httpOnly: true` | XSS 竊取 Cookie | `document.cookie` 無法存取 |
| `__Host-` prefix | Cookie 注入 | 瀏覽器強制 `secure=true`、`path=/`、禁止指定 `Domain` |

`sameSite: 'strict'` 會阻止所有跨站導航附帶 Cookie，包括從外部連結點進來的情境——對多數應用程式而言過於激進。`'lax'` 允許頂層 GET 導航附帶 Cookie，同時阻擋 POST/PUT/DELETE 跨站請求。

### Secret 管理與輪換

`SESSION_SECRET` 必須：

- 至少 32 bytes 的密碼學安全隨機值
- 儲存於環境變數或 secret manager（AWS Secrets Manager、HashiCorp Vault）
- 定期輪換

```bash
# 產生 secret
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

**Secret 輪換策略：** 多數 session 框架支援多 secret 模式——新的 secret 用於簽發，舊的 secret 仍可驗證。這允許 zero-downtime 輪換：

```tsx
// iron-session 等框架支援 password 陣列
// 第一個用於加密新 session，其餘用於解密既有 session
const SESSION_SECRETS = [
  process.env.SESSION_SECRET_CURRENT!,
  process.env.SESSION_SECRET_PREVIOUS!, // 輪換過渡期保留
]
```

輪換流程：部署新 secret 為 `CURRENT` → 原 `CURRENT` 降為 `PREVIOUS` → 等待 `maxAge` 週期後移除舊 secret。

## 3. 速率限制

### 生產級分散式速率限制（Redis Sliding Window）

記憶體內 `Map` 僅適用於單一 process。生產環境需要跨節點共享狀態：

```tsx
import { Redis } from 'ioredis'

const redis = new Redis(process.env.REDIS_URL!)

interface RateLimitResult {
  allowed: boolean
  remaining: number
  retryAfterMs: number
}

/**
 * Sliding window rate limiter using Redis sorted sets.
 * 相比 fixed window 不會出現邊界突增問題。
 */
export async function checkRateLimit(
  key: string,
  maxRequests: number,
  windowMs: number,
): Promise<RateLimitResult> {
  const now = Date.now()
  const windowStart = now - windowMs
  const redisKey = `rate_limit:${key}`

  const pipeline = redis.pipeline()
  pipeline.zremrangebyscore(redisKey, 0, windowStart) // 清除過期記錄
  pipeline.zadd(redisKey, now.toString(), `${now}:${crypto.randomUUID()}`)
  pipeline.zcard(redisKey) // 計算窗口內請求數
  pipeline.pexpire(redisKey, windowMs) // 設定 key 過期

  const results = await pipeline.exec()
  const currentCount = (results?.[2]?.[1] as number) ?? 0

  if (currentCount > maxRequests) {
    // 超出限制，移除剛才新增的記錄
    await redis.zremrangebyscore(redisKey, now, now)
    const oldestInWindow = await redis.zrange(redisKey, 0, 0, 'WITHSCORES')
    const retryAfterMs = oldestInWindow.length >= 2
      ? Number(oldestInWindow[1]) + windowMs - now
      : windowMs

    return { allowed: false, remaining: 0, retryAfterMs }
  }

  return {
    allowed: true,
    remaining: maxRequests - currentCount,
    retryAfterMs: 0,
  }
}
```

### 整合至登入流程

```tsx
export const loginFn = createServerFn({ method: 'POST' })
  .validator((data: unknown) => loginSchema.parse(data))
  .handler(async ({ data }) => {
    const clientIp = getClientIp()
    const emailKey = data.email.toLowerCase()

    // 雙維度速率限制：IP + email
    const [ipLimit, emailLimit] = await Promise.all([
      checkRateLimit(`ip:${clientIp}`, 20, 15 * 60 * 1000),
      checkRateLimit(`email:${emailKey}`, 5, 15 * 60 * 1000),
    ])

    if (!ipLimit.allowed || !emailLimit.allowed) {
      const retryAfter = Math.max(ipLimit.retryAfterMs, emailLimit.retryAfterMs)
      return {
        error: 'Too many login attempts. Please try again later.',
        retryAfterMs: retryAfter,
      }
    }

    // ... 驗證邏輯
  })
```

雙維度限制的原因：僅限制 IP 會被分散式攻擊繞過；僅限制 email 會讓攻擊者對目標帳號發動鎖定攻擊。兩者結合可相互補強。

### IP 取得的陷阱：X-Forwarded-For 偽造

```tsx
/**
 * 僅信任已知的 reverse proxy 設定的 header。
 * 直接讀取 X-Forwarded-For 的第一個值是常見的安全漏洞——
 * 攻擊者可自行注入假 header 繞過速率限制。
 */
export function getClientIp(request: Request): string {
  // 若你的基礎架構使用 Cloudflare：
  const cfIp = request.headers.get('cf-connecting-ip')
  if (cfIp) return cfIp

  // 若部署於 AWS ALB / nginx 等已知 proxy 後方，
  // 取 X-Forwarded-For 的「最右側非信任 proxy」值
  const xff = request.headers.get('x-forwarded-for')
  if (xff) {
    const ips = xff.split(',').map((ip) => ip.trim())
    // 從右往左，跳過已知的 proxy IP，取第一個外部 IP
    // 需根據實際部署拓撲設定 trustedProxies
    const trustedProxies = new Set(
      (process.env.TRUSTED_PROXIES ?? '').split(','),
    )
    for (let i = ips.length - 1; i >= 0; i--) {
      if (!trustedProxies.has(ips[i])) return ips[i]
    }
  }

  return '0.0.0.0' // fallback
}
```

## 4. 輸入驗證

### Zod Schema 設計

```tsx
import { z } from 'zod'

const loginSchema = z.object({
  email: z
    .string()
    .email('Invalid email format')
    .max(255)
    .transform((e) => e.toLowerCase().trim()),
  password: z
    .string()
    .min(8)
    .max(128), // 上限防止 bcrypt 72-byte 截斷造成的混淆
})

const registerSchema = z.object({
  email: z
    .string()
    .email()
    .max(255)
    .transform((e) => e.toLowerCase().trim()),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .max(128),
  name: z
    .string()
    .min(1, 'Name is required')
    .max(100)
    .trim(),
})
```

### Zod Coercion 陷阱

`z.coerce.string()` 會對任意輸入呼叫 `String()` 轉型——包括物件。這意味著攻擊者送出 `{ email: { toString: "..." } }` 之類的 payload 時可能產生非預期行為：

```tsx
// 危險：z.coerce.string() 會將 {} 轉成 "[object Object]"
const unsafeSchema = z.object({
  email: z.coerce.string().email(),
})

// 安全：先用 z.string() 確保型別，再用 transform 處理
const safeSchema = z.object({
  email: z.string().email().transform((v) => v.toLowerCase()),
})
```

對密碼欄位亦然——`.max(128)` 之所以重要，是因為 bcrypt 在多數實作中會靜默截斷超過 72 bytes 的輸入。使用者設定了 100 字元密碼，但實際只有前 72 bytes 參與雜湊，這會造成安全誤期。若需要支援更長密碼，可先做一次 SHA-256 pre-hash。

## 5. Timing Attack 防護

`bcrypt.compare` 已是 constant-time，但其他 string 比對場景（API key、reset token、HMAC）需額外注意：

```tsx
import { timingSafeEqual, createHmac } from 'node:crypto'

/**
 * Timing-safe string comparison.
 * 注意：長度不同時仍會提前返回 false，這洩漏了長度資訊。
 * 若連長度也需保密，應先 HMAC 再比較固定長度的 digest。
 */
export function safeCompare(a: string, b: string): boolean {
  if (a.length !== b.length) return false
  return timingSafeEqual(Buffer.from(a, 'utf-8'), Buffer.from(b, 'utf-8'))
}

/**
 * 當兩個值可能長度不同時（例如使用者輸入 vs 儲存值），
 * 先 HMAC 確保比較的長度一致。
 */
export function safeCompareHmac(
  untrusted: string,
  trusted: string,
  secret: string,
): boolean {
  const hmac = (val: string) =>
    createHmac('sha256', secret).update(val).digest()
  return timingSafeEqual(hmac(untrusted), hmac(trusted))
}
```

## 6. 資訊洩漏防護

```tsx
// 錯誤：揭露帳號是否存在
return { error: 'Wrong password for user@example.com' }

// 正確：一致性的模糊回應
return { error: 'Invalid email or password' }
```

帳號列舉不只出現在登入——註冊（「此 email 已被使用」）與密碼重設也是常見洩漏點。對這些端點統一回應 generic message，並在背景執行實際邏輯。

## 7. Security Headers

這些 headers 構成縱深防禦的外層，對應 OWASP Top 10 中的多項風險（A03: Injection、A05: Security Misconfiguration）。

### 設定範例

```tsx
// middleware 或 server plugin 中統一設定
function setSecurityHeaders(response: Response): void {
  // Content-Security-Policy：XSS 的最強縱深防禦
  response.headers.set(
    'Content-Security-Policy',
    [
      "default-src 'self'",
      "script-src 'self'", // 禁止 inline script；若需要，使用 nonce
      "style-src 'self' 'unsafe-inline'", // CSS-in-JS 框架常需要此設定
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self'",
      "frame-ancestors 'none'", // 等同 X-Frame-Options: DENY
      "base-uri 'self'",
      "form-action 'self'",
      "upgrade-insecure-requests",
    ].join('; '),
  )

  // HSTS：強制瀏覽器後續只用 HTTPS 連線
  response.headers.set(
    'Strict-Transport-Security',
    'max-age=63072000; includeSubDomains; preload', // 2 年
  )

  // 防止瀏覽器 MIME type sniffing
  response.headers.set('X-Content-Type-Options', 'nosniff')

  // Referrer 資訊控制
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')

  // 限制瀏覽器 API 存取
  response.headers.set(
    'Permissions-Policy',
    'camera=(), microphone=(), geolocation=()',
  )
}
```

### Security Headers 檢查清單

| Header | 用途 | OWASP 對應 |
|--------|------|-----------|
| `Content-Security-Policy` | 限制可執行的腳本來源，XSS 縱深防禦 | A03: Injection |
| `Strict-Transport-Security` | 強制 HTTPS，防止 SSL stripping | A02: Cryptographic Failures |
| `X-Content-Type-Options: nosniff` | 阻止 MIME type sniffing 觸發腳本執行 | A05: Security Misconfiguration |
| `Referrer-Policy` | 控制跨站導航時洩漏的 URL 資訊 | A01: Broken Access Control |
| `Permissions-Policy` | 停用不需要的瀏覽器 API | A05: Security Misconfiguration |
| `X-Frame-Options` / CSP `frame-ancestors` | 防止 clickjacking | A01: Broken Access Control |

### Subresource Integrity（SRI）

載入第三方 CDN 資源時，SRI 確保檔案未被竄改：

```html
<script
  src="https://cdn.example.com/library@3.2.1/dist/lib.min.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxAh7..."
  crossorigin="anonymous"
></script>
```

若 CDN 被入侵或遭受供應鏈攻擊，hash 不符時瀏覽器會拒絕執行該資源。搭配 CSP 的 `require-sri-for` directive 可強制所有外部腳本都需要 SRI。

在 build 流程中自動產生 integrity hash：

```bash
# 產生 SRI hash
openssl dgst -sha384 -binary ./dist/bundle.js | openssl base64 -A
```

## 8. HTTPS 強制

生產環境必須全站 HTTPS。HSTS header（見上方）確保瀏覽器在首次成功 HTTPS 連線後，後續所有請求自動升級為 HTTPS——即使使用者手動輸入 `http://`。

提交至 [HSTS Preload List](https://hstspreload.org/) 可在瀏覽器層級硬編碼你的域名為 HTTPS-only，消除首次連線的降級風險。

## 重點整理

- **密碼雜湊**：bcrypt（`saltRounds >= 12`）或 argon2id；注意 bcrypt 的 72-byte 截斷限制
- **Session**：`httpOnly`、`secure`、`sameSite: 'lax'`、`__Host-` prefix；secret 輪換採多 key 模式實現 zero-downtime
- **速率限制**：Redis sorted set sliding window；雙維度（IP + email）限制；正確處理 `X-Forwarded-For` 信任鏈
- **輸入驗證**：Zod `z.string()` 而非 `z.coerce.string()`；密碼 `.max(128)` 對應 bcrypt 截斷
- **Timing attack**：非密碼場景使用 `timingSafeEqual`；長度不等時先 HMAC 再比較
- **Security headers**：CSP 阻止 XSS、HSTS 強制 HTTPS、SRI 防禦供應鏈攻擊
- **資訊洩漏**：登入、註冊、密碼重設統一使用模糊回應

---

下一篇：[測試驗證機制](./09-testing-authentication.md)
