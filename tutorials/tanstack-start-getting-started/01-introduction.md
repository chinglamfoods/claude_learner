# 01：TanStack Start 入門概覽

> TanStack Start 的定位、架構特性、上手方式一覽，以及本教學的涵蓋範圍。

## TanStack Start 是什麼

TanStack Start 是建構於 [TanStack Router](https://tanstack.com/router) 與 [Vite](https://vite.dev/) 之上的全端 React 框架。它提供 file-based routing、server functions、SSR/SSG、以及跨伺服器與客戶端的型別安全整合。與 Next.js 或 Remix 的定位相近，但路由層完全由 TanStack Router 驅動，搭配 Vite 作為建構工具。

核心特性：

- **File-based routing** — 以 `src/routes/` 目錄結構自動產生路由樹，支援巢狀路由、layout routes、路徑參數
- **Server functions** — 透過 `createServerFn` 將伺服器邏輯封裝為可直接呼叫的函式，框架自動處理 RPC 序列化
- **Full-stack type safety** — 從路由參數、loader 資料到 server function 的輸入輸出，全程型別推導
- **Vite-powered** — 開發時享有 HMR、生產建構由 Vite 的 Rollup pipeline 處理

## 上手方式

根據需求選擇適合的起步方式：

| 方式 | 適用情境 | 所需時間 |
|------|---------|---------|
| **CLI 快速建立** (`npm create @tanstack/start@latest`) | 直接開始新專案，可選配 Tailwind、ESLint 等 | < 2 分鐘 |
| **TanStack Builder** (tanstack.com/builder) | 透過視覺化介面設定專案選項 | < 2 分鐘 |
| **Clone 範例專案** | 需要特定功能組合的起點（Auth、React Query、Clerk 等） | < 5 分鐘 |
| **從零建構** | 想完整理解每個檔案的用途與設定 | 10–15 分鐘 |
| **從其他框架遷移** | 已有 Next.js 或 Remix 專案要轉換 | 視專案規模而定 |

## 技術先決條件

- Node.js 18+
- 熟悉 React（hooks、元件模式）
- 基本 TypeScript 能力（泛型、型別推導）
- 了解 Vite 的基礎運作方式（plugin 系統、dev server）

## 本教學涵蓋的內容

1. **介紹**（本篇）— 架構概覽與上手方式選擇
2. **快速開始** — CLI 建立專案與範例專案一覽
3. **專案初始化與相依套件** — 從空目錄開始設定 TypeScript 與安裝套件
4. **Vite 與 Start Plugin 設定** — `vite.config.ts` 與 plugin 順序
5. **Router 設定與根元件** — `router.tsx`、`__root.tsx` 的結構與用途
6. **第一個路由與 Server Function** — 建立首頁路由、讀寫伺服器端狀態

## 參考來源

- [Getting Started](https://tanstack.com/start/latest/docs/framework/react/getting-started)
- [Quick Start](https://tanstack.com/start/latest/docs/framework/react/quick-start)
- [Build from Scratch](https://tanstack.com/start/latest/docs/framework/react/build-from-scratch)

## 重點整理

- TanStack Start 是基於 TanStack Router + Vite 的全端 React 框架
- 提供 file-based routing、server functions、SSR、full-stack type safety
- 上手方式涵蓋 CLI 建立、範例 clone、從零建構、以及框架遷移
- 本教學以「從零建構」為主軸，同時涵蓋 CLI 快速建立的流程

---

下一篇：[快速開始](./02-quick-start.md)
