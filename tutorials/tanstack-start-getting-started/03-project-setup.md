# 03：專案初始化與相依套件

> 從空目錄開始，設定 TypeScript 並安裝 TanStack Start 所需的全部套件。

## 建立專案目錄

```bash
mkdir myApp
cd myApp
npm init -y
```

## TypeScript 設定

建立 `tsconfig.json`，以下是 TanStack Start 所需的最低配置：

```json
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "moduleResolution": "Bundler",
    "module": "ESNext",
    "target": "ES2022",
    "skipLibCheck": true,
    "strictNullChecks": true
  }
}
```

各選項的用途：

| 選項 | 說明 |
|------|------|
| `jsx: "react-jsx"` | 使用 React 17+ 的自動 JSX transform，不需手動 `import React` |
| `moduleResolution: "Bundler"` | 配合 Vite 的模組解析策略，支援 package.json `exports` 欄位 |
| `module: "ESNext"` | 輸出 ES modules，由 Vite 處理最終 bundling |
| `target: "ES2022"` | 支援 top-level await、`cause` on Error 等現代語法 |
| `strictNullChecks: true` | 型別層級的 null safety — 配合 TanStack Router 的型別推導使用 |

> **注意：** 不要啟用 `verbatimModuleSyntax`。這個選項會導致 server bundle 的程式碼洩漏到 client bundle 中，造成不預期的安全與效能問題。

## 安裝相依套件

TanStack Start 的核心相依分為四個部分：

**1. Start 與 Router（核心）：**

```bash
npm i @tanstack/react-start @tanstack/react-router
```

**2. React：**

```bash
npm i react react-dom
```

**3. Vite 與 React plugin（開發工具）：**

```bash
npm i -D vite @vitejs/plugin-react
```

`@vitejs/plugin-react` 也可替換為效能更好的替代方案：
- `@vitejs/plugin-react-swc` — 使用 SWC 編譯，速度較快
- `@vitejs/plugin-react-oxc` — 使用 Oxc 編譯，目前最快的選項

**4. TypeScript 與型別定義：**

```bash
npm i -D typescript @types/react @types/react-dom @types/node vite-tsconfig-paths
```

`vite-tsconfig-paths` 讓 Vite 讀取 `tsconfig.json` 中的 `paths` 設定，支援路徑別名（如 `@/components/Button`）。

## 更新 package.json

設定 `"type": "module"` 以啟用 ES modules，並加入 dev 與 build 指令：

```jsonc
{
  // ... 其他欄位
  "type": "module",
  "scripts": {
    "dev": "vite dev",
    "build": "vite build"
  }
}
```

`"type": "module"` 讓 Node.js 以 ESM 模式解析 `.js` 檔案，這是 Vite 與 TanStack Start 的必要設定。

## 完成後的檔案結構

到這一步，專案目錄應包含：

```
myApp/
├── node_modules/
├── package.json
├── package-lock.json
└── tsconfig.json
```

下一步將設定 Vite 配置檔與 TanStack Start plugin。

## 重點整理

- `tsconfig.json` 需設定 `jsx: "react-jsx"` 與 `moduleResolution: "Bundler"` 以配合 Vite
- 避免啟用 `verbatimModuleSyntax`，它會造成 server code 洩漏至 client bundle
- 核心套件為 `@tanstack/react-start`、`@tanstack/react-router`、`react`、`react-dom`
- React 的 Vite plugin 有三種選擇：`plugin-react`（Babel）、`plugin-react-swc`、`plugin-react-oxc`
- `package.json` 必須設定 `"type": "module"`

---

下一篇：[Vite 與 Start Plugin 設定](./04-vite-configuration.md)
