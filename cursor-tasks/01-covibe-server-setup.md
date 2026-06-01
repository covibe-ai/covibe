# Covibe Server — Cursor Task: Project Setup & Core Models

## 背景

Covibe 是一个 Vibecoding 远程控制 SaaS 平台。后端基于 Django + DRF + Celery，管理用户、工作区、会员、支付等业务逻辑，与 Happy Server（Node.js）共享 PostgreSQL。

## 仓库

- 源码: `github.com/covibe-ai/covibe-server`
- 本地: `workspace/covibe-ai/covibe/covibe-server`
- 基于模板: `github.com/wtlyu/django-startup-kit`（已 fork 并重命名）

## 已完成

- [x] 从 `django-startup-kit` fork，项目包名 `covibe_server`
- [x] 目录结构 `covibe_server/`, `account/`, `member/`, `order/`, `system/`, `wechat/`, `asset/`, `deploy/`
- [x] Django 配置：Unfold admin、DRF、Celery、Constance、SimpleJWT 已就绪
- [x] `covibe_server/models.py` 基础 BaseModel（ShortUUID 主键 + 时间戳 + History）
- [x] 模板中已有的 account/（User + SocialLogin + WeixinUser）、order/（Order + OrderItem）、wechat/（微信支付）、system/（日志）
- [x] member/ 模块（MemberTier + Subscription 含 extend_or_create 购买逻辑）

## 已实现的核心模型

### 1. 自定义 Account 模型 + OIDC 认证

Django 已使用自定义 User 模型（`AUTH_USER_MODEL = 'account.User'`）：

**User 模型** (`account/models.py`):
- 基础字段：email (unique, USERNAME_FIELD), phone, nickname, avatar, phone_verified
- username = None（禁用 Django 默认 username）
- 社交登录信息在 SocialLogin 表（一对多，一个用户可以有多个社交账号）
- 会员信息在 Subscription 模型（通过 `extend_or_create()` 管理），不在 User 上
- 自定义覆盖字段：`max_sessions_override`、`max_workspaces_override`（null = 用会员等级默认值）
- 属性：`active_subscription`、`effective_max_sessions`、`effective_max_workspaces`、`effective_idle_minutes`

**SocialLogin 模型**: `provider` + `sub`（唯一联合）、`username`、`avatar_url`

**WeixinUser 模型**: 微信 OpenID（微信支付 JSAPI 预下单必需）

**管理员** (`account/admin.py`): UserAdmin 使用 Unfold 主题，含 SocialLoginInline + WeixinUserInline

- 根用户通过 `scripts/seed_data.py` 创建（email: root@covibe.ai, password: rootroot）

### 2. 会员等级（Tier）模型

`member/models.py`:

**MemberTier** 模型：
- `name`, `price_per_month_minor`, `max_sessions`, `max_workspaces`, `max_idle_minutes`, `sort_order`, `is_default`

**Subscription** 模型：
- `user` → User, `tier` → MemberTier
- `started_at`, `expired_at`, `paid_amount_minor`
- `is_active` 属性（now 在 started_at ~ expired_at 之间）
- `extend_or_create(user, tier, days, amount_minor)` 类方法处理购买逻辑

### 3. 订单折抵计算

在 `member/models.py` 的 `extend_or_create()` 中实现：
- 同级续购：不折抵，延长 expired_at
- 升级：终止旧会员，折抵剩余天数按旧等级月费/30 计算
- 无旧会员：新建

### 4. 微信支付

`wechat/` 模块完整：
- `wechat/api.py`: WechatPayAPIV3Mixin（JSAPI/Native 统一下单、验签、解密通知、退款）
- `wechat/views.py`: WechatPayNotifyView + WechatRefundNotifyView（回调处理）
- `wechat/urls.py`: `pay/notify/` + `refund/notify/`
- `wechat/models.py`: WechatPayTransaction（支付流水记录）

## 开发要求

- **Unfold 主题**所有 admin 页面使用 Unfold（已配置）
- **DRF** 所有 API 用 ViewSet + Serializer
- **ShortUUID** 主键全部用 ShortUUIDField
- **History** 关键模型加 `HistoricalRecords`
- **Celery** 异步任务用 Celery（已配置）

## API 前缀

```
/covibe_api/v1/*  — 业务 API（DRF）
/admin/*          — Unfold 后台管理
/healthcheck/     — 健康检查
```

## 当前实际 API 路由

```
POST   /covibe_api/v1/orders/                   — 创建通用订单
GET    /covibe_api/v1/orders/                   — 订单列表
GET    /covibe_api/v1/orders/:uuid/             — 订单详情
POST   /covibe_api/v1/orders/:uuid/pay/wechat/prepay/   — 微信预支付
POST   /covibe_api/v1/orders/:uuid/check_payment/       — 查单
POST   /covibe_api/v1/orders/:uuid/refunds/             — 退款（未实现）
POST   /covibe_api/v1/payments/wechat/pay/notify/       — 微信支付回调
POST   /covibe_api/v1/payments/wechat/refund/notify/    — 微信退款回调

/admin/*      — Unfold 后台管理（管理员用）
/healthcheck/ — 健康检查
```

## HAPPY 集成

covibe-server 和 happy-server 共享同一个 PostgreSQL 数据库：
- `happy-server` 的表（accounts, sessions, session_messages, machines）直接可用
- covibe-server 的 Django User 通过 SocialLogin（provider + sub）与 happy-server 的 Account 关联
- 配额拦截：创建 session 前检查 `User.effective_max_sessions`（通过自定义属性）

## 快速启动

```bash
cd covibe-server
uv sync
uv run python manage.py migrate
uv run python manage.py runscript scripts/seed_data
uv run python manage.py runserver
```
