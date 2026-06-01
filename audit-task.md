你是一个代码审计专家。需要扫描整个 Covibe 项目，检查各模块间的依赖关系和文档与代码的一致性。

## 仓库结构

```
/Users/wtlyu.ai/workspace/covibe-ai/covibe/     ← 总入口
├── covibe-server/     ← Django 后端 (Python 3.11 + Django 5.2)
├── covibe-desktop/    ← 桌面端 (Electrobun + Vite + React)
├── covibe-dashboard/  ← Web 前端 (Vite + React)
├── covibe-happy/      ← Happy monorepo fork (不改)
├── cursor-tasks/      ← 文档目录
│   ├── 01-covibe-server-setup.md
│   ├── 02-covibe-desktop-init.md
│   ├── 03-covibe-server-api.md
│   └── api-docs.md
└── deploy/            ← Docker Compose + Nginx
```

## 审计要求

### 1. API 路径一致性
检查以下文件中的所有 API 路径，确保：
- Happy 原生 API 使用 `/v1/`、`/v2/`、`/v3/` 前缀（NOT `/happy-api/` NOT `/api/v1/`）
- Covibe 新 API 使用 `/covibe_api/v1/` 前缀
- 列出所有路径不合规的文件和具体行

### 2. 模型/字段一致性
检查 `covibe-server/account/models.py` 和 `covibe-server/member/models.py` 的实际字段，然后对照：
- `cursor-tasks/01-covibe-server-setup.md` 中描述的模型字段
- `cursor-tasks/api-docs.md` 中描述的 API 响应字段
- admin.py 中引用的字段
- serializers.py 中引用的字段
- 列出所有不存在字段或已删除字段的引用

### 3. 模块关系图
梳理以下依赖关系：
- account/models.py 被哪些文件 import？
- member/models.py 被哪些文件 import？
- order/models.py 被哪些文件 import？
- 所有 API URL 路由和对应的 View
- 所有 admin 注册的 Model

### 4. 文档与代码差异
逐段对比描述，列出每一个不匹配：

### 5. 种子数据
- scripts/seed_data.py 是否兼容当前模型？
- 管理员用户创建是否符合新模型（email required, SocialLogin 替代 OIDC 字段）？

## 输出要求
- 每个问题标注 文件:行号
- 严重程度：HIGH/MEDIUM/LOW
- 修复建议
- 如果是文档问题，给出修正后的文本
- 如果是代码问题，给出 patch
