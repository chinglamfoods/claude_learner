# 04：Vite 與 Start Plugin 設定

> 設定 `vite.config.ts`，正確配置 TanStack Start 的 Vite plugin 與 plugin 順序。

## 建立 vite.config.ts

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import tsConfigPaths from 'vite-tsconfig-paths'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  server: {
    port: 3000,
  },
  plugins: [
    tsConfigPaths(),
    tanstackStart(),
    // React 的 Vite plugin 必須放在 Start plugin 之後
    viteReact(),
  ],
})
```

## Plugin 順序

**`viteReact()` 必須放在 `tanstackStart()` 之後。** 這不是建議，而是硬性要求。

原因：`tanstackStart()` 會對 route 檔案進行程式碼轉換（code transformation），包括：
- 自動產生路由樹（`routeTree.gen.ts`）
- 處理 server function 的程式碼分割（將 `createServerFn` 的 handler 從 client bundle 中移除）
- 注入 SSR 與 hydration 所需的程式碼

如果 `viteReact()` 先執行 JSX transform，`tanstackStart()` 會收到已轉換的程式碼，導致上述處理無法正確運作。

## Plugin 功能說明

| Plugin | 功能 |
|--------|------|
| `tsConfigPaths()` | 讀取 `tsconfig.json` 的 `paths` 設定，讓 Vite 支援路徑別名 |
| `tanstackStart()` | TanStack Start 的核心 plugin — 路由產生、server function 處理、SSR 配置 |
| `viteReact()` | React 的 JSX transform、Fast Refresh（HMR） |

## tanstackStart() 選項

`tanstackStart()` 接受配置物件，常用選項包括：

```ts
tanstackStart({
  // 自訂路由目錄（預設為 'src/routes'）
  routesDirectory: 'src/routes',
  // 自訂產生的路由樹檔案位置
  generatedRouteTree: 'src/routeTree.gen.ts',
})
```

多數情況下使用預設值即可，不需傳入任何參數。

## 開發伺服器設定

`server.port` 設定 dev server 的埠號。如果 3000 已被佔用，Vite 會自動選擇下一個可用埠號並在終端機顯示。

其他常用的 dev server 設定：

```ts
export default defineConfig({
  server: {
    port: 3000,
    // 允許區域網路內其他裝置存取（行動裝置測試用）
    host: true,
    // 自訂 HTTPS（搭配 mkcert 等工具）
    // https: true,
  },
  // ...
})
```

## 重點整理

- `vite.config.ts` 需引入三個 plugin：`tsConfigPaths`、`tanstackStart`、`viteReact`
- **Plugin 順序至關重要**：`tanstackStart()` 必須在 `viteReact()` 之前
- `tanstackStart()` 負責路由產生、server function 的 code splitting、以及 SSR 配置
- 預設設定已能滿足多數需求，通常不需額外傳入配置

---

下一篇：[Router 設定與根元件](./05-router-and-root.md)
