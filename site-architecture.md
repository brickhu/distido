# distito 站点架构与项目结构

> v0.2 · 2026-07-24
> 关联文档: [injection-design.md](./injection-design.md)

---

## 1. 域名方案

```
distito.com          公开站点（静态 HTML + widget.js）
  ├── /               官网首页 / 关于 / 定价等
  ├── /:baseSlug       Base 首页
  ├── /:baseSlug/:articleSlug  文章页
  ├── /explore         发现页
  ├── /sitemap.xml     站点地图
  └── /feed.xml        RSS

app.distito.com      管理后台（SolidJS SPA，需登录）
  ├── /dashboard       总览
  ├── /bases           Base 列表与管理
  ├── /bases/:slug     Base 详情
  ├── /articles        文章管理
  └── /settings        个人设置

api.distito.com      后端 API（Supabase Edge Functions）
  ├── /api/widget/*    公开 widget 数据
  ├── /api/bases/*     业务 API
  ├── /api/articles/*
  ├── /api/auth/*      认证
  └── /admin/*         管理接口
```

---

## 2. 项目结构

```
distito/
├── builder/               ← 静态站点构建器
│   ├── builder/
│   │   ├── __init__.py
│   │   ├── site_builder.py     # 主构建流程
│   │   ├── article_renderer.py # 文章页渲染
│   │   ├── base_renderer.py    # Base 首页渲染
│   │   ├── homepage_renderer.py# 官网页面渲染（首页/关于等）
│   │   ├── feed_generator.py   # RSS / sitemap 生成
│   │   └── templates/          # Jinja2 模板
│   │       ├── base.html
│   │       ├── article.html
│   │       ├── base_index.html
│   │       ├── homepage.html
│   │       └── ...
│   ├── tests/
│   │   └── test_builder.py
│   ├── requirements.txt
│   ├── pyproject.toml
│   └── README.md
│
├── webapp/                ← 管理后台 SPA (app.distito.com)
│   ├── src/
│   │   ├── components/        # 函数式组件
│   │   ├── pages/             # 路由页面
│   │   ├── api/               # HTTP 调用
│   │   ├── stores/            # SolidJS stores
│   │   ├── routes.ts          # 路由定义
│   │   ├── App.tsx
│   │   └── index.tsx
│   ├── public/
│   ├── tests/
│   │   └── ...
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── package.json
│   └── README.md
│
├── api/                ← 后端 API（Supabase Edge Functions）
│   ├── functions/
│   │   ├── __init__.py
│   │   ├── articles.ts       # 文章 CRUD
│   │   ├── bases.ts          # Base CRUD
│   │   ├── widget.ts         # widget 数据（likes/comments/stats）
│   │   ├── auth.ts           # GitHub OAuth + JWT
│   │   └── resolve.ts        # URL 解析
│   ├── migrations/           # Supabase CLI 数据库迁移
│   ├── tests/
│   ├── supabase.toml
│   ├── seed.sql
│
├── website/               ← 官网 SPA (distito.com 知识聚合)
│   ├── src/
│   │   ├── components/
│   │   ├── sections/
│   │   ├── pages/
│   │   ├── App.tsx
│   │   └── index.tsx
│   ├── public/
│   │   └── assets/
│   ├── tests/
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── package.json
│   └── README.md
│
├── mcp-server/             ← MCP 协议处理（一行安装）
│   ├── distito_mcp/
│   │   ├── __init__.py
│   │   ├── cli.py              # 入口：distito-mcp ↔ Kun/Claude 通信
│   │   ├── setup.py            # 自动配置检测与安装
│   │   ├── handler.py          # => 指令处理主逻辑
│   │   ├── fetcher.py          # web fetch + 内容提取
│   │   ├── resolver.py         # URL 解析 + distito+json 识别
│   │   ├── validator.py        # 内容质量校验
│   │   ├── injector.py         # 参考资料注入格式化
│   │   └── templates.py        # 注入模板 / 反馈格式
│   ├── tests/
│   ├── requirements.txt
│   ├── pyproject.toml
│   └── README.md
│
├── database/               ← 数据库 DDL 与初始化脚本
│   ├── schema.sql             # 完整 DDL（建表语句 + 索引 + 约束）
│   ├── seed.sql               # 开发用种子数据
│   └── README.md              # 数据库设计说明
│
├── ops/                   ← 运维配置
│   ├── wrangler.distito.toml     # distito.com Cloudflare Pages 配置
│   ├── wrangler.app.toml         # app.distito.com Cloudflare Pages 配置
│   └── reserved-slugs.schema.json
│
├── reserved-slugs.json    ← 保留路径列表（不可用作 base-slug 的二级路径）
├── AGENT.md               ← 项目总览与约定
├── injection-design.md        ← Inject 功能设计文档
└── site-architecture.md   ← 本文档（站点架构与项目结构）
```

---

## 3. 各模块详述

### 3.1 builder/ — 静态站点构建器

| 属性 | 值 |
|------|-----|
| 语言 | Python 3.12+ |
| 核心依赖 | Jinja2、httpx |
| 输入 | 调 `api.distito.com` 获取公开数据 |
| 输出 | `dist/` 目录（完整静态网站） |
| 部署触发 | Cloudflare Pages Webhook（发布文章时调用） |
| 构建时间 | 30s-2min（全量重建） |

**职责**：
- 拉取所有公开文章 + Base 信息
- 渲染文章页 HTML（SEO 内容已嵌入）
- 渲染 Base 首页
- 生成 sitemap.xml + feed.xml
- 复制 widget.js 到 `dist/assets/widget.js`

**不负责**：
- 运行时请求处理（纯构建时）
- 用户认证
- 官网主页/发现页（由 website/ SPA 通过 API 动态加载）
- 动态数据（交给 widget.js 运行时加载）

### 3.2 webapp/ — 管理后台 SPA

| 属性 | 值 |
|------|-----|
| 语言 | TypeScript |
| 框架 | SolidJS |
| 样式 | StyleX |
| 构建 | Vite |
| 路由 | @solidjs/router |
| 测试 | Vitest |
| 部署 | Cloudflare Pages |
| 域名 | app.distito.com |

**职责**：
- 用户登录（GitHub OAuth）
- Base 创建 / 管理 / 设置
- 文章管理（列表 / 编辑 / 删除）
- 发布工作流（`=>` 蒸馏确认）
- 统计面板
- API Key 管理（未来）

### 3.3 api/ — Supabase Edge Functions

| 属性 | 值 |
|------|-----|
| 语言 | TypeScript（Deno 运行时） |
| 框架 | Supabase Edge Functions |
| 数据库 | PostgreSQL (Supabase) |
| 迁移 | Supabase CLI |
| 测试 | Deno test |
| 部署 | `supabase functions deploy`（含在 Supabase Free Tier 中） |
| 域名 | api.distito.com |

**职责**：
- 文章 CRUD
- Base CRUD + 成员管理
- 用户认证（通过 Supabase Auth 内置的 GitHub OAuth）
- Widget 数据 API（likes / injectes / comments）
- Inject 关系管理
- 搜索
- Webhook 触发构建通知

### 3.4 website/ — 官网 SPA

| 属性 | 值 |
|------|-----|
| 语言 | TypeScript |
| 框架 | SolidJS + StyleX |
| 构建 | Vite |
| 测试 | Vitest |
| 部署 | Cloudflare Pages（distito.com） |

**职责**：
- 知识聚合首页（热门知识、推荐、趋势）
- 发现页（按标签/Base/作者浏览）
- 关于页、定价页等
- 所有数据通过 API 动态获取，不依赖 builder 构建

**路由冲突防护**：所有 SPA 路由路径必须登记在 `reserved-slugs.json` 中，避免与 base-slug 冲突。

### 3.5 mcp-server/ — MCP 协议处理器

| 属性 | 值 |
|------|-----|
| 语言 | Python 3.12+ |
| 框架 | mcp SDK / httpx |
| 分发 | PyPI（`pip install distito-mcp`） |
| 安装 | **一行命令自动安装+配置**：`pip install distito-mcp && distito-mcp setup` |
| setup 行为 | 自动检测当前平台（Kun / Claude Desktop），写入 MCP 配置文件 |
| 运行 | 用户本地，无需服务端 |

**职责**：
- 监听 `=>` 指令，解析 URL 和自然语言指令
- web fetch 获取内容、解析 `distito+json` 结构化元数据
- 内容质量校验
- 组装参考资料注入格式，注入到 AI 对话上下文
- 执行功能指令（翻译/融合）
- 调用 `api.distito.com` 发布文章

**安装方式**：
```bash
# 方式 1：一键安装+配置（推荐）
pip install distito-mcp && distito-mcp setup

# 方式 2：仅安装，手动配置
pip install distito-mcp
# 然后在 MCP 配置文件中添加：
# { "distito": { "command": "distito-mcp", "args": [] } }
```

**`distito-mcp setup` 自动完成**：
1. 检测操作系统和已安装的 AI 客户端
2. 定位 MCP 配置文件（Kun / Claude Desktop）
3. 写入 distito 条目
4. 验证运行正常

---

## 4. 公开站点页面规范

### 4.1 HTML 结构

每个公开文章页由 builder 生成，包含 SEO 内容 + JSON-LD 元数据 + widget.js：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <title>CI/CD 核心原理 | distito</title>
  <meta name="description" content="CI/CD 的核心是自动化...">

  <!-- Open Graph -->
  <meta property="og:title" content="CI/CD 核心原理">
  <meta property="og:type" content="article">
  <meta property="og:url" content="https://distito.com/tech/ci-cd-core-principles">
  <meta property="og:description" content="CI/CD 的核心是自动化...">

  <!-- distito JSON-LD 元数据 -->
  <!-- 供 => 注入识别、widget.js 读取、开放标准引用 -->
  <script type="application/distito+json">
  {
    "@context": "https://distito.com/rsd/1.0",
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "title": "CI/CD 核心原理",
    "author": {
      "slug": "alice",
      "display_name": "Alice"
    },
    "base": {
      "slug": "tech",
      "name": "技术博客"
    },
    "published_at": "2026-07-23T12:00:00Z",
    "inject_from": [],
    "tags": ["devops", "ci-cd"],
    "url": "https://distito.com/tech/ci-cd-core-principles"
  }
  </script>

  <link rel="canonical" href="https://distito.com/tech/ci-cd-core-principles">
  <script src="/assets/widget.js" defer></script>
</head>
<body>
  <article>
    <h1>CI/CD 核心原理</h1>
    <div class="content">…全文…</div>
  </article>

  <div id="distito-widget-root"></div>
</body>
</html>
```

### 4.2 widget.js

一个极小的 SolidJS 微应用（约 3-5KB gzipped），部署在 `distito.com/assets/widget.js`。

```
widget.js
├── 核心功能：
│   ├── 从 <script type="application/distito+json"> 读取 article_id / base_slug
│   ├── 调 GET api.distito.com/api/widget/:articleId/stats
│   │   → 返回 { likes, injections, comments_count }
│   ├── 调 GET api.distito.com/api/widget/:articleId/comments?limit=5
│   │   → 返回最近评论
│   ├── 渲染底部交互栏（like / inject / comment 按钮 + 计数）
│   └── 渲染全局浮动操作入口（floating action button）
│
├── 认证态：
│   ├── 未登录 → 按钮点击后弹出登录提示
│   └── 已登录 → 直接操作（like / comment / inject）
│
└── 构建输出：
    └── 单文件 widget.js → 上传到 Cloudflare Pages 的 /assets/widget.js
```

### 4.3 全局操作入口（FAB）

浮动操作按钮，右下角 fixed 定位：

```
                      [未登录态]
         ┌───────────────┐
         │  📎 引用这篇文章 │  → 复制引用 URL
         │  ★ 关注 Base    │     提示在 Kun 中使用
         │  ↗ 分享         │
         │  💬 写评论      │
         └───────────────┘
                ▲
                │
          ┌─────┴──────┐
          │  ✦          │
          └────────────┘

                      [已登录态]
         ┌───────────────┐
         │  📎 引用到对话  │  → 复制引用 URL
         │  ★ 已关注      │
         │  ↗ 分享        │
         │  💬 写评论      │  → 展开评论输入框
         └───────────────┘
                ▲
                │
          ┌─────┴──────┐
          │  ✦  N       │  ← 未读评论通知
          └────────────┘
```

---

## 5. 部署拓扑

```
                           Cloudflare
                          ┌──────────┐
        用户 ─── DNS ──▶ │  CDN     │
                          └────┬─────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
      distito.com        app.distito.com    api.distito.com
    ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
    │ Cloudflare   │   │ Cloudflare   │   │ Supabase      │
    │ Pages        │   │ Pages        │   │ (Edge Funcs) │
    │              │   │              │   │              │
    │ builder/     │   │ webapp/      │   │ fastapi/     │
    │ 构建输出      │   │ Vite 构建    │   │ uvicorn 运行 │
    │ dist/        │   │ dist/        │   │              │
    └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
           │                  │                  │
           │                  │                  ▼
           │                  │           ┌──────────────┐
           │                  │           │ Supabase     │
           │                  │           │ PostgreSQL   │
           │                  │           └──────────────┘
           │                  │
           │    Webhook       │
           └──────────────────┘
               发布文章时
               POST → Cloudflare Pages
               触发构建
```

---

## 6. 构建 & 部署流水线

### 6.1 各项目构建命令

```bash
# builder — 生成静态站点到 dist/
cd builder && python -m builder.site_builder

# webapp — 构建 SPA
cd webapp && vite build

# website — 构建营销页面
cd website && vite build

# api — Supabase Edge Functions 部署
cd api && supabase functions deploy --no-verify-jwt

# 未来迁移到其他平台：Edge Functions 标准 TypeScript，迁移成本低
```

### 6.2 Cloudflare Pages 配置

```toml
# distito.com — Cloudflare Pages 项目
# build command:  python builder/site_builder.py
# build output:   builder/dist/
# webhook:        POST on article publish

# app.distito.com — Cloudflare Pages 项目
# build command:  cd webapp && vite build
# build output:   webapp/dist/
```

### 6.3 构建触发流程

```
用户发布文章（POST /api/articles）
  │
  ├── 1. Edge Functions 保存文章到数据库
  │      ↓
  │     Supabase PostgreSQL（托管服务，无需部署）
  │
  ├── 2. Edge Functions 调用 distito Cloudflare Pages Webhook
  │        POST https://api.cloudflare.com/.../webhook
  │
  └── 3. Cloudflare Pages 执行构建
           git pull → python build_site.py → deploy dist/
```

### 6.4 完整部署拓扑

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│ distito.com │     │ app.distito  │     │  api.distito    │
│ Cloudflare  │     │ Cloudflare   │     │  Supabase       │
│ Pages       │     │ Pages        │     │  Edge Functions │
│ (静态文件)   │     │ (SolidJS SPA)│     │                 │
└─────────────┘     └──────────────┘     └────────┬────────┘
                                                   │
                                                   ▼
                                          ┌─────────────────┐
                                          │  Supabase Cloud  │
                                          │  PostgreSQL 16   │
                                          │  (托管，零运维)   │
                                          └─────────────────┘
                         ┌──────────────┐
                         │  用户本地     │
                         │  mcp-server   │
                         │  (一行安装)    │
                         └──────────────┘
```

---

## 7. 技术栈总览

| 模块 | 语言 | 框架 / 工具 | 测试 | 部署 |
|------|------|------------|------|------|
| builder | Python | Jinja2, httpx | pytest | Cloudflare Pages (构建时) |
| webapp | TypeScript | SolidJS, StyleX, Vite | Vitest | Cloudflare Pages |
| website | TypeScript | SolidJS, StyleX, Vite | Vitest | Cloudflare Pages |
| api | TypeScript (Deno) | Supabase Edge Functions | Deno test | Supabase（Free Tier，零额外成本） |
| mcp-server | Python | mcp SDK, httpx | pytest | PyPI（`pip install distito-mcp && distito-mcp setup`，一行完成） |
| shared | Python | Pydantic | — | 内嵌（非独立部署） |
| 数据库 | SQL | PostgreSQL 16 (Supabase) | — | **托管服务，无需部署**（在线创建项目即可） |

---

## 8. 文件间引用关系

```
AGENT.md  ─── 项目总览
  │
  ├──→ injection-design.md     ─── Inject 功能的数据模型 + API + 流程 + 提示词模板
  │
  └──→ site-architecture.md ─── 本文档：域名、项目结构、部署
                                └── 引用 injection-design.md 中的模型定义


builder/        ───→ 调 api.distito.com 获取数据
webapp/         ───→ 调 api.distito.com 进行管理操作
mcp-server/     ───→ 用户本地运行，调 api.distito.com 发布文章
fastapi/        ───→ 提供 api.distito.com 接口
website/        ───→ 调 api.distito.com 展示知识聚合
```

---

## 9. 保留路径与命名冲突防护

### 9.1 背景

`distito.com/<base-slug>/<article-slug>` 格式中，`<base-slug>` 是一级路径。
某些路径必须预留，不可被用户注册为 base-slug。

### 9.2 保留路径列表

| 路径 | 用途 |
|------|------|
| `distito.com/about` | 关于页面 |
| `distito.com/explore` | 发现页 |
| `distito.com/pricing` | 定价页（预留） |
| `distito.com/blog` | 官方博客（预留） |
| `distito.com/help` | 帮助中心（预留） |
| `distito.com/docs` | 文档（预留） |
| `distito.com/terms` | 服务条款 |
| `distito.com/privacy` | 隐私政策 |
| `distito.com/status` | 服务状态 |
| `distito.com/sitemap` | 站点地图 |
| `distito.com/feed` | RSS Feed |
| `distito.com/app` | 重定向到 app.distito.com（预留） |
| `distito.com/api` | 重定向到 api.distito.com（预留） |
| `distito.com/admin` | 管理入口（预留） |
| `distito.com/_*` | `_` 下划线前缀全部保留（内部路由） |
| `distito.com/*` | **≤ 3 字符的路径全部保留（如 `/cn`、`/js`、`/go`）** |

### 9.3 Base slug 长度限制

| 规则 | 值 | 理由 |
|------|-----|------|
| 最小长度 | **4 字符** | 3 字符以内路径全部保留，用于 SPA 路由或未来扩展 |
| 最大长度 | 32 字符 | 合理范围 |
| 字段名 | 英文小写字母、数字、连字符 | SEO 友好 |

### 9.4 校验规则

创建 Base 时，`slug` 字段必须通过以下校验：

```python
# fastapi/app/services/base_service.py

RESERVED = json.load(open("reserved-slugs.json"))
reserved_slugs = {item["slug"] for item in RESERVED["reserved"]}
reserved_prefixes = tuple(RESERVED["patterns"]["prefixes"])

def validate_base_slug(slug: str):
    if slug in reserved_slugs:
        raise ValueError(f"'{slug}' 是保留路径，不可用作 Base")
    if slug.startswith(reserved_prefixes):
        raise ValueError(f"'{slug[0]}' 前缀保留，不可用作 Base")
    if len(slug) < 4:
        raise ValueError("Base slug 至少 4 个字符（3 字符以内保留）")
    if len(slug) > 32:
        raise ValueError("Base slug 最多 32 个字符")
    # 字母数字 + 连字符
    if not re.match(r'^[a-z0-9]([a-z0-9-]*[a-z0-9])?$', slug):
        raise ValueError("Base slug 只允许小写字母、数字和连字符")
```

`reserved-slugs.json` 同时被 builder 和 fastapi 引用，确保两端校验一致。

**简化理解**：`distito.com` 上，3 字符以内的路径归官网 SPA，4 字符以上开放为 base-slug。这意味着 `reserved-slugs.json` 只需登记 > 3 字符的保留路径（如 `about`、`explore`），而 `cn`、`en`、`zh`、`js`、`go` 等 2-3 字符路径天然受长度限制保护。

---

## 10. 运维配置

### 10.1 ops/ 目录

```
ops/
├── wrangler.distito.toml     # distito.com — Cloudflare Pages 配置
├── wrangler.app.toml         # app.distito.com — Cloudflare Pages 配置
└── reserved-slugs.schema.json
```

### 10.2 部署命令速查

```bash
# distito.com — 全量静态构建
cd builder && python -m builder.site_builder
# 输出到 builder/dist/ → Cloudflare Pages 自动部署

# app.distito.com — SPA 构建
cd webapp && npm run build
# 输出到 webapp/dist/ → Cloudflare Pages 自动部署

# api.distito.com — Supabase Edge Functions 部署
cd api && supabase functions deploy
```

### 10.3 环境变量

| 变量 | 用途 | 项目 |
|------|------|------|
| `API_BASE` | API 地址 | builder |
| `VITE_API_BASE` | API 地址 | webapp |
| `DATABASE_URL` | PostgreSQL 连接串 | api（Edge Functions） |
| `SUPABASE_URL` | Supabase 项目 URL | api |
| `SUPABASE_SERVICE_ROLE_KEY` | 服务端密钥（管理操作） | api |
| `CLOUDFLARE_WEBHOOK_URL` | Pages 构建触发 | api |

### 10.4 免费额度预算

| 资源 | 服务 | 每月免费额度 | MVP 预估用量 |
|------|------|------------|-------------|
| Cloudflare Pages 构建 | distito.com | 500 次 | ~300 次（日更10篇） |
| Cloudflare Pages 带宽 | distito.com | 1 GB | 远低于 |
| Cloudflare Pages 构建 | app.distito.com | 500 次 | ~30 次（代码变更时） |
| Cloudflare Workers | 预留（未来升级用） | 10 万请求/天 | 暂不用 |
| Supabase 数据库 | 托管 PostgreSQL | 500MB | ~50MB（10万篇文章） |
### 10.5 用户认证架构

全部基于 Supabase Auth 内置能力，无需自建认证服务：

```
用户（app.distito.com）
  │
  ├─ 点击「GitHub 登录」
  ├─ @supabase/supabase-js → 唤起 Supabase Auth
  ├─ GitHub OAuth 授权 → Supabase 返回 JWT
  └─ 后续请求携带 JWT（Authorization: Bearer xxx）
         │
         ▼
Edge Functions
  ├─ supabase-js getJWT() / getUser() 自动验证
  ├─ 无需自配 JWT_SECRET、GITHUB_CLIENT_*
  └─ RLS 策略在数据库层二次校验

公开端点（无需登录）：
  ├─ GET /api/resolve           — URL 解析
  ├─ GET /api/widget/:id/stats  — likes/injections/comments
  └─ GET /api/widget/:id/comments

需认证端点：
  ├─ POST /api/bases/*          — Base CRUD
  ├─ POST /api/articles/*       — 文章发布/编辑
  └─ POST /api/widget/:id/like  — 交互操作
```

| 组件 | OAuth 配置入口 |
|------|---------------|
| GitHub OAuth | Supabase Dashboard → Authentication → Providers → GitHub |
| JWT 验证 | Supabase 自动处理，Edge Functions 通过 `supabase-js` 获取用户 |
| RLS 策略 | `api/migrations/` 中的 SQL（`CREATE POLICY ...`） |
| 客户端 SDK | `webapp/` 用 `@supabase/supabase-js` 登录 |

### 10.6 数据库部署

| 属性 | 值 |
|------|-----|
| **服务** | Supabase（托管 PostgreSQL） |
| **版本** | PostgreSQL 16 |
| **套餐** | Free Tier（MVP 足够） |
| **容量** | 500MB（约 10 万篇文章） |
| **网络** | 默认 us-east-1，仅限 API 内网访问 |
| **连接** | `DATABASE_URL` 环境变量注入 fastapi 容器 |
| **认证** | 内置 GitHub OAuth + Row Level Security |
| **迁移** | Alembic（`fastapi/migrations/`） |

**选择 Supabase Edge Functions 的理由**：
- Supabase Free Tier 已包含 Edge Functions，零额外成本
- 数据库 + API + Auth 同一平台，统一管理
- Edge Functions 标准 TypeScript，未来迁移成本低

**数据库选 Supabase 的理由**：
- 托管 PostgreSQL，MVP 不应碰运维（备份、扩容、监控）
- Free Tier 500MB 够用一年以上
- 内置 GitHub OAuth + RLS，省下认证系统开发成本
- 标准 PostgreSQL，未来可 `pg_dump` → `pg_restore` 自建，无 vendor lock-in

---

## 11. 自定义域名路由

### 11.1 概念

自定义域名是一张**通用 URL 重写表**。每条规则定义：

> 自定义域名上的某个**挂载点** → 重写为 `distito.com` 的某个**路径前缀**

`rewrite_prefix` 可以是任意 distito.com 路径：一个 Base、一篇文章、一个用户页。

### 11.2 示例

一条内容 `shit`（位于 Base `tech`），可以通过多条规则被多个路径访问：

```
distito.com/tech/shit           ← 标准平台路径

distito.com/tech/<uuid>         ← 按 ID 访问

fei.me/tech/shit               ← mount:/tech → rewrite:tech → distito.com/tech/shit

fei.me/shit                    ← mount:/tech → rewrite:tech (同时)
                                suffix:/shit → distito.com/tech/shit

cn.fei.me/shit                 ← subdomain:cn → rewrite:tech
                                suffix:/shit → distito.com/tech/shit

fei.me/about                   ← mount:/about → rewrite:_pages/tech/about
                                → distito.com/_pages/tech/about
```

### 11.3 术语

| 概念 | 说明 |
|------|------|
| **自定义域名** | 用户自有域名，如 `fei.me`、`blog.zhang.xyz` |
| **挂载模式** | `subdomain`（子域名模式）或 `path`（路径前缀模式） |
| **mount_point** | 挂载点。子域名模式 = `cn`；路径模式 = `/cn` 或 `/` |
| **rewrite_prefix** | 替换为的 distito.com 路径前缀，如 `tech`、`user/fei`、`_pages/about` |

### 11.4 数据模型

```sql
CustomDomain
├── id                      UUID PRIMARY KEY
├── user_id → User          NOT NULL      -- 域名所有者
├── domain                  VARCHAR NOT NULL  -- fei.me
├── mount_type              VARCHAR NOT NULL DEFAULT 'path'  -- 'subdomain' | 'path'
├── mount_point             VARCHAR NOT NULL  -- 'cn', '/cn', '/'
├── rewrite_prefix          VARCHAR NOT NULL  -- 'tech', 'user/fei', '_pages/about'
├── dns_verified            BOOLEAN DEFAULT FALSE
├── verified_at             TIMESTAMPTZ?
├── created_at             TIMESTAMPTZ NOT NULL
│
UNIQUE(domain, mount_type, mount_point)
```

**关键**：`rewrite_prefix` 不限定为 Base slug。它可以是任何已有路径，如 `tech`（Base）、`_pages/tech/about`（系统页面）、`user/fei`（用户页）。

### 11.5 路由规则：Worker 逻辑

```
请求: cn.fei.me/shit  (host=cn.fei.me, path=/shit)
          │
          ▼
① 域名 key 提取：hostnameToKey("cn.fei.me") → "mapping:fei.me"
          │
          ▼
② 子域名匹配：subdomain="cn" → 找到 mount_type=subdomain, mount_point="cn"
   → rewrite_prefix="tech"
          │
          ▼
③ 计算后缀：suffix = path (完整路径保留) = "/shit"
          │
          ▼
④ 拼接：
   suffix === "/" 或 ""  →  distito_path = "/" + rewrite_prefix
   否则                →  distito_path = "/" + rewrite_prefix + suffix
   结果: "/tech/shit"
          │
          ▼
⑤ fetch → https://distito.com/tech/shit → 返回 HTML + 边缘缓存
```

**两种模式的统一规则**：

| 模式 | mount_point | 请求路径 | suffix | rewrite_prefix | distito 路径 |
|------|------------|---------|--------|---------------|-------------|
| path | `/` | `/` | `/` (精确匹配) | `user/fei` | `/user/fei` |
| path | `/tech` | `/tech/shit` | `/shit` | `tech` | `/tech/shit` |
| path | `/tech` | `/tech` | `` | `tech` | `/tech` |
| path | `/about` | `/about` | `` | `_pages/tech/about` | `/_pages/tech/about` |
| subdomain | `cn` | `/shit` | `/shit` | `zhongwen` | `/zhongwen/shit` |
| subdomain | `cn` | `/` | `/` | `zhongwen` | `/zhongwen` |

### 11.6 Worker 代码

```js
// Cloudflare Worker — 部署到 fei.me（含 *.fei.me 通配）

const DISTITO_ORIGIN = 'https://distito.com'

addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)
  const host = url.hostname
  const path = url.pathname

  // 1. 从 KV 获取该域名的映射
  const domainKey = hostnameToKey(host)
  const mappings = await DOMAIN_MAPPINGS.get(domainKey, 'json')
  if (!mappings) {
    return fetch(`${DISTITO_ORIGIN}${path === '/' ? '' : path}`)
  }

  let rewritePrefix, suffix

  // 2a. 优先子域名模式
  const subdomain = host.split('.')[0]
  const subMatch = mappings.find(m =>
    m.mount_type === 'subdomain' && m.mount_point === subdomain
  )
  if (subMatch) {
    rewritePrefix = subMatch.rewrite_prefix
    suffix = path
  } else {
    // 2b. 路径模式（最长匹配；根路径 / 只精确匹配）
    const pathMappings = mappings
      .filter(m => m.mount_type === 'path')
      .sort((a, b) => b.mount_point.length - a.mount_point.length)

    const pathMatch = pathMappings.find(m => {
      if (m.mount_point === '/') return path === '/'
      return path === m.mount_point || path.startsWith(m.mount_point + '/')
    })

    if (!pathMatch) return fetch(`${DISTITO_ORIGIN}/404`)

    rewritePrefix = pathMatch.rewrite_prefix
    suffix = pathMatch.mount_point === '/'
      ? '/'
      : path.slice(pathMatch.mount_point.length) || '/'
  }

  // 3. 拼接 distito.com 路径
  const distitoPath = suffix === '/' || suffix === ''
    ? `/${rewritePrefix}`
    : `/${rewritePrefix}${suffix}`

  // 4. fetch + 边缘缓存
  const response = await fetch(`${DISTITO_ORIGIN}${distitoPath}`)
  if (response.ok) {
    const headers = new Headers(response.headers)
    headers.set('cache-control', 'public, max-age=3600, s-maxage=3600')
    return new Response(response.body, { headers })
  }
  return response
}

function hostnameToKey(host) {
  const parts = host.split('.')
  return parts.length >= 3
    ? `mapping:${parts.slice(-2).join('.')}`
    : `mapping:${host}`
}
```

### 11.7 KV 数据

```
KV Namespace: DISTITO_DOMAIN_MAPPINGS

# fei.me 的映射表
mapping:fei.me → [
  {"mount_type":"subdomain", "mount_point":"cn",  "rewrite_prefix":"zhongwen"},
  {"mount_type":"subdomain", "mount_point":"en",  "rewrite_prefix":"yingwen"},
  {"mount_type":"path",      "mount_point":"/",    "rewrite_prefix":"user/fei"},
  {"mount_type":"path",      "mount_point":"/about","rewrite_prefix":"_pages/tech/about"}
]

# 独立的 Base 级域名
mapping:devops.cn → [
  {"mount_type":"path", "mount_point":"/", "rewrite_prefix":"tech"}
]
```

### 11.8 数据同步（Edge Functions → KV）

```python
# fastapi/app/services/domain_service.py

async def upsert_mapping(domain, mount_type, mount_point, rewrite_prefix):
    await repo.upsert(domain, mount_type, mount_point, rewrite_prefix)

    key = f"mapping:{domain}"
    mappings = await cf_kv.get(key, 'json') or []
    mappings = [m for m in mappings if not (
        m['mount_type'] == mount_type and m['mount_point'] == mount_point)]
    mappings.append({
        'mount_type': mount_type,
        'mount_point': mount_point,
        'rewrite_prefix': rewrite_prefix
    })
    await cf_kv.put(key, json.dumps(mappings))


async def remove_mapping(domain, mount_type, mount_point):
    await repo.delete(domain, mount_type, mount_point)
    key = f"mapping:{domain}"
    mappings = await cf_kv.get(key, 'json') or []
    mappings = [m for m in mappings if not (
        m['mount_type'] == mount_type and m['mount_point'] == mount_point)]
    if mappings:
        await cf_kv.put(key, json.dumps(mappings))
    else:
        await cf_kv.delete(key)
```

### 11.9 域名验证流程

```
① 用户在 app.distito.com/settings/domains 输入域名 fei.me
② 系统生成验证 token → 提示添加 DNS TXT 记录:
   _distito-verify.fei.me  TXT  "random-token"
③ 用户添加记录后点击「验证」
④ Edge Functions 查询 DNS TXT 记录，匹配 token → dns_verified = true
⑤ 用户开始配置路由规则:
   ┌─ 子域名 cn  → rewrite_prefix = "zhongwen"
   ├─ 路径 /about → rewrite_prefix = "_pages/about"
   └─ ...
⑥ 用户将域名 CNAME 到 distito.com
⑦ Edge Functions 同步规则到 KV
⑧ Worker 生效
```

### 11.10 对现有架构的影响

| 模块 | 影响 |
|------|------|
| **builder** | **无影响**。只生成 `distito.com` 的静态文件 |
| **webapp** | 新增 Settings → Domains 页面，路由规则管理 UI |
| **fastapi** | 新增 `CustomDomain` 模型 + 路由 + KV 同步 |
| **widget.js** | **无影响**。从 meta 读 article_id，不依赖域名 |
| **SEO** | `<link rel="canonical">` 固定指向 `distito.com` |
| **费用** | Worker 免费 10 万请求/天，KV 免费 1000 读/天 |

### 11.11 限制与边界

| 限制 | 说明 |
|------|------|
| 免费用户 | 最多绑定 3 个域名（可调） |
| SSL | 自动通过 Cloudflare 提供（免费） |
| 根域名 | 支持 `fei.me` 和 `sub.fei.me` |
| 备案 | 中国大陆域名需 ICP 备案，distito 不负责 |

### 11.12 Phase 计划

| 阶段 | 功能 |
|------|------|
| **Phase 1** | 根路径 `/` 映射；DNS TXT 验证；Worker 手动部署 |
| **Phase 2** | 子域名模式 + 路径前缀模式；KV 自动同步；路由规则 UI |
| **Phase 3** | 自助 Worker 部署；多域名管理面板 |

---

## 12. 关键设计记录

| 决策 | 选择 | 理由 |
|------|------|------|
| webapp vs website 分离 | **两个独立项目** | website 是营销展示页，webapp 是功能型 SPA，关注点不同 |
| builder 用 Python | 复用 Edge Functions 模型 | 统一语言，减少技术栈种类 |
| website 与 builder 关系 | **独立，不重叠** | website 是 SPA 通过 API 取数据；builder 只生成静态文章页 |
| Webhook 触发构建 | **发布时实时触发** | 零成本（免费额度内），用户无等待 |
| 域名分离 | **distito.com + app.distito.com** | 内容站和管理站关注点分离，SEO 不受登录态干扰 |
| URL 格式 | **`base-slug/article-slug`** | 见 injection-design.md，为未来多用户协作预留 |
| Base slug 最小长度 | **4 字符** | 3 字符以内归官网 SPA 路由，天然保护 |
| 保留路径 | **配置文件管理** | `reserved-slugs.json` 被 builder + fastapi 共用校验 |
| 自定义域名模型 | **通用 URL 重写表** | `rewrite_prefix` 不限定为 Base，支持任意路径 |
| 根路径 `/` | **精确匹配，后缀不拼接** | 防止 `/` 错误匹配所有子路径 |
| Canonical URL | **固定指向 distito.com** | 避免重复内容惩罚 |
| MVP 支持 | **不做** | Phase 1-2 引入 |
