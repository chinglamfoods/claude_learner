# 01：TanStack Start 認證功能介紹

> TanStack Start 的認證架構概覽——支援的認證策略、整合選項，以及本教學的涵蓋範圍。

## TanStack Start 認證架構

TanStack Start 建構於 TanStack Router 之上，透過 server functions 將認證邏輯隔離在伺服器端執行，避免敏感資料洩漏至 client bundle。其路由系統原生支援 `beforeLoad` 攔截，可在路由層級實現存取控制，搭配基於 cookie 的 session 機制維持認證狀態。

核心認證元件包含：

- **Server functions** — 認證邏輯（驗證、session 操作）僅在伺服器端執行，不進入 client bundle
- **Cookie-based sessions** — 透過 `httpOnly`、`secure`、`sameSite` 屬性保護的 session cookie
- **Route-level guards** — 利用 `beforeLoad` 在路由解析前攔截未認證請求
- **Auth context** — 透過 React context 將認證狀態注入整個元件樹

## 認證策略選擇

| 策略 | 適用情境 | 取捨 |
|------|---------|------|
| **自行建構（DIY）** | 需要完全掌控認證流程、有特殊合規需求 | 需自行處理安全細節（密碼雜湊、CSRF、rate limiting） |
| **合作夥伴方案（Clerk、WorkOS）** | 需要企業級功能（SSO、SCIM、MFA）且預算允許 | 廠商鎖定風險、按 MAU 計費、資料駐留限制 |
| **開源方案（Better Auth、Auth.js）** | 需要自主部署、保有資料控制權 | 社群維護品質不一，重大版本升級可能有 breaking changes |
| **託管服務（Supabase Auth、Auth0、Firebase Auth）** | 希望將認證基礎設施完全外包 | 客製化受限、遷移成本高、可能有冷啟動延遲 |

**實務考量：** 自行建構適合對認證流程有深度客製需求的團隊，但需投入持續的安全維護成本。多數產品團隊建議從 Better Auth 或 Clerk 起步，待需求明確後再評估是否遷移。

## 可實作的功能

- Email / password 登入與註冊（含密碼強度驗證、帳號鎖定機制）
- OAuth 社群登入（Google、GitHub 等，需處理 account linking 情境）
- Role-based access control（RBAC）— 在路由與 API 層級限制操作權限
- 受保護路由 — `beforeLoad` 攔截搭配 redirect 至登入頁
- Session 管理 — cookie rotation、過期策略、多裝置登入控制
- 密碼重設 — 基於一次性 token 的重設流程，需設定 token 過期時間

## 本教學涵蓋的內容

1. **介紹**（本篇）— 架構概覽與策略選擇
2. **核心概念** — Authentication vs. authorization 模型與資料流
3. **Server functions** — 伺服器端認證邏輯的實作與安全性
4. **Session 管理** — Cookie-based session 的設定與 rotation 策略
5. **Auth context** — 認證狀態的傳遞與 hydration
6. **路由保護** — `beforeLoad` guard 與 redirect 模式
7. **實作模式** — Email/password、OAuth、RBAC、密碼重設的完整實作
8. **安全性最佳實務** — 密碼雜湊、rate limiting、CSRF 防護、input validation
9. **測試** — 認證流程的 unit test 與 integration test 策略
10. **Production 部署** — Loading states、persistent sessions、遷移策略

## 來源參考

本教學基於 TanStack Start 官方文件：
- [Authentication Overview](https://tanstack.com/start/latest/docs/framework/react/guide/authentication-overview)
- [Authentication Guide](https://tanstack.com/start/latest/docs/framework/react/guide/authentication)

## 重點整理

- TanStack Start 透過 server functions 將認證邏輯限制在伺服器端，搭配 cookie-based session 與 route guards 組成完整認證架構
- 認證策略的選擇取決於團隊規模、合規需求與長期維護成本——沒有通用最佳解
- 自行建構提供最大彈性，但需承擔安全維護責任；第三方方案降低初期成本，但需評估鎖定風險
- 本教學涵蓋從基礎架構到 production 部署的完整認證實作流程

---

下一篇：[核心概念](./02-core-concepts.md)
