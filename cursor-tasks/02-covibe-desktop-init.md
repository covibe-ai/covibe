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

## 待完成

### 任务 1：渲染层 Vite + React + Tailwind + shadcn

Electrobun 的 renderer 是一个本地 HTML 页面。我们需要搭建 Vite + React + JS 作为渲染框架。

**步骤：**

1. 在项目根目录安装 Vite + React：
   ```bash
   bun add -d vite @vitejs/plugin-react
   bun add react react-dom
   ```

2. 创建 `vite.config.js`：
   ```js
   import { defineConfig } from 'vite'
   import react from '@vitejs/plugin-react'
   export default defineConfig({
     plugins: [react()],
     server: { port: 5173 },
     build: { outDir: 'dist/renderer' }
   })
   ```

3. 创建 `index.html`（在根目录）—— Vite 入口 HTML

4. 创建 `src/renderer/main.jsx` —— React 入口

5. 创建 `src/renderer/App.jsx` —— 根组件

6. 修改 `src/main.ts`，将 BrowserWindow 的 `url` 指向 Vite dev server：
   ```ts
   const win = new BrowserWindow({
     title: "Covibe Desktop",
     // 开发时指向 Vite 开发服务器
     url: process.env.NODE_ENV === 'development' 
       ? 'http://localhost:5173' 
       : `file://${import.meta.dir}/../dist/renderer/index.html`,
     width: 1200,
     height: 800,
   });
   ```

### 任务 2：Tailwind CSS v4

1. 安装 Tailwind CSS v4：
   ```bash
   bun add tailwindcss @tailwindcss/vite
   ```

2. 在 `vite.config.js` 添加 tailwindcss 插件

3. 创建 `src/renderer/style.css` 引入 Tailwind

### 任务 3：shadcn 最新版

1. 初始化 shadcn（最新版）：
   ```bash
   bunx shadcn@latest init
   ```
   选择：React + JS + Tailwind + 默认配色

2. 添加 UI 组件，直接按 shadcn 官方文档的最新命令操作

### 任务 4：Electrobun 开发调试

1. `package.json` 已有 `bun run dev` 脚本（先 build 再启动 electrobun）
2. 本地开发时需要同时启动 Vite dev server 和 electrobun
3. 建议在 dev 脚本中先启动 Vite 再启动 electrobun：
   ```
   "dev": "concurrently \"vite\" \"electrobun dev\""
   ```

## 技术栈

| 组件 | 选型 |
|------|------|
| 桌面框架 | Electrobun (CEF + Bun) |
| 构建工具 | Vite |
| UI 框架 | React + JS (not TypeScript) |
| 样式 | Tailwind CSS v4 |
| 组件库 | shadcn（最新版） |
| 图标 | lucide-react（shadcn 默认） |

## 注意事项

1. Electrobun 使用 CEF（Chromium Embedded Framework），不是 Electron
2. 主进程用 Bun API（`import { BrowserWindow } from "electrobun/bun"`），不是 Node.js
3. 渲染层是标准 Web 页面，Vite 开发时用 `http://localhost:5173`
4. 生产构建时用 `file://` 加载本地构建产物
5. 严格按照 shadcn 官方最新文档初始化，不要用旧版命令
6. 本项目**先不开发实际业务功能**，只搭好脚手架框架

## 调试命令

```bash
# 安装所有依赖
bun install

# 开发模式（启动 Vite + electrobun）
bun run dev

# 生产构建
bun run build:prod
```
