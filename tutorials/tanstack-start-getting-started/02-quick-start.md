# 02：快速開始

> 使用 CLI 在兩分鐘內建立 TanStack Start 專案，或從範例專案開始。

## CLI 建立專案

最快的方式是透過 `create` 指令：

```bash
# 使用 pnpm
pnpm create @tanstack/start@latest

# 使用 npm
npm create @tanstack/start@latest
```

CLI 會引導你選擇專案名稱，以及是否加入 Tailwind CSS、ESLint 等工具。完成後即可啟動 dev server：

```bash
cd my-project
npm install
npm run dev
```

## 從範例專案開始

如果你需要特定功能組合作為起點，可以直接 clone 官方範例。使用 `gitpick` 擷取特定範例目錄：

```bash
# clone start-basic 範例
npx gitpick TanStack/router/tree/main/examples/react/start-basic start-basic
cd start-basic
npm install
npm run dev
```

將 `start-basic` 替換為下列任一範例的 slug 即可。

## 官方範例一覽

| 範例 | Slug | 說明 |
|------|------|------|
| [Basic](https://github.com/TanStack/router/tree/main/examples/react/start-basic) | `start-basic` | 最基礎的 Start 專案結構 |
| [Basic + Auth](https://github.com/TanStack/router/tree/main/examples/react/start-basic-auth) | `start-basic-auth` | 基礎認證流程 |
| [Counter](https://github.com/TanStack/router/tree/main/examples/react/start-counter) | `start-counter` | Server function 搭配計數器 |
| [Basic + React Query](https://github.com/TanStack/router/tree/main/examples/react/start-basic-react-query) | `start-basic-react-query` | 整合 TanStack Query |
| [Clerk Auth](https://github.com/TanStack/router/tree/main/examples/react/start-clerk-basic) | `start-clerk-basic` | Clerk 認證整合 |
| [Convex + Trellaux](https://github.com/TanStack/router/tree/main/examples/react/start-convex-trellaux) | `start-convex-trellaux` | Convex 後端 + Trello-like UI |
| [Supabase](https://github.com/TanStack/router/tree/main/examples/react/start-supabase-basic) | `start-supabase-basic` | Supabase 整合 |
| [Trellaux](https://github.com/TanStack/router/tree/main/examples/react/start-trellaux) | `start-trellaux` | Trello-like 看板應用 |
| [WorkOS](https://github.com/TanStack/router/tree/main/examples/react/start-workos) | `start-workos` | WorkOS 企業認證 |
| [Material UI](https://github.com/TanStack/router/tree/main/examples/react/start-material-ui) | `start-material-ui` | MUI 元件庫整合 |

每個範例頁面都附有 StackBlitz 預覽，可在瀏覽器中直接試用。

## 部署範例專案

**快速部署：** 範例頁面上有「Deploy to Netlify」按鈕，一鍵 clone + 部署。

**手動部署：** 使用 `gitpick` clone 後，依據目標平台的部署方式處理：

```bash
npx gitpick TanStack/router/tree/main/examples/react/EXAMPLE_SLUG my-new-project
cd my-new-project
npm install
npm run build
# 依據平台部署 build 產出
```

## TanStack Router 延伸範例

以下範例不是 Start 專屬，但有助於理解 TanStack Router 的進階功能：

- [Quickstart (file-based)](https://github.com/TanStack/router/tree/main/examples/react/quickstart-file-based) — file-based routing 最小範例
- [Kitchen Sink (file-based)](https://github.com/TanStack/router/tree/main/examples/react/kitchen-sink-file-based) — 涵蓋多數 Router 功能的完整範例
- [Authenticated Routes](https://github.com/TanStack/router/tree/main/examples/react/authenticated-routes) — 路由層級認證攔截
- [Scroll Restoration](https://github.com/TanStack/router/tree/main/examples/react/scroll-restoration) — 頁面捲動位置恢復
- [Deferred Data](https://github.com/TanStack/router/tree/main/examples/react/deferred-data) — 延遲載入資料模式
- [Navigation Blocking](https://github.com/TanStack/router/tree/main/examples/react/navigation-blocking) — 導航攔截（離開未儲存表單時的確認）
- [View Transitions](https://github.com/TanStack/router/tree/main/examples/react/view-transitions) — View Transitions API 整合

## 重點整理

- `npm create @tanstack/start@latest` 是最快的起手方式，CLI 提供互動式選項
- `npx gitpick` 可擷取特定範例目錄作為新專案起點
- 官方提供 10+ 個 Start 範例，涵蓋認證、React Query、Supabase 等常見整合
- 使用 CLI 或範例建立的專案可直接跳到 [Routing 指南](https://tanstack.com/start/latest/docs/framework/react/guide/routing) 繼續學習

---

下一篇：[專案初始化與相依套件](./03-project-setup.md)
