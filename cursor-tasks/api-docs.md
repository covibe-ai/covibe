# Covibe API 文档

## 概述

Covibe 由两个服务提供 API：

| 服务 | 基础 URL | 职责 | 协议 |
|------|---------|------|------|
| **happy-server** | `/v1/*`, `/v2/*`, `/v3/*` | Session/消息/Machine（上游原生） | HTTP + WebSocket |
| **covibe-server** | `/covibe_api/v1/*` | Workspace/会员/用户/OAuth | HTTP REST (DRF) |

---

## 一、covibe-server HTTP API

所有请求需在 Header 携带 Bearer Token（SimpleJWT）：
```
Authorization: Bearer <access_token>
```

### 1.1 认证

#### POST /covibe_api/v1/auth/oidc/login
OIDC 第三方登录。

```json
// Request
{ "id_token": "...", "provider": "google" }

// Response 200
{
  "access": "eyJ...",        // JWT access token (有效期24h)
  "refresh": "eyJ...",       // JWT refresh token (有效期30天)
  "user": {
    "uuid": "abc123",
    "email": "user@example.com",
    "nickname": "User",
    "is_new": true             // 是否首次登录（需要设置密钥）
  }
}

// Response 401
{ "error": "Invalid token" }
```

#### POST /covibe_api/v1/auth/token/refresh
刷新 access token。

```json
// Request
{ "refresh": "eyJ..." }

// Response 200
{ "access": "eyJ..." }
```

#### POST /covibe_api/v1/auth/register-key
注册用户的加密密钥对。

```json
// Request
{
  "public_key": "base64...",
  "encrypted_private_key": "base64..."
  // publicKey 是明文，encryptedPrivateKey 是 AES-GCM 加密后的私钥
  // 加密密码只有用户自己知道，服务器不存储
}

// Response 200
{ "success": true }
```

#### GET /covibe_api/v1/users/me
获取当前用户信息。

```json
// Response 200
{
  "uuid": "abc123",
  "email": "user@example.com",
  "nickname": "User",
  "avatar": "",
  "oidc_sub": "google-12345",
  "has_key": true,
  "effective_max_sessions": 5,
  "effective_max_workspaces": 3,
  "effective_idle_minutes": 60,
  "tier": { "name": "专业版", "expired_at": "2026-06-30T00:00:00Z" },
  "date_joined": "2026-01-01T00:00:00Z"
}
```

### 1.2 Workspace

#### GET /covibe_api/v1/workspaces
获取用户所有 workspace。

```json
// Response 200
[{
  "uuid": "wks_001",
  "name": "我的项目",
  "root_dir": "/Users/me/projects/my-app",
  "tool_type": "cursor",
  "is_active": true,
  "idle_timeout_minutes": 30,
  "max_sessions": 3,
  "created_at": "2026-01-01T00:00:00Z",
  "machine": { "uuid": "mac_001", "name": "我的 MacBook" }
}]
```

#### POST /covibe_api/v1/workspaces
创建 workspace。

```json
// Request
{
  "name": "我的项目",
  "root_dir": "/Users/me/projects/my-app",
  "tool_type": "cursor",
  "machine_uuid": "mac_001",
  "idle_timeout_minutes": 30,
  "max_sessions": 3
}

// Response 201
{ "uuid": "wks_002", ... }
```

#### PATCH /covibe_api/v1/workspaces/:uuid
更新 workspace。

```json
// Request
{ "name": "新名称", "idle_timeout_minutes": 15 }

// Response 200
{ "uuid": "wks_001", ... }
```

#### DELETE /covibe_api/v1/workspaces/:uuid
删除 workspace（同时通知关联 machine 关闭 session）。

```json
// Response 200
{ "success": true }
```

### 1.3 Machine

#### POST /covibe_api/v1/machines/register
注册/更新本机信息（由 covibe-desktop 调用）。

```json
// Request
{
  "machine_id": "covibe-xxxxx",
  "name": "我的 MacBook",
  "host": "MacBook-Pro.local",
  "platform": "darwin",
  "cli_version": "1.0.0",
  "public_key": "base64...",
  "available_tools": ["cursor", "claude", "codex"],
  "system_info": {
    "cpu_usage": 45.2,
    "memory_usage": 60.1,
    "total_memory_gb": 16,
    "uptime_hours": 120
  }
}

// Response 200
{
  "uuid": "mac_001",
  "is_online": true,
  "active_session_count": 3
}
```

#### GET /covibe_api/v1/machines
获取用户的机器列表（含在线状态）。

```json
// Response 200
[{
  "uuid": "mac_001",
  "name": "我的 MacBook",
  "host": "MacBook-Pro.local",
  "platform": "darwin",
  "is_online": true,
  "last_active_at": "2026-01-15T10:30:00Z",
  "active_session_count": 2,
  "system_info": { "cpu_usage": 45, "memory_usage": 60 }
}]
```

#### GET /covibe_api/v1/machines/:uuid/directories
列出机器某目录下的内容（用于 workspace 设置时选择目录）。
需要目标 machine 在线，通过 RPC 转发到该机器执行 `ls`。

```json
// Query
GET /covibe_api/v1/machines/mac_001/directories?path=/Users

// Response 200
{
  "path": "/Users",
  "entries": [
    { "name": "me", "type": "dir" },
    { "name": "shared", "type": "dir" }
  ]
}
```

### 1.4 Session 配额管理

#### GET /covibe_api/v1/sessions/count
```json
// Response
{ "active": 2, "max": 5, "pct": 40 }
```

### 1.5 会员

#### GET /covibe_api/v1/tiers
会员等级列表。

```json
[{
  "uuid": "tier_001",
  "name": "免费版",
  "price_per_month_minor": 0,
  "max_sessions": 1,
  "max_workspaces": 1,
  "max_idle_minutes": 30
}, {
  "name": "专业版",
  "price_per_month_minor": 4900,
  "max_sessions": 5,
  ...
}]
```

#### POST /covibe_api/v1/orders/membership
创建会员购买订单。

```json
// Request
{ "tier_uuid": "tier_002" }

// Response 201
{
  "uuid": "ord_001",
  "amount_minor": 4900,
  "original_amount_minor": 4900,
  "discount_minor": 0,
  "wechat_pay_params": { "code_url": "weixin://..." }
}
```

---

## 二、happy-server HTTP API

Happy-server 的原生 API，大部分被 covibe-server 代理拦截。这里列出桌面端和 App 直接需要的。

### 2.1 Session（原生）

#### POST /v1/sessions
创建 session（通常由 happy CLI 调用，配额检查由 covibe-server 在代理层处理）。

```json
// Request
{
  "tag": "uuid",
  "metadata": "<base64 加密>",
  "agentState": "<base64 加密>",
  "dataEncryptionKey": "<base64 加密>"
}

// Response 200
{
  "session": {
    "id": "sess_001",
    "tag": "uuid",
    "metadata": "<base64 加密>",
    "metadataVersion": 1,
    "agentState": "<base64 加密>",
    "active": true,
    "lastActiveAt": "2026-01-15T10:00:00Z"
  }
}
```

#### GET /v1/sessions
获取用户所有 session。

```json
// Response
{ "sessions": [{ "id": "sess_001", "tag": "...", "active": true, ... }] }
```

---

## 三、WebSocket 接口

Covibe 有**三个不同作用域**的 WebSocket 连接，各有不同的认证和目的。

### 3.1 Machine WebSocket（covibe-desktop → happy-server）

**连接地址：** `wss://api.covibe.ai/v1/updates?token=<bearer>`

**认证方式：** Query param `token` + Socket.IO auth `{ machineId }`

**连接类型：** `machine-scoped`

**事件：**

| 方向 | 事件 | 说明 | 频率 |
|------|------|------|------|
| → | `machine-alive` | 心跳 `{ machineId, time }` | 每 60s |
| → | `machine-update-state` | 更新 daemon 状态 `{ machineId, daemonState, expectedVersion }` | 变化时 |
| → | `machine-update-metadata` | 更新机器元数据 `{ machineId, metadata, expectedVersion }` | 变化时 |
| ← | `rpc-request` | 服务器指令 `{ method, params: { happySessionId, reason } }` | 按需 |
| | | method 包括: `spawnSession`, `killSession`, `resumeSession`, `requestShutdown` | |

**RPC 指令示例：**
```json
// Server → Desktop: 杀死 session
{
  "method": "killSession",
  "params": { "happySessionId": "sess_001", "reason": "quota_exceeded" }
}

// Desktop → Server: 确认执行
// (通过 machine-update-state 回写状态)
```

### 3.2 User WebSocket（covibe-app / covibe-dashboard → happy-server）

**连接地址：** 同上

**认证方式：** Bearer token（同 HTTP）

**连接类型：** `user-scoped`

**事件：**

| 方向 | 事件 | 说明 |
|------|------|------|
| → | `session-alive` | 监听 session 心跳 `{ sid, time }` |
| ← | `update` | session/machine 状态变更推送 `{ type: "new-session"|"update-session"|"machine-status", ... }` |

用户作用域的连接**只能接收更新，不能发送指令**。所有控制操作通过 HTTP API 发起。

### 3.3 Session WebSocket（happy CLI → happy-server）

**连接地址：** 同上

**认证方式：** Bearer token + session ID in auth

**连接类型：** `session-scoped`

**事件：**

| 方向 | 事件 | 说明 |
|------|------|------|
| → | `message` | 加密消息 `{ sid, message, localId }` |
| → | `update-metadata` | 更新 metadata `{ sid, metadata, expectedVersion }` |
| → | `update-state` | 更新 agent state `{ sid, agentState, expectedVersion }` |
| → | `session-alive` | session 心跳 `{ sid, time }` |
| → | `session-end` | session 结束 `{ sid, time }` |
| ← | `update` | 收到消息推送（从手机/Web 发送的） |

---

## 四、不同客户端的连接拓扑

```
covibe-desktop ←→ happy-server (machine-scoped WS)
     │                      │
     │  HTTP API            │ user-scoped WS
     ▼                      ▼
covibe-server ◄────── covibe-dashboard (Web)
     │                      │
     │  HTTP API            │ user-scoped WS
     ▼                      ▼
covibe-app (手机) ───── happy-server (user-scoped WS)
```

| 客户端 | 连接对象 | 用途 |
|--------|---------|------|
| **covibe-desktop** | happy-server WS (machine) | 心跳、RPC 指令 |
| **covibe-desktop** | covibe-server HTTP | 登录、workspace 同步、机器注册 |
| **covibe-dashboard** | happy-server WS (user) | 接收 session/machine 状态更新 |
| **covibe-dashboard** | covibe-server HTTP | 用户管理、设置、会员 |
| **covibe-app** | happy-server WS (user) | 接收 session 更新、发送 message |
| **happy CLI** | happy-server WS (session) | 加密消息收发、状态同步 |
