# Covibe API 文档

## 概述

Covibe 由两个服务提供 API：

| 服务 | 基础 URL | 职责 | 协议 |
|------|---------|------|------|
| **happy-server** | `/v1/*`, `/v2/*`, `/v3/*` | Session/消息/Machine（上游原生） | HTTP + WebSocket |
| **covibe-server** | `/covibe_api/v1/*` | 订单/支付/会员/用户/OAuth | HTTP REST (DRF) |

> ⚠️ **注意**: 以下 API 文档记录了**已设计但尚未全部实现**的端点。
> 已实现的 endpoints 标记为 ✅，未实现的标记为 📋。

---

## 一、covibe-server HTTP API

所有请求需在 Header 携带 Bearer Token（SimpleJWT）：
```
Authorization: Bearer ***
```

### 1.1 认证 📋（未实现）

以下认证 endpoints **尚未实现**：

#### POST /covibe_api/v1/auth/oidc/login 📋
OIDC 第三方登录。

```json
// Request
{ "id_token": "...", "provider": "google" }

// Response 200
{
  "access": "eyJ...",
  "refresh": "eyJ...",
  "user": {
    "uuid": "abc123",
    "email": "user@example.com",
    "nickname": "User",
    "is_new": true
  }
}
```

#### POST /covibe_api/v1/auth/token/refresh 📋
刷新 access token。

#### POST /covibe_api/v1/auth/register-key 📋
注册用户的加密密钥对。

#### GET /covibe_api/v1/users/me 📋
获取当前用户信息。

```json
// Response 200
{
  "uuid": "abc123",
  "email": "user@example.com",
  "nickname": "User",
  "avatar": "",
  "effective_max_sessions": 5,
  "effective_max_workspaces": 3,
  "effective_idle_minutes": 60,
  "tier": { "name": "专业版", "expired_at": "2026-06-30T00:00:00Z" },
  "date_joined": "2026-01-01T00:00:00Z"
}
```

> 注意：`tier` 字段格式匹配 `UserSerializer.get_tier()` 实现（返回 dict with name+expired_at or null）

### 1.2 Workspace 📋（未实现）

以下 workspace endpoints **尚未实现**：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/covibe_api/v1/workspaces` | 获取 workspace 列表 |
| POST | `/covibe_api/v1/workspaces` | 创建 workspace |
| PATCH | `/covibe_api/v1/workspaces/:uuid` | 更新 workspace |
| DELETE | `/covibe_api/v1/workspaces/:uuid` | 删除 workspace |

### 1.3 Machine 📋（未实现）

以下 machine endpoints **尚未实现**：

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/covibe_api/v1/machines/register` | 注册/更新本机信息 |
| GET | `/covibe_api/v1/machines` | 获取机器列表 |
| GET | `/covibe_api/v1/machines/:uuid/directories` | 列出目录内容 |

### 1.4 Session 配额管理 📋（未实现）

```
GET /covibe_api/v1/sessions/count
// Response
{ "active": 2, "max": 5, "pct": 40 }
```

### 1.5 订单 ✅（已实现）

#### POST /covibe_api/v1/orders/
创建通用订单。

```json
// Request
{
  "kind": "generic",
  "params": {},
  "amount_minor": 4900,
  "payment_platform": "wechatpay_native"
}

// Response 201
{
  "uuid": "ord_001",
  "buyer": 1,
  "status": 10,
  "payment_platform": "wechatpay_native",
  "amount_minor": 4900,
  "item": {
    "uuid": "...",
    "kind": "generic",
    "params": {}
  }
}
```

| 字段 | 说明 |
|------|------|
| `kind` | 品类标识（默认 `generic`） |
| `params` | 业务参数 JSON |
| `amount_minor` | 金额（分），必填 |
| `payment_platform` | `wechatpay_native` 或 `wechatpay_jsapi` |

#### POST /covibe_api/v1/orders/{uuid}/pay/wechat/prepay/
微信预支付（需先创建订单）。

```json
// Request (empty body for native, requires WeixinUser for jsapi)
{}

// Response 200 (Native)
{ "code_url": "weixin://wxpay/bizpayurl?pr=..." }

// Response 200 (JSAPI)
{
  "appId": "wx...",
  "timeStamp": "...",
  "nonceStr": "...",
  "package": "prepay_id=...",
  "signType": "RSA",
  "paySign": "..."
}

// Error 400
{ "code": "WECHAT_USER_NOT_BOUND", "detail": "用户未绑定微信" }
```

#### POST /covibe_api/v1/orders/{uuid}/check_payment/
主动查单（2s 限频，支持 WeChat Native/JSAPI）。

```json
// Response 200
{
  "uuid": "ord_001",
  "status": 30,
  "paid_amount_minor": 4900,
  "paid_at": "2026-01-15T10:00:00Z"
}
```

### 1.6 微信支付回调 ✅（已实现）

#### POST /covibe_api/v1/payments/wechat/pay/notify/
微信支付异步通知回调。

#### POST /covibe_api/v1/payments/wechat/refund/notify/
微信退款异步通知回调。

### 1.7 会员 📋（未实现）

以下会员 endpoints **尚未实现**：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/covibe_api/v1/tiers` | 会员等级列表 |
| POST | `/covibe_api/v1/orders/membership` | 创建会员购买订单 |

会员购买将使用通用订单创建流程，通过 `kind` 和 `params` 区分业务类型，
支付成功后经由 `OrderItem.fulfill()` 调用 `Subscription.extend_or_create()`。

---

## 二、happy-server HTTP API

Happy-server 的原生 API，covibe-server 不拦截，直接透传。

### 2.1 完整路由列表

| 模块 | 方法 | 路由 | 说明 |
|------|------|------|------|
| Auth | POST | `/v1/auth` | 公钥签名 → Bearer Token |
| Auth | POST | `/v1/auth/request` | 发起配对请求 |
| Auth | GET | `/v1/auth/request/status` | 查询配对状态 |
| Auth | POST | `/v1/auth/response` | 批准配对 |
| Auth | POST | `/v1/auth/account/request` | 账号配对请求 |
| Auth | POST | `/v1/auth/account/response` | 账号配对批准 |
| Session | GET | `/v1/sessions` | 会话列表 |
| Session | POST | `/v1/sessions` | 创建会话 |
| Session | GET | `/v2/sessions` | v2 会话列表 |
| Session | GET | `/v2/sessions/active` | v2 活跃会话 |
| Session | GET | `/v1/sessions/:id/messages` | 历史消息 |
| Session | POST | `/v1/sessions/:id/archive` | 归档会话 |
| Session | DELETE | `/v1/sessions/:id` | 删除会话 |
| Session | POST | `/v3/sessions/:id/messages` | v3 消息协议 |
| Session | POST | `/v1/sessions/:id/push-event` | 推送事件 |
| Attachment | POST | `/v1/sessions/:id/attachments/request-upload` | 请求上传 |
| Attachment | PUT | `/v1/sessions/:id/attachments/:file` | 上传文件 |
| Attachment | GET | `/v1/sessions/:id/attachments/:file` | 下载文件 |
| Machine | POST | `/v1/machines` | 注册机器 |
| Machine | GET | `/v1/machines` | 机器列表 |
| Machine | GET | `/v1/machines/:id` | 机器详情 |
| Machine | DELETE | `/v1/machines/:id` | 删除机器 |
| Artifact | GET | `/v1/artifacts` | 工件列表 |
| Artifact | POST | `/v1/artifacts` | 创建工件 |
| Artifact | GET | `/v1/artifacts/:id` | 工件详情 |
| Artifact | POST | `/v1/artifacts/:id` | 更新工件 |
| Artifact | DELETE | `/v1/artifacts/:id` | 删除工件 |
| KV | GET | `/v1/kv/:key` | 读取 KV |
| KV | GET | `/v1/kv` | KV 列表 |
| KV | POST | `/v1/kv` | 写入 KV |
| KV | POST | `/v1/kv/bulk` | 批量写入 |
| Connect | GET | `/v1/connect/github/params` | GitHub OAuth 参数 |
| Connect | GET | `/v1/connect/github/callback` | GitHub OAuth 回调 |
| Connect | POST | `/v1/connect/github/webhook` | GitHub Webhook |
| Connect | DELETE | `/v1/connect/github` | 断开 GitHub |
| Connect | POST | `/v1/connect/:vendor/register` | 注册 AI vendor token |
| Connect | GET | `/v1/connect/:vendor/token` | 读取 vendor token |
| Connect | DELETE | `/v1/connect/:vendor` | 删除 vendor token |
| Connect | GET | `/v1/connect/tokens` | 所有已连接 vendor |
| Account | GET | `/v1/account/profile` | 个人资料 |
| Account | GET | `/v1/account/settings` | 读取设置 |
| Account | POST | `/v1/account/settings` | 更新设置 |
| User | GET | `/v1/user/:id` | 用户信息 |
| User | GET | `/v1/user/search` | 搜索用户 |
| Push | POST | `/v1/push-tokens` | 注册推送 Token |
| Push | DELETE | `/v1/push-tokens/:token` | 注销推送 Token |
| Push | GET | `/v1/push-tokens` | 推送 Token 列表 |
| Feed | GET | `/v1/feed` | 社交信息流 |
| Friends | POST | `/v1/friends/add` | 添加好友 |
| Friends | POST | `/v1/friends/remove` | 删除好友 |
| Friends | GET | `/v1/friends` | 好友列表 |
| Voice | GET | `/v1/voice/conversations` | 语音会话列表 |
| Voice | GET | `/v1/voice/usage` | 语音用量 |
| AccessKey | POST | `/v1/access-keys/:sessionId/:machineId` | 创建访问密钥 |
| AccessKey | GET | `/v1/access-keys/:sessionId/:machineId` | 读取访问密钥 |
| AccessKey | DELETE | `/v1/access-keys/:sessionId/:machineId` | 删除访问密钥 |
| Version | POST | `/v1/version` | 版本检查 |
| Usage | GET | `/v1/usage/query` | 用量查询 |
| Health | GET | `/health` | 健康检查 |

Covibe 有**三个不同作用域**的 WebSocket 连接，各有不同的认证和目的。

### 3.1 Machine WebSocket（covibe-desktop → happy-server）

**连接地址：** `wss://api.covibe.ai/v1/updates?token=<bearer>`

**认证方式：** Socket.IO auth `{ token }`（desktop 端使用 bearer token）

**事件：**

| 方向 | 事件 | 说明 | 频率 |
|------|------|------|------|
| → | `machine-alive` | 心跳 `{ machineId, time }` | 每 60s |
| → | `machine-update-state` | 更新 daemon 状态 | 变化时 |
| ← | `rpc-request` | 服务器指令 | 按需 |

**RPC method 包括:** `spawnSession`, `killSession`, `requestShutdown`

### 3.2 User WebSocket（covibe-dashboard → happy-server）

**事件：**

| 方向 | 事件 | 说明 |
|------|------|------|
| ← | `update` | session/machine 状态变更推送 |

### 3.3 Session WebSocket（happy CLI → happy-server）

**事件：**

| 方向 | 事件 | 说明 |
|------|------|------|
| → | `message` | 加密消息 |
| → | `update-metadata` | 更新 metadata |
| → | `update-state` | 更新 agent state |
| → | `session-alive` | session 心跳 |
| → | `session-end` | session 结束 |
| ← | `update` | 收到消息推送 |

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
| **covibe-app** | happy-server WS (user) | 接收 session 更新 |
| **happy CLI** | happy-server WS (session) | 加密消息收发、状态同步 |
