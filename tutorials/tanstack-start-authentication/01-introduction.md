# 01：TanStack Start 認證功能介紹

> 一份適合初學者的概覽，介紹 TanStack Start 在認證方面提供了哪些功能、你可以用它打造什麼，以及開始前需要準備的事項。

## 什麼是 TanStack Start？

TanStack Start 是一個建立在 TanStack Router 之上的全端 React 框架。它提供伺服器端渲染（SSR）、伺服器函式（server functions），以及強大的路由系統——全部具備端對端的型別安全。

在**認證**（authentication，驗證使用者的身分）和**授權**（authorization，控制使用者可以執行的操作）方面，TanStack Start 提供了實作安全、可上線的認證系統所需的基礎元件。

## 你可以打造什麼？

使用 TanStack Start 的認證工具，你可以：

- **電子郵件／密碼登入系統** — 傳統的註冊與登入流程
- **社群登入** — 透過 OAuth 實現「使用 Google／GitHub 登入」
- **角色型存取控制（RBAC）** — 根據使用者角色（管理員、版主、一般使用者）限制頁面與操作
- **受保護的路由** — 需要登入才能存取的頁面
- **Session 管理** — 安全的、基於 cookie 的 session，可在多次請求之間保持狀態
- **密碼重設流程** — 安全的、基於 token 的密碼復原機制

你也可以整合第三方認證供應商，例如 Clerk、WorkOS、Auth.js、Better Auth 或 Supabase Auth，而不必從零開始自己建構。

## 認證方式

TanStack Start 支援多種方式：

| 方式 | 最適合場景 | 取捨 |
|------|-----------|------|
| **自行建構（DIY）** | 完全掌控、客製化需求 | 工作量較大，需自行處理安全性 |
| **合作夥伴方案（Clerk、WorkOS）** | 企業級功能、快速建置 | 廠商綁定、按使用者計費 |
| **開源函式庫（Better Auth、Auth.js）** | 社群驅動、可自行部署 | 支援不如託管方案完善 |
| **託管服務（Supabase、Auth0、Firebase）** | 代管基礎設施 | 客製化程度較低 |

## 先備條件

在開始本教學之前，你應該具備：

- **基本的 React 知識** — 元件、hooks、JSX
- **已安裝 Node.js**（建議 v18 或更新版本）
- **一個 TanStack Start 專案** — 如果還沒有，請參考 [Quick Start 指南](https://tanstack.com/start/latest/docs/framework/react/quick-start)
- **基本了解 HTTP** — 請求、回應、cookie（有幫助但非必要）

## 本教學涵蓋的內容

本教學將逐步帶你瀏覽 TanStack Start 認證文件：

1. **介紹**（本篇）— 概覽與先備條件
2. **核心概念** — 認證 vs. 授權，以及架構模型
3. **伺服器函式** — 在伺服器端安全地處理認證邏輯
4. **Session 管理** — 設定安全的、基於 cookie 的 session
5. **認證 Context** — 在整個 React 應用程式中共享認證狀態
6. **路由保護** — 阻止未經授權的存取
7. **實作模式** — 電子郵件／密碼、RBAC、OAuth、密碼重設
8. **安全性最佳實務** — 密碼雜湊、速率限制、輸入驗證
9. **測試認證功能** — 單元測試與整合測試
10. **常見模式與正式上線** — 載入狀態、記住我、遷移建議

## 來源參考

本教學基於 TanStack Start 官方文件：
- [Authentication Overview](https://tanstack.com/start/latest/docs/framework/react/guide/authentication-overview)
- [Authentication Guide](https://tanstack.com/start/latest/docs/framework/react/guide/authentication)

## 重點整理

- TanStack Start 是一個全端 React 框架，內建認證所需的工具
- 你可以自行建構認證系統，也可以整合第三方供應商
- 此框架以伺服器函式、session 及路由保護作為核心基礎元件
- 本教學將逐步帶你了解每一個概念

---

下一篇：[核心概念](./02-core-concepts.md)
