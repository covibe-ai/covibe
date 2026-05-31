# Covibe Server — Cursor Task: Project Setup & Core Models

## 背景

Covibe 是一个 Vibecoding 远程控制 SaaS 平台。后端基于 Django + DRF + Celery，管理用户、工作区、会员、支付等业务逻辑，与 Happy Server（Node.js）共享 PostgreSQL。

## 仓库

- 源码: `github.com/covibe-ai/covibe-server`
- 本地: `workspace/covibe-ai/covibe/covibe-server`
- 基于模板: `github.com/wtlyu/django-startup-kit`（已 fork 并重命名）

## 已完成

- [x] 从 `django-startup-kit` fork，项目包名 `covibe_server`
- [x] 目录结构 `covibe_server/`, `account/`, `order/`, `system/`, `wechat/`, `asset/`, `deploy/`
- [x] Django 配置：Unfold admin、DRF、Celery、Constance、SimpleJWT 已就绪
- [x] `covibe_server/models.py` 基础 BaseModel（ShortUUID 主键 + 时间戳 + History）
- [x] 模板中已有的 account/（WeixinUser）、order/（Order + OrderItem）、wechat/（微信支付）、system/（日志）

## 待完成的任务（按顺序）

### 任务 1：自定义 Account 模型 + OIDC 认证

目前 Django 默认用 `auth.User`。需要：

1. 创建自定义 User 模型替换 `auth.User`（`AUTH_USER_MODEL = 'account.User'`）
2. User 模型包含：
   - 基础字段：email, nickname, avatar
   - OIDC 字段：`oidc_sub`（唯一）、`oidc_issuer`、`oidc_provider`
   - 会员字段：`tier`（关联会员等级）、`vip_expires_at`
   - 自定义字段：`max_sessions_override`（覆盖等级默认值，null 表示用等级的）、`max_workspaces_override`
3. 自定义 UserAdmin（Unfold 主题）
4. OIDC 认证后端（`mozilla-django-oidc` 或自实现）：
   - 用户通过第三方 OIDC 登录后，根据 `sub` 查找或创建 User
   - 根用户 `root` 密码 `rootroot`，通过 `python manage.py createsuperuser` 创建
5. 迁移 `account_weixinuser` 外键指向新的 User 模型

**文件在**：`account/models.py`, `account/admin.py`, `account/auth_backends.py`

### 任务 2：会员等级（Tier）模型

创建 `member` app（新模块），包含：

1. `MemberTier` 模型：
   - `name`（名称: 免费版/专业版/企业版）
   - `price_per_month_minor`（月费，单位分）
   - `max_sessions`（最大在线 session 数）
   - `max_workspaces`（最大 workspace 数）
   - `max_idle_minutes`（闲置超时分钟）
   - `sort_order`（排序）

2. Constance 默认配置（在 `covibe_server/settings/constance.py` 追加）：
   - 默认三个等级的配额参数
   - 管理员可在 Unfold 后台实时修改

3. `Subscription` 模型：
   - `user` → User
   - `tier` → MemberTier
   - `started_at`, `expired_at`
   - `paid_amount_minor`（已付金额，累计）
   - `is_active` 属性（`now()` 在 `started_at` ~ `expired_at` 之间）

4. 履约逻辑：购买会员 → 创建/延长 Subscription，折抵未过期天数

**新文件在**：`member/models.py`, `member/admin.py`, `member/apps.py`

### 任务 3：订单折抵计算

在 `member/models.py` 或 `order/models.py` 中实现：

1. 用户购买会员时，检查当前是否有未过期会员
2. 如果有，计算剩余天数对应的金额：
   ```
   剩余天数 = (existing.expired_at - now()).days
   折抵金额 = 剩余天数 × (existing.tier.price_per_month_minor / 30)
   ```
3. 新订单金额 = max(0, 新会员价格 - 折抵金额)
4. 支付成功后，履约函数：
   - 无旧会员 → 创建新 Subscription
   - 有旧会员 → 延长 expired_at（取消旧的，创建新的覆盖时间线）

### 任务 4：微信支付

wechat/ 模块已从模板继承。需要：

1. 确保 `wechat/models.py`, `wechat/api.py`, `wechat/views.py` 能正常工作
2. Constance 配置中填好微信支付参数（模板已有，字段齐全）
3. 实现 `wechat.api.create_native_pay(order)` 生成二维码支付链接
4. 实现支付回调 webhook：`/api/v1/wechat/pay/notify`
5. 关联 `Order.make_paid()` 和 `member` 的履约逻辑

## 开发要求

- **Unfold 主题**所有 admin 页面使用 Unfold（已配置）
- **DRF** 所有 API 用 ViewSet + Serializer
- **ShortUUID** 主键全部用 ShortUUIDField
- **History** 关键模型加 `HistoricalRecords`
- **Celery** 异步任务用 Celery（已配置）

## API 设计

```
POST /api/v1/auth/oidc/login   — OIDC 登录
GET  /api/v1/users/me           — 当前用户信息（含 tier 和配额）
GET  /api/v1/tiers              — 会员等级列表
POST /api/v1/orders/membership  — 创建会员购买订单
POST /api/v1/wechat/pay/notify  — 微信支付回调

/admin/* — Unfold 后台管理（管理员用）
```

## HAPPY 集成

covibe-server 和 happy-server 共享同一个 PostgreSQL 数据库：
- `happy-server` 的表（accounts, sessions, session_messages, machines）直接可用
- covibe-server 的 Django User 通过 `oidc_sub` 与 happy-server 的 Account 关联
- 配额拦截：创建 session 前检查 `User.max_sessions`

## 快速启动

```bash
cd covibe-server
uv sync
uv run python manage.py migrate
uv run python manage.py createsuperuser --username root
uv run python manage.py runserver
```
