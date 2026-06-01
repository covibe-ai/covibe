# Covibe Deploy

## 架构总览

```
                      ┌──────────────┐
                      │  Nginx 80    │ ← 反向代理 + 静态文件
                      │  (in server) │
                      └──────┬───────┘
                             │
               ┌─────────────┼─────────────┐
               │             │             │
         ┌─────┴─────┐ ┌────┴────┐ ┌──────┴──────┐
         │ Daphne    │ │ Celery  │ │ Celery Beat  │
         │ 8080      │ │ Worker  │ │ (定时任务)    │
         │ (ASGI)    │ │         │ │              │
         └─────┬─────┘ └─────────┘ └─────────────┘
               │
               │ HTTP API         WebSocket
               │                  (Socket.IO)
         ┌─────┴─────┐      ┌─────┴─────┐
         │ covibe-   │      │ happy-    │
         │ server    │      │ server    │
         │ (Django)  │      │ (:3005)   │
         │ + Nginx   │      │           │
         └─────┬─────┘      └─────┬─────┘
               │                  │
         ┌─────┴──────────────────┴─────┐
         │         PostgreSQL           │
         └──────────────────────────────┘
                    │
               ┌────┴────┐
               │  Redis  │ ← Celery broker + Socket.IO pub/sub
               └─────────┘
```

> 注意：Nginx 和 Daphne 都运行在 `covibe-server` 容器内。Nginx 监听 80 端口，反向代理到 Daphne (8080) 和 happy-server (3005)。

## 快速启动

```bash
# 1. 复制环境变量并修改
cp deploy/.env.example .env
# 填入 HANDY_MASTER_SECRET, DJANGO_SECRET_KEY, ROOT_PASSWORD

# 2. 开发模式（热重载）
docker compose -f deploy/docker-compose.yml -f deploy/docker-compose.dev.yml up

# 3. 生产模式
docker compose -f deploy/docker-compose.yml up -d

# 4. 查看日志
docker compose -f deploy/docker-compose.yml logs -f
```

## 服务说明

| 服务 | 端口 | 说明 |
|------|------|------|
| **nginx** (in covibe-server) | 80 | 反向代理 + 静态文件 + dashboard |
| **happy-server** | 3005 | Socket.IO 实时通信 + Session CRUD |
| **Daphne** (in covibe-server) | 8080 | Django ASGI (内部，仅 nginx 可见) |
| **Celery Worker** | - | 异步任务执行 |
| **Celery Beat** | - | 定时任务调度 |
| **PostgreSQL** | 5432 | 数据库 |
| **Redis** | 6379 | 缓存 + 消息队列 |

> 注意：Celery Flower 监控面板当前未在 docker-compose 中配置。

## 环境变量

| 变量 | 必需 | 说明 | 默认值 |
|------|------|------|--------|
| `HANDY_MASTER_SECRET` | ✅ | Happy Server 主密钥（任何字符串） | - |
| `DJANGO_SECRET_KEY` | ✅ | Django 密钥 | - |
| `ROOT_PASSWORD` | ✅ | 管理员密码 | - |
| `DB_PASSWORD` | ❌ | 数据库密码 | covibe |
| `DJANGO_ALLOWED_HOSTS` | ❌ | 允许的域名 | localhost,api.covibe.ai |
| `CORS_ALLOWED_ORIGINS` | ❌ | 跨域来源 | https://covibe.ai,https://dashboard.covibe.ai |
| `SUPERUSER_EMAIL` | ❌ | 超级管理员邮箱 | (见 .env.example) |

## 扩展方案

### 水平扩展 happy-server

```
docker compose up -d --scale happy-server=3

前提：
  - happy-server 使用 Redis 作为 Socket.IO 适配器（已有 `@socket.io/redis-streams-adapter`）
  - 共享 PostgreSQL（已有）
```

### 水平扩展 covibe-server

```
docker compose up -d --scale covibe-server=3

前提：
  - 无状态 Django，只需调整 PG 连接池
  - Celery worker 独立扩展：--scale covibe-worker=4
```

### 数据库扩展

```
小型（<10万用户）：单机 PostgreSQL + PgBouncer
中型（<100万）：主从复制 + 读写分离  
大型（>100万）：分表（按 accountId hash）
```

### 完整分布式架构

```
           ┌──────────┐
           │  Nginx   │ ← SSL termination + 负载均衡
           │ (LB)     │
           └────┬─────┘
                │
     ┌──────────┼──────────┐
     │          │          │
┌────┴───┐ ┌───┴────┐ ┌───┴────┐
│ happy  │ │ happy  │ │ happy  │ ← 水平扩展 (3+)
│ svr-1  │ │ svr-2  │ │ svr-3  │
│ :3005  │ │ :3005  │ │ :3005  │
└───┬────┘ └───┬────┘ └───┬────┘
    │          │          │
    └──────────┼──────────┘
               │
          ┌────┴────┐
          │  Redis  │ ← Socket.IO pub/sub
          └─────────┘

┌──────────┐  ┌──────────┐  ┌──────────┐
│ Django-1 │  │ Django-2 │  │ Celery   │
│ (stateless)│  │ (stateless)│  │ Worker-1│
│ :8080    │  │ :8080    │  │          │
└────┬─────┘  └────┬─────┘  └──────────┘
     │             │
     └──────┬──────┘
            │
       ┌────┴────┐
       │   PG    │
       │ Master  │
       └────┬────┘
            │
       ┌────┴────┐
       │ PG      │
       │ Replica │    ← 只读查询
       └─────────┘
```
