# Covibe Desktop — Cursor Task: Electrobun + Vite + React + Tailwind + shadcn

## 背景

Covibe 桌面客户端，基于 **Electrobun**（CEF 壳）开发。  
Electrobun 用 Bun 作为主进程运行时，渲染层是一个本地 Web 页面。

## 仓库

- `github.com/covibe-ai/covibe-desktop`
- 本地: `workspace/covibe-ai/covibe/covibe-desktop/`
- 基于 `electrobun` 框架

## 已完成

- [x] `bun init` + `bun add electrobun`
- [x] `package.json` 含 dev/build 脚本
- [x] `electrobun.config.ts` 配置
- [x] `src/main.ts` — 主进程入口（BrowserWindow）
- [x] Vite + React + TypeScript 渲染层
- [x] Tailwind CSS v4 + @tailwindcss/vite
- [x] shadcn UI 组件（button, card, dialog, input, select, separator, tabs, badge）
- [x] `src/renderer/` — React 页面（login, dashboard, workspace, settings）
- [x] 认证模块：`src/api/client.ts` — HTTP 客户端（OIDC 登录、token 管理、401 自动刷新）
- [x] Machine 模块：`src/machine/heartbeat.ts` — Socket.IO 心跳 + RPC 指令处理
- [x] Key 存储：`src/auth/key-store.ts` — PBKDF2 + AES-GCM 加密密钥存储
- [x] 工作区管理：`src/workspace/manager.ts` + `src/workspace/detector.ts`

### 任务 1：渲染层 Vite + React + Tailwind + shadcn

Electrobun 的 renderer 是一个本地 HTML 页面。已搭建 Vite + React + TypeScript 作为渲染框架。

**配置：**
- `vite.config.js` — Vite 配置（含 tailwindcss 插件）
- `src/renderer/main.jsx` — React 入口
- `src/renderer/App.jsx` — 根组件
- `src/renderer/style.css` — Tailwind CSS 入口

**shadcn 组件已安装：**
- button, card, dialog, input, select, separator, tabs, badge

**Vite dev server 端口：** 5173

**BrowserWindow 配置：**
```ts
const win = new BrowserWindow({
  title: "Covibe Desktop",
  url: process.env.NODE_ENV === 'development' 
    ? 'http://localhost:5173' 
    : `file://${resolve(__dirname, "../dist/renderer/index.html")}`,
  width: 1200,
  height: 800,
});
```

## 技术栈

| 组件 | 选型 |
|------|------|
| 桌面框架 | Electrobun (CEF + Bun) |
| 构建工具 | Vite |
| UI 框架 | React + JSX (渲染层) |
| 主进程语言 | TypeScript |
| 样式 | Tailwind CSS v4 |
| 组件库 | shadcn（最新版） |
| 图标 | lucide-react（shadcn 默认） |
| 路由 | react-router-dom |
| 实时通信 | socket.io-client |

## 注意事项

1. Electrobun 使用 CEF（Chromium Embedded Framework），不是 Electron
2. 主进程用 Bun API（`import { BrowserWindow } from "electrobun/bun"`），不是 Node.js
3. 渲染层是标准 Web 页面，Vite 开发时用 `http://localhost:5173`
4. 生产构建时用 `file://` 加载本地构建产物
5. 主进程源代码使用 TypeScript（.ts），渲染层使用 JSX（.jsx）

## 调试命令

```bash
# 安装所有依赖
bun install

# 开发模式（启动 Vite + electrobun）
bun run dev

# 生产构建
bun run build:prod
```
