# Covibe Server — Cursor Task: REST API Layer

## 背景

covibe-server 需要暴露 RESTful API 供 covibe-dashboard（前端）和 covibe-desktop（桌面客户端）调用。  
部分请求需要转发到 Happy Server，部分由 Django 直接处理。

## API 前缀

- **业务 API**: `/covibe_api/v1/*`（由 covibe-server 处理）  
- **Happy API 代理**: `/v1/*`（由反代转发到 happy-server，happy 原生使用 `/v1/` 前缀）  
- **Admin**: `/admin/*`（Unfold 后台，管理员用）

## 当前状态

### 已完成

- [x] DRF 已配置（`covibe_server/settings/rest_framework.py`）
- [x] SimpleJWT 已配置（`covibe_server/settings/simplejwt.py`）
- [x] `covibe_server/api_urls.py` — 路由注册（orders + wechat payments）

### 实际 API 端点

以下是当前代码中**实际注册**的 API 路由：

```
# 订单
POST   /covibe_api/v1/orders/                             — 创建通用订单
GET    /covibe_api/v1/orders/                             — 订单列表
GET    /covibe_api/v1/orders/{uuid}/                      — 订单详情
POST   /covibe_api/v1/orders/{uuid}/pay/wechat/prepay/    — 微信预支付
POST   /covibe_api/v1/orders/{uuid}/check_payment/        — 查单
POST   /covibe_api/v1/orders/{uuid}/refunds/              — 退款（501 未实现）

# 微信支付回调
POST   /covibe_api/v1/payments/wechat/pay/notify/         — 微信支付回调
POST   /covibe_api/v1/payments/wechat/refund/notify/      — 微信退款回调
```

### 未实现（待开发）

以下 API 在文档中已设计但**尚未实现**：

```
# 认证
POST /covibe_api/v1/auth/oidc/login      — OIDC 第三方登录
POST /covibe_api/v1/auth/token/refresh   — 刷新 JWT
GET  /covibe_api/v1/users/me             — 当前用户信息

# Workspace
GET    /covibe_api/v1/workspaces           — 获取用户 workspace 列表
POST   /covibe_api/v1/workspaces           — 创建 workspace
PATCH  /covibe_api/v1/workspaces/:uuid     — 更新 workspace
DELETE /covibe_api/v1/workspaces/:uuid     — 删除 workspace

# Machine
POST /covibe_api/v1/machines/register          — 注册本机
GET  /covibe_api/v1/machines                   — 机器列表
GET  /covibe_api/v1/machines/:uuid/directories — 列出目录

# 会员
GET  /covibe_api/v1/tiers              — 会员等级列表
POST /covibe_api/v1/orders/membership  — 创建会员购买订单

# Session
GET /covibe_api/v1/sessions/count — Session 配额查询
```

### 认证实现说明

当前项目**尚未实现** auth API endpoints：
- 没有 `account/views.py`（不存在）
- 没有 `account/urls.py`（不存在）
- 没有 OIDC 登录视图
- 没有用户信息接口

**UserSerializer** 已定义（`account/serializers.py`），包含 `tier` 字段（返回 dict: name + expired_at）。

### 微信支付 API

`wechat/views.py` 实现了支付回调处理：

```python
POST /covibe_api/v1/payments/wechat/pay/notify/     → WechatPayNotifyView
POST /covibe_api/v1/payments/wechat/refund/notify/  → WechatRefundNotifyView
```

路由通过 `covibe_server/api_urls.py` → `wechat/urls.py` 注册。

### URL 路由配置

```python
# covibe_server/urls.py 实际结构
urlpatterns = [
    path('healthcheck/', healthcheck, name='healthcheck'),
    path('admin/', admin.site.urls),
    path('covibe_api/v1/', include(api_urlpatterns)),  # ViewSets + wechat callbacks
]
```

### Happy Server 集成

两款服务共享 PostgreSQL 数据库：
- covibe-server 的 Django User 通过 SocialLogin（provider + sub）与 happy-server 的 Account 关联
- **未实现** JWT 共享签发（需要统一 `HANDY_MASTER_SECRET`）
- **未实现** session 配额拦截

## 开发顺序（剩余）

1. [ ] 认证 API（OIDC 登录 + JWT）
2. [ ] 会员 API（tiers, membership orders）
3. [ ] Workspace API
4. [ ] Machine API
5. [ ] Happy Server JWT 集成

## 测试

```bash
# 启动开发服务器
uv run python manage.py runserver
```
