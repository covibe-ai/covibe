# Covibe Server — Cursor Task: REST API Layer

## 背景

covibe-server 需要暴露 RESTful API 供 covibe-web（前端）和 covibe-app（手机）调用。  
部分请求需要转发到 Happy Server，部分由 Django 直接处理。

## API 前缀

- **业务 API**: `/covibe_api/v1/*`（由 covibe-server 处理）  
- **Happy API 代理**: `/v1/*`（由反代/covibe-server 转发到 happy-server，happy 原生使用 `/v1/` 前缀）  
- **Admin**: `/admin/*`（Unfold 后台，管理员用）

## 已完成

- [x] DRF 已配置（`covibe_server/settings/rest_framework.py`）
- [x] SimpleJWT 已配置（`covibe_server/settings/simplejwt.py`）
- [x] `covibe_server/api_urls.py` 已初始化（nested routers）

## 待完成

### 任务 1：认证 API

**OIDC 登录流程：**

```
客户端 → POST /covibe_api/v1/auth/oidc/login
           { id_token, provider: "google"|"github"|... }
         → 验证 OIDC token（验证 iss, aud, sub）
         → 查找或创建 SocialLogin（关联 User）
         → 返回 JWT token（SimpleJWT）

客户端 → POST /covibe_api/v1/auth/token/refresh
           { refresh }
         → 刷新 JWT

客户端 → GET /covibe_api/v1/users/me
         → 返回用户信息 + tier 等级 + 当前可用配额
```

**实现：**
- `account/serializers.py` — UserSerializer, LoginSerializer
- `account/views.py` — OIDCLoginView, UserMeView
- `account/urls.py` — 注册路由
- DRF 权限：`IsAuthenticated` 用于需要登录的接口

### 任务 2：会员与配额 API

```
GET  /covibe_api/v1/tiers              → 返回所有会员等级和价格
POST /covibe_api/v1/orders/membership  → 创建会员购买订单
  Body: { tier_id }
  → 计算折抵金额 → 创建 Order + OrderItem
  → 返回 Order（含 amount_minor）
```

**配额检查（关键逻辑）：**  
happy-server 创建 session 之前，covibe-server 需要拦截检查：

```
检查流程：
  1. 查 User 的会员等级（或自定义覆盖）
  2. 查当前活跃 session 数（通过 happy-server 的 Session 表，count WHERE active=true）
  3. 如果 >= max_sessions → 拒绝
  4. 如果 < max_sessions → 放行
```

### 任务 3：微信支付 API

wechat/ 模块已有模板代码，需要确认：

```
POST /covibe_api/v1/wechat/pay/native    → 创建 Native 支付二维码
  Body: { order_id }
  → 调用微信支付 API → 返回 code_url（二维码内容）

POST /covibe_api/v1/wechat/pay/notify    → 微信支付回调（反向代理暴露到公网）
  → 验签 → 更新订单 → 触发履约
```

### 任务 4：Happy Server 集成

**共享数据库：**  
covibe-server 和 happy-server 共用同一个 PostgreSQL。  
Django 可以直接操作 happy-server 的表（通过 `db_table` 映射或用 raw SQL）：

```python
# 代理：通过 Django 的数据库连接查询 happy-server 的 Session 表
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute("SELECT count(*) FROM \"Session\" WHERE \"accountId\" = %s AND active = true", [user.oidc_sub or user.email])
    active_sessions = cursor.fetchone()[0]
```

**JWT 兼容：**  
covibe-server 签发的 JWT 需要被 happy-server 的 Bearer token 验证接受。  
有两种方案：

| 方案 | 做法 |
|------|------|
| **A: 共享密钥** | covibe-server 跟 happy-server 共用一个 HMAC secret |
| **B: 双验证** | happy-server 加 preHandler 支持两种 token |

**推荐方案 A** — covibe-server 用跟 happy-server 同样的 `HANDY_MASTER_SECRET` 签发 token，
确保格式兼容。

### 任务 5：URL 路由配置

```python
# covibe_server/urls.py 最终结构
urlpatterns = [
    path('admin/', admin.site.urls),
    path('covibe_api/v1/auth/', include('account.urls')),
    path('covibe_api/v1/', include('api_urls')),  # ViewSets
    path('v1/', include('happy_proxy.urls')),  # 转发到 happy-server（happy 使用原生 /v1/ 前缀）
]
```

## 开发顺序

1. 先实现 OIDC 登录 + JWT（任务 1）
2. 再实现会员 API（任务 2）
3. 再集成微信支付（任务 3）
4. 最后集成 Happy Server（任务 4）

## 测试

```bash
# 启动开发服务器
uv run python manage.py runserver

# 测试 OIDC 登录
curl -X POST http://localhost:8000/covibe_api/v1/auth/oidc/login \
  -H "Content-Type: application/json" \
  -d '{"id_token": "...", "provider": "google"}'

# 测试需要认证的接口（带上 JWT）
curl http://localhost:8000/covibe_api/v1/users/me \
  -H "Authorization: Bearer ***"
```
