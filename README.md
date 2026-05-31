# Covibe

Vibecoding 远程控制平台。基于 [Happy](https://github.com/slopus/happy) (MIT) 构建，零信任 E2E 加密。

## 仓库结构

```
covibe/
├── covibe-happy/                # [submodule] fork of slopus/happy
│   ├── packages/happy-cli/      # CLI 主程序（稍作修改）
│   ├── packages/happy-server/   # 后端服务（复用不改）
│   └── packages/happy-wire/     # 共享协议类型（复用不改）
├── covibe-dashboard/            # [submodule] SaaS 管理后端 (Django DRF)
├── covibe-desktop/              # [submodule] 桌面 GUI (Electron/Tauri)
├── covibe-app/                  # [submodule] Web + 手机客户端
└── deploy/                      # 部署编排
    ├── docker-compose.yml
    └── k8s/
```

## 快速开始

```bash
# 克隆所有代码
git clone --recursive git@github.com:covibe-ai/covibe.git
cd covibe

# 本地一键构建+启动
cd deploy && docker compose up --build
```

## 架构

| 组件 | 技术栈 | 运行位置 |
|------|--------|---------|
| covibe-dashboard | Django + DRF + Celery | 云端服务器 |
| happy-server | Node.js + Fastify + Socket.IO | 云端服务器 |
| covibe-desktop | Electron / Tauri | 用户电脑 |
| happy-cli | Node.js (Bun 编译) | 用户电脑（由 desktop 管理） |
| covibe-app | React / Expo | 浏览器 / 手机 |

## License

MIT
