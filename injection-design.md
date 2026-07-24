# distito Inject 知识引用与衍生网络设计

> 版本：v0.2 · 2026-07-24
> 领域：distito.com — 基于AI 对话的知识协作网络

---

## 1. 概念模型

### 1.1 什么是 Inject

**Inject** 是 distito 的知识衍生机制——**类似 GitHub 的 Fork，但面向知识文章。**

用户可以在 AI 对话中引用现有文章，将它的内容以「知识参考」身份注入当前上下文；当用户蒸馏发布新文章时，新文章自动标记「衍生自」被引用的文章，形成一张**有向知识协作网络**。

### 1.2 比喻对照

| GitHub Fork | distito Inject |
|---|---|
| Fork 一个仓库 | 引用一篇文章到对话上下文 |
| 在 Fork 上做修改 | 结合参考资料继续对话、思考 |
| 提交 PR / 发布 Release | 蒸馏发布新文章 |
| Fork 关系追溯 | `injections` 元数据字段 |
| 仓库网络图 | 知识衍生图谱 |
| 个人/组织下多个仓库 | 用户下多个 Base |

### 1.3 核心流程

```
                           引用（参考资料注入）                   蒸馏
  [原文A · 原文B] ──────────────> [当前对话] ──────────────> [新文章C]
                                     │                          │
                                     │ context:                  │ metadata:
                                     │   ┌─ 参考资料 ─┐          │   injections:
                                     │   │ 原文A (全文) │          │     [A, B]
                                     │   │ 原文B (全文) │          │   base_id: B
                                     │   └─────────────┘          │
```

---

## 2. Base（知识基）概念

### 2.1 什么是 Base

Base 是知识的集合容器，类似 GitHub 的仓库 / Medium 的出版物。每篇文章必须属于且只属于一个 Base。

### 2.2 URL 结构

```
distito.com/<user-slug>/<base-slug>/<article-slug>

示例：
  distito.com/alice/tech/ci-cd-core-principles
  distito.com/alice/thoughts/ai-insights
  distito.com/bob/life-notes/2026-summer
```

| 层级 | 说明 |
|------|------|
| `user-slug` | 用户唯一标识，如 `alice`、`bob` |
| `base-slug` | Base 在用户下的唯一标识，如 `tech`、`thoughts` |
| `article-slug` | 文章在 Base 内的唯一标识 |
| **访问用户主页** | `distito.com/<user-slug>` — 展示用户所有公开 Base |
| **访问 Base** | `distito.com/<user-slug>/<base-slug>` — Base 首页 |
| **访问文章** | `distito.com/<user-slug>/<base-slug>/<article-slug>` — 文章页 |

**引用语法**：
```
=> http://distito.com/alice/tech/ci-cd-core-principles
=> http://distito.com/alice/thoughts/ai-insights
=> http://distito.com/alice/tech/ci-cd-core-principles http://distito.com/bob/devops/production-practice
```

### 2.3 Base 的核心属性

| 属性 | 说明 |
|------|------|
| **可见性** | 全部公开可见（任何人都可阅读） |
| **蒸馏权限** | 可配置：谁有权通过 `=>` 向该 Base 蒸馏发布文章 |
| **管理权限** | 可配置：谁有权管理 Base 的配置/成员/删除文章 |
| **协作模型** | 初期一人可多个 Base，未来支持多人协同一个 Base |

---

## 3. 数据模型

### 3.1 Base

```sql
Base
├── id                      UUID PRIMARY KEY
├── user_id → User          NOT NULL             -- 所有者
├── slug                    VARCHAR NOT NULL      -- 用户内唯一
├── name                    VARCHAR NOT NULL      -- 显示名称
├── description             TEXT?                 -- 简介
├── logo                    TEXT?                 -- Base 图标/Logo URL
│
├── settings                JSONB NOT NULL DEFAULT '{}'
│   └─ {
│       "distillation": {
│           "who_can_publish": "owner",      -- "owner" | "members" | "anyone"
│           "require_approval": true,         -- 是否需要审核
│           "max_content_length": 5000,        -- 文章最大字数上限
│           "daily_publish_limit": 5           -- 每 24 小时最多发布篇数
│       },
│       "management": {
│           "who_can_admin": "owner"          -- "owner" | "admins"
│       }
│     }
│
├── created_at              TIMESTAMPTZ NOT NULL
└── updated_at              TIMESTAMPTZ NOT NULL
│
UNIQUE(user_id, slug)                              -- 用户内 base slug 唯一
```

### 3.2 User

```sql
User
├── id                      UUID PRIMARY KEY
├── slug                    VARCHAR UNIQUE NOT NULL  -- 用户唯一标识（distito.com/<slug>）
├── email                   VARCHAR?
├── github_id               VARCHAR?
├── display_name            VARCHAR NOT NULL
├── avatar_url              TEXT?
├── created_at             TIMESTAMPTZ NOT NULL
└── updated_at             TIMESTAMPTZ NOT NULL
│
UNIQUE(slug)
```

### 3.3 BaseMember（用户-Base 关系表）

```sql
BaseMember
├── id                      UUID PRIMARY KEY
├── base_id → Base          NOT NULL
├── user_id → User          NOT NULL
├── role                    VARCHAR NOT NULL DEFAULT 'owner'
│                             owner       - 创建者，完全控制
│                             admin       - 管理蒸馏配置、成员
│                             contributor - 可向该 Base 蒸馏发布
│                             viewer      - 仅可阅读（默认所有用户）
│
├── joined_at               TIMESTAMPTZ NOT NULL
│
UNIQUE(base_id, user_id)
```

**关键设计**：Base 的可见性是公开的，所以 `viewer` 角色是所有已登录用户的隐式默认角色。显式写入 `BaseMember` 的角色是 `owner` / `admin` / `contributor`。

### 3.4 Article（扩展 inject 字段）

```sql
Article
├── id                      UUID PRIMARY KEY
├── base_id → Base          NOT NULL       -- 隶属于的 Base
├── user_id → User          NOT NULL       -- 作者
│
├── title                   VARCHAR NOT NULL
├── slug                    VARCHAR NOT NULL  -- Base 内唯一
│                             UNIQUE(base_id, slug)
├── content                 TEXT NOT NULL     -- 正文（Markdown）
├── excerpt                 TEXT?             -- 摘要 / 引言
│
├── injections             UUID[] DEFAULT '{}'  -- 本文注入了哪些 distito 文章 ID（计入注入网络）
├── external_refs          TEXT[] DEFAULT '{}'  -- 本文引用了哪些外部 URL（结构化后注入，不计入网络）
├── injected_count         INTEGER DEFAULT 0    -- 被多少篇文章注入引用
│
├── related_to             UUID[] DEFAULT '{}'  -- 同一次会话中产生的关联文章
├── session_id             UUID?             -- 来源对话的会话 ID
├── agent_id               VARCHAR?          -- 蒸馏本文的 AI 代理标识，如 "kun"
├── model_id               VARCHAR?          -- 使用的模型标识，如 "gpt-4o"、"deepseek-v4-pro"
├── session_meta           JSONB DEFAULT '{}' -- 来源会话的全部元数据
│   └─ {
│       "provider": "openai",
│       "platform": "kun",
│       "title": "对话标题",
│       "message_count": 12,
│       "total_tokens": 15234,
│       "started_at": "2026-07-24T08:00:00Z"
│     }
│
├── metadata               JSONB DEFAULT '{}'
│   └─ {
│       "tags": [...],
│       "injection_notes": {
│         "<article_id>": "用户引用时写的补充说明"
│       }
│     }
│
├── status                 VARCHAR NOT NULL DEFAULT 'published'
│                             published | draft | deleted
├── published_at           TIMESTAMPTZ
├── created_at             TIMESTAMPTZ NOT NULL
└── updated_at             TIMESTAMPTZ NOT NULL
│
UNIQUE(base_id, slug)                              -- Base 内文章 slug 唯一
CHECK(length(content) <= 5000)          -- 免费用户上限 5000 字

INDEX(session_id)            -- 按会话查询所有生成的文章
INDEX(agent_id)              -- 按代理筛选
INDEX(model_id)              -- 按模型筛选
```

**`injections` vs `related_to` 的语义区别**：

| 字段 | 关系类型 | 触发条件 | 用途 |
|------|---------|---------|------|
| `injections` | **知识引用**（跨会话） | 用户 `=> <URL>` 引用外部知识 | 知识图谱追溯、inject 网络 |
| `related_to` | **同源文章**（同会话） | 同一次对话中蒸馏出多篇文章 | 追踪一次对话产出了哪些文章 |

**会话追溯**：通过 `session_id` + `agent_id`，用户可在 `app.distito.com` 上查看到：
- 某次对话蒸馏出的所有文章
- 某个 AI 代理（如 Kun）蒸馏的所有文章

### 3.5 InjectionEdge（引用关系边表）

```sql
InjectionEdge
├── id                      UUID PRIMARY KEY
├── source_article_id → Article     NOT NULL  -- 被引用的文章（upstream）
├── derived_article_id → Article    NOT NULL  -- 衍生出的文章（fork）
├── position                INTEGER NOT NULL DEFAULT 0  -- 多篇引用时的排序
├── note                    TEXT?             -- 用户引用时写的补充说明
├── created_at             TIMESTAMPTZ NOT NULL
│
UNIQUE(source_article_id, derived_article_id)
INDEX(derived_article_id)    -- 查某篇文章引用了谁
INDEX(source_article_id)     -- 查某篇文章被谁引用
```

### 3.6 SessionCitation（运行时模型，不入库）

在 Kun Loop / API 会话上下文中维护，用于追踪当前对话引用了哪些知识：

```
SessionCitation
├── session_id              当前对话的会话 ID
├── cited_articles          引用文章列表
│   └─ [
│       {
│           "article_id": UUID,
│           "base_slug": "tech",
│           "title": "CI/CD 核心原理",
│           "author": { "id": UUID, "display_name": "Alice" },
│           "excerpt": "CI/CD 的核心是自动化...",
│           "url": "https://distito.com/tech/ci-cd-core-principles",
│           "cited_at": ISO8601,
│           "note": "用户引用时写的说明"
│       }
│     ]
├── citations_count         累计引用篇数
├── total_tokens_estimate   注入内容的大致 token 量
```

**会话与文章的关系**：
- 草稿仅存在本地对话窗口中，不写入服务端
- 用户确认发布后，一次性 POST 到 API 持久化
- 用户发布后，当前对话可继续用于新的蒸馏（前文作为上下文参考）

### 3.7 Activity（用户行为日志）

```sql
Activity
├── id                      UUID PRIMARY KEY
├── user_id → User          NOT NULL          -- 操作用户
├── base_id → Base?                           -- 关联 Base（如有）
├── article_id → Article?                     -- 关联文章（如有）
├── action_type             VARCHAR NOT NULL  -- 操作类型
│                           -- "article_published"
│                           -- "article_updated"
│                           -- "article_deleted"
│                           -- "base_created"
│                           -- "base_updated"
│                           -- "base_deleted"
│                           -- "injection_made"
│                           -- "external_ref_added"
│                           -- "session_distilled"
├── metadata                JSONB DEFAULT '{}'
│   └─ {
│       "article_title": "...",
│       "base_name": "...",
│       "injected_from": ["uuid1", "uuid2"],
│       "content_length": 3200
│     }
├── created_at             TIMESTAMPTZ NOT NULL
│
INDEX(user_id, created_at)       -- 用户按时间线查询
INDEX(base_id, action_type)       -- Base 维度的统计分析
INDEX(action_type, created_at)    -- 全局活动聚合
```

### 3.8 完整 ER 关系

```
User ──1:N── BaseMember ──N:1── Base
 │                                  │
 │                                  │ 1:N
 │                                  │
 │ 1:N                             Article
 └─────────────────────────────────┤
                                   │
                                   ├── injections: UUID[] (自引用多对多)
                                   │
                                   └── 1:N ── InjectionEdge (as derived)
                                                   │
                                                   └── N:1 ── Article (as source)

User ──1:N── Activity
Base ──1:N── Activity
```

---

## 4. API 设计

### 4.1 Base

```
GET    /api/bases                          — 列出所有公开 Base
POST   /api/bases                          — 创建 Base
GET    /api/bases/:baseSlug                — 获取 Base 详情
PATCH  /api/bases/:baseSlug                — 更新 Base 设置
DELETE /api/bases/:baseSlug                — 删除 Base

GET    /api/bases/:baseSlug/members        — 成员列表
POST   /api/bases/:baseSlug/members        — 添加成员（仅 owner/admin）
DELETE /api/bases/:baseSlug/members/:id    — 移除成员
```

### 4.2 URL 解析（供引用使用）

```
GET /api/resolve?url=https://distito.com/tech/ci-cd-core-principles
```

**解析逻辑**：
1. 从 URL 提取 `base-slug` + `article-slug`
2. 查 `Base WHERE slug = :base_slug`
3. 查 `Article WHERE base_id = :base_id AND slug = :article_slug`
4. 返回文章完整内容 + metadata

**响应**：
```json
{
  "article": {
    "id": "a1b2c3d4...",
    "base": {
      "slug": "tech",
      "name": "技术博客"
    },
    "title": "CI/CD 核心原理",
    "author": {
      "id": "user-id",
      "display_name": "Alice",
      "avatar_url": "..."
    },
    "excerpt": "CI/CD 的核心是自动化...",
    "content": "## CI/CD 核心原理\n\n...全文...",
    "published_at": "2026-07-23T12:00:00Z",
    "injections": [],
    "injected_count": 3,
    "metadata": {
      "tags": ["devops", "ci-cd"]
    }
  },
  "resolved_from": {
    "type": "slug",
    "base_slug": "tech",
    "article_slug": "ci-cd-core-principles"
  }
}
```

### 4.3 文章

```
GET    /api/bases/:baseSlug/articles                              — 文章列表
GET    /api/bases/:baseSlug/articles/:articleSlug                 — 文章详情
POST   /api/bases/:baseSlug/articles                              — 发布文章
PUT    /api/bases/:baseSlug/articles/:articleSlug                 — 更新文章（替换式）
PATCH  /api/bases/:baseSlug/articles/:articleSlug                 — 编辑文章部分字段
DELETE /api/bases/:baseSlug/articles/:articleSlug                 — 删除文章

# 会话追溯
GET    /api/sessions/:sessionId/articles                          — 某次对话的所有文章
GET    /api/agents/:agentId/articles                              — 某个代理的所有文章
GET    /api/models/:modelId/articles                              — 某个模型生成的所有文章
```

**POST 发布（含 inject / related_to）**：
```json
{
  "title": "我对 CI/CD 的新思考",
  "content": "基于引用文章，我补充了...",
  "excerpt": "从 CI/CD 到持续部署的延伸思考",
  "injections": [
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  ],
  "external_refs": [
    "https://example.com/event-loop"
  ],
  "related_to": [
    "x9y8z7..."
  ],
  "session_id": "会话 UUID",
  "agent_id": "kun",
  "injection_notes": {
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890": "这篇给了我很大启发"
  },
  "metadata": {
    "tags": ["devops", "ci-cd", "deployment"]
  }
}
```

服务端自动操作：
1. 创建 Article，写入 `base_id` / `injections`
2. 为每个 source → derived 创建 `InjectionEdge`
3. 递增每个 source 文章的 `injected_count`

### 4.4 Inject 查询

```
GET /api/articles/:articleId/injections
  → 双向引用列表（哪些文章是 injections，哪些是 injected_by）

GET /api/articles/:articleId/injection-graph?depth=2
  → 知识衍生图谱（上游祖先 + 下游派生）
```

---

## 5. 引用注入设计

### 5.1 统一流程

`=>` 后可以是任意 URL。Agent 统一走一条流程：

```
用户输入 => <URL>
         │
         ▼
① web fetch → 获取 HTML
         │
         ▼
② 解析 HTML
         │
         ├─ 有 <script type="application/distito+json">?
         │    ↓
         │  识别为 inject → 提取结构化元数据
         │  从 <body> 正文标签提取全文
         │
         └─ 无 distito+json 标记?
              ↓
            降级为外部参考 → 从 OG / title 提取元数据
            从 <body> 提取正文
         │
         ▼
③ 内容质量校验 ← 新增
   ├─  正文可读内容 < 100 字符?      → ❌ 知识注入失败
   ├─  页面标题为空或含 "404/403"?    → ❌ 知识注入失败
   ├─  无明显正文（只剩导航/广告）?   → ❌ 知识注入失败
   └─  通过 → 继续
         │
         ▼
④ 组装为统一参考资料格式
   ├─ 有 distito+json + injections? → 注入 + 记录 SessionCitation
   ├─ 有 distito+json 无 injections? → 注入 + reference 类型
   └─ 无 distito+json?               → 注入 + reference 类型
```

| 来源 | 类型 | 追踪 inject |
|------|------|-----------|
| `=> distito.com/...` | `inject` | ✅ |
| `=> fei.me/...`（含 distito+json） | `inject` | ✅ |
| `=> 任意 URL（无 distito+json）` | `reference` | ❌ |

### 5.2 application/distito+json 标准

distito 生成的每个公开文章页都内嵌一个 JSON-LD 块，供 Agent 在 `=>` 注入时识别。

```html
<script type="application/distito+json">
{
  "@context": "https://distito.com/rsd/1.0",
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "title": "CI/CD 核心原理",
  "author": {
    "slug": "alice",
    "display_name": "Alice",
    "avatar_url": "https://distito.com/avatar/alice.png"
  },
  "base": {
    "slug": "tech",
    "name": "技术博客"
  },
  "published_at": "2026-07-23T12:00:00Z",
  "injections": ["uuid-xyz-789"],
  "tags": ["devops", "ci-cd"],
  "url": "https://distito.com/tech/ci-cd-core-principles"
}
</script>
```

**字段说明**：

| 字段 | 必填 | 说明 |
|------|------|------|
| `@context` | 是 | 标准标识符，Agent 靠此判定是否为 distito 内容 |
| `id` | 是 | 文章 UUID |
| `title` | 是 | 标题 |
| `author` | 是 | 作者信息（slug / display_name） |
| `base` | 是 | 所属 Base（slug / name） |
| `published_at` | 是 | 发布时间 |
| `url` | 是 | 原始文章链接（任何域名下都可追溯回原址） |
| `injections` | 否 | 上游文章 UUID 列表，有则标记为 inject 类型 |
| `tags` | 否 | 标签 |

**Agent 处理规则**：
1. fetch HTML 后扫描 `<script type="application/distito+json">`
2. 检查 `@context === "https://distito.com/rsd/1.0"`
3. 是 → 从 `<article>` 或 `<body>` 提取正文；从 script block 提取元数据
4. 有 `injections` → 标记为 inject` 类型并记录引用
5. 无 `injections` → 标记为 `reference` 类型
6. 合并组装为参考资料注入

> **开放标准**：任何第三方站点只需在页面中加入这个 `<script>` 块，
> distito 用户即可通过 `=> URL` 结构化引用其内容。
> 标准本身是开放的，未来可发布为独立规范供其他 AI 工具使用。

### 5.3 内容质量校验

注入前必须校验提取的内容是否有实质性知识，避免浪费 Token 和污染上下文。

**校验规则**：

| 规则 | 触发条件 | 反馈 |
|------|---------|------|
| 内容过短 | 清洗后的纯文本 < 100 字符 | `❌ 知识注入失败：目标页面内容过短（仅 N 字符），无实质知识` |
| 无效页面 | 标题为空，或含 `404`、`403`、`Not Found`、`Login` 等 | `❌ 知识注入失败：无法访问目标页面（状态或标题异常）` |
| 无正文节点 | `<body>` 中无可识别的 `<article>`、`<main>`、`<p>` 等正文容器 | `❌ 知识注入失败：目标页面未包含可提取的正文内容` |
| 低内容密度 | 正文字符数 < 页面 HTML 总大小的 5%（大量模板/广告代码） | `❌ 知识注入失败：目标页面实质性内容占比过低` |

**Agent 判定失败后的行为**：
- 不注入任何内容到上下文
- 不记录 SessionCitation
- 返回失败反馈给用户，不阻塞后续对话

### 5.4 注入模板（统一格式）

无论来源是 distito 文章、自定义域名还是外部 URL，注入格式一致：

```
## 📎 参考资料

以下内容已在本次对话中引用，作为参考知识。

---

### [1] CI/CD 核心原理

- **类型**: Inject 🔗（可追踪衍生关系）
- **作者**: Alice
- **Base**: 技术博客
- **原始链接**: https://distito.com/tech/ci-cd-core-principles
- **摘要**: CI/CD 的核心是自动化，包括持续集成和持续部署...

--- 正文开始 ---
## CI/CD 核心原理
...全文...
--- 正文结束 ---

---

### [2] 理解事件循环

- **类型**: 外部引用 📎（不可追踪）
- **来源**: https://example.com/event-loop
- **摘要**: JavaScript 事件循环是异步编程的核心...

--- 正文开始 ---
...
--- 正文结束 ---
```

### 5.5 设计要点

| 要点 | 说明 |
|------|------|
| **统一格式** | inject 和 reference 使用同一模板，仅类型标签不同 |
| **类型区分** | Inject 🔗 可追踪衍生；外部引用 📎 仅参考 |
| **元数据前置** | 作者/来源/摘要先于正文，模型可快速判断相关性 |
| **正文边界** | `--- 正文开始/结束 ---` 标记，防止与消息混淆 |
| **Token 优化** | 超长（>4000 token）自动截断，附截断提示 |

### 5.6 SessionCitation（仅 inject 类型记录）

```json
{
  "cited_articles": [
    {
      "article_id": "a1b2c3d4...",
      "type": "inject",
      "title": "CI/CD 核心原理",
      "author": { "id": "...", "display_name": "Alice" },
      "base": { "slug": "tech", "name": "技术博客" },
      "url": "https://distito.com/tech/ci-cd-core-principles",
      "cited_at": "2026-07-24T10:00:00Z"
    }
  ]
}
```

外部引用不写入 `cited_articles`，因此后续蒸馏发布的 `injections` 仅包含 distito 可定位的内容。

### 5.7 引用确认回复

无论成功或失败，Agent 应给出明确的反馈：

```
✅成功蒸馏 2 篇知识：

1. CI/CD 核心原理 — https://distito.com/tech/ci-cd-core-principles
2. 理解事件循环 — https://example.com/event-loop

我们可以探讨相关话题。
```

```
❌ 知识注入失败

目标页面 https://example.com/xyz 内容过短（仅 23 字符），无实质知识。
请检查链接是否正确，或换一篇内容丰富的文章。
```

| 结果 | 反馈模板 |
|------|---------|
| ✅ 注入成功 | `✅成功蒸馏 N 篇知识` + 编号列表（标题 — URL）+ `我们可以探讨相关话题。` |
| ❌ 注入失败 | `❌ 知识注入失败` + 具体原因 + 后续建议 |

---

## 6. 蒸馏发布流程（含 Injection）

### 6.1 发布流程

**会话与文章的关系**：

- 草稿仅保存在本地对话上下文中，**不发送到服务端**
- 用户通过 `=>` 反复修改、打磨草稿，全部发生在本地
- 用户回复「确认」→ 一次性 POST 到 API 持久化并公开发布
- 一次对话默认对应**一篇**蒸馏中的文章

- 用户首次 `=> 总结发布` → 创建草稿 → 预览 → 确认 → 发布
- 用户再次 `=>` 蒸馏动作 → **意图识别**：

```
用户输入 => <指令>
         │
         ▼
意图分类：
         │
 ├─ 修改/修正/补充已有内容？
 │   如 "有个单词写错了，请纠正为 xxx"
 │   "第三段可以再展开说明"
 │   "补充一个实际案例"
 │   → 自动更新当前文章（不弹窗）
 │
 ├─ 明显是另一个话题？
 │   如 "写一篇关于 xxx 的新文章"
 │   "另外总结一下项目经验"
 │   → 自动创建新文章（不弹窗）
 │
 └─ 意图不明确？
     如 "总结为文章"
     → 询问用户：

       ┌────────────────────────────────────┐
       │ 📝 当前对话已有一篇已发布文章       │
       │                                    │
       │ 「CI/CD 核心原理」                  │
       │ → https://distito.com/tech/cicd    │
       │                                    │
       │ 你要？                              │
       │ ○ 更新这篇文章（补充/修改）          │
       │ ○ 蒸馏为新的文章                    │
       │                                    │
       │ └─ 回复编号或直接描述你的需求        │
       └────────────────────────────────────┘
```

| 意图 | 示例 | 行为 |
|------|------|------|
| 明确修改 | `纠正一个错别字`、`第三段改写`、`补充案例` | 自动更新，不弹窗 |
| 明确新文章 | `写一篇新文章`、`另外总结一下 xxx` | 自动创建新文，不弹窗 |
| 不明确 | `总结为文章`、`帮我发布` | 弹窗让用户选择 |

- 选择「更新」→ PUT 更新已有文章（injections 累计追加）
- 选择「新文章」→ 创建新草稿，当前会话可绑定多篇文章
- 架构上通过 `SessionCitation.current_draft` 管理

```
① 用户基于引用内容对话后输入：
   => 总结为文章并发布到 Base「技术博客」

         ↓
② 系统蒸馏 → 生成文章预览

         ↓
③ 构建 injections
   ├─ 从 SessionCitation.cited_articles 提取文章 ID 列表
   └─ 写入新文章的 injections

         ↓
④ 展示预览（含 Inject 来源信息）
   ┌─────────────────────────────────────────┐
   │ 📝 草稿                                 │
   │                                         │
   │ # 从 CI/CD 到持续部署的延伸思考          │
   │                                         │
   │ 发布到：技术博客 (distito.com/tech)      │
   │                                         │
   │ 正文...                                 │
   │                                         │
   │ 🔗 本文衍生自：                         │
   │   · CI/CD 核心原理 — Alice              │
   │   · 生产环境部署实践 — Bob              │
   │                                         │
   │ ⚠️ 今日已发布 4 篇，剩 1 篇（限额 5/天） │
   │                                         │
   │ ├─ 回复「确认」立即发布                  │
   │ └─ 直接提修改意见                        │
   └─────────────────────────────────────────┘

         ↓
⑤ 后端校验
   ├─ 检查该 Base 今日已发布篇数
   ├─ 超过 `daily_publish_limit`? → 返回限额错误
   │     "❌ 发布失败：技术博客今日已发布 5 篇，
   │      达到每日上限（5 篇/24h），请明天再发布。"
   └─ 未超限 → 创建 Article（含 base_id / injections / injection_notes）
   ├─ 写入 InjectionEdge
   ├─ 递增 injected_count
   └─ 返回发布成功
```

### 6.2 如果用户未指定 Base

如果用户没有在 `=>` 中明确指定发布到哪个 Base：

1. 系统检测用户是否拥有多个 Base
2. 如果只有一个 → 默认发布到该 Base
3. 如果有多个 → 预览中让用户选择

```
=> 总结为文章并发布

┌─────────────────────────────────────────┐
│ 📝 草稿                                 │
│                                         │
│ ...                                     │
│                                         │
│ 📂 发布到哪个 Base？                    │
│   ○ 技术博客（当前默认）                 │
│   ○ 生活随笔                            │
│   ○ 创建新的 Base                       │
│                                         │
│ └─ 回复编号或直接输入 base-slug          │
└─────────────────────────────────────────┘
```

---

## 7. Kun Loop 指令处理

### 7.1 统一处理流

`=>` 后面可以是 URL、自然语言指令、或两者的组合。Kun Loop 统一处理：

```
用户输入
 => <URL 1> [, <URL 2> ...] [自然语言指令]
 或
 => [自然语言指令]（已在对话中上传了文件）
         │
         ▼
① 指令解析
   ├─ 前缀 `=>` 识别
   ├─ 提取 URL 列表（0 到多个）
   └─ 提取剩余的自然语言指令
         │
         ▼
② 获取内容来源
   ├─ 有 URL?          → fetch → 解析 → 质量校验
   │                     ├─ distito+json? → inject 类型
   │                     └─ 无?           → reference 类型
   │
   ├─ 有附件/文件?      → 从对话上下文读取已上传的文件内容
   │                     作为蒸馏源材料（不标记 inject）
   │
   └─ 两者都无?         → 蒸馏当前对话内容本身
         │
         ▼
③ 注入到上下文（仅 URL 引用场景需要注入）
         │
         ▼
④ 执行自然语言指令
   ├─ 有指令 → 执行指令（翻译/融合/发布…）
   │           生成内容后给出融合反馈
   │
   └─ 无指令 → 给出独立注入反馈
               "✅成功蒸馏 N 篇知识：\n1. 标题 — <url>\n我们可以探讨相关话题。"
```

**三种内容来源的行为差异**：

| 来源 | 触发 | 内容获取 | inject 记录 |
|------|------|---------|-----------|
| URL | `=> <url> 指令` | web fetch | ✅ 记录 injections |
| 附件/文件 | `=> 指令`（已传文件） | 从对话上下文读取 | ❌ 不记录 |
| 对话本身 | `=> 指令`（无 URL 无文件） | 蒸馏对话历史 | ❌ 不记录 |

### 7.2 指令示例

| 输入 | 行为 |
|------|------|
| `=> 总结为文章并发布` | 蒸馏当前对话 → 预览 → 确认 → 发布 |
| `=> <url>` | 引用知识注入上下文，等待后续对话 |
| `=> <url> 翻译为英文` | 引用 + AI 翻译 → 输出 |
| `=> <url1>, <url2> 融合观点` | 引用多篇 + AI 融合 → 输出 |
| `=> <url> 总结为文章并发布到 tech base` | 引用 + 蒸馏 → 预览 → 确认 → 发布（含 injections） |

### 7.3 提示词与输出模板

#### 系统行为定义

当用户输入以 `=>` 开头时，Agent 切换为「distito 处理模式」：

```
你正在运行 distito 知识协作协议。

核心理念：Distill what people learned and what others like to read.
从每一次对话和每一次 fetch 中，提取真正有价值的知识，
去除噪声、精简冗余，只留下可供分享和引用的精华。

用户输入以 => 开头，后接 URL 和/或自然语言指令。

你的职责：
1. 解析指令，提取 URL 列表和自然语言指令
2. 对每个 URL 执行 web fetch，获取内容
3. 检查内容质量，拒绝无实质知识的页面
4. 将有效内容以参考资料格式注入当前上下文
5. 如有功能指令（翻译/融合/发布），执行后输出结果

注意事项：
- 从 HTML 中优先提取正文内容，去除导航/广告/页脚
- 检查 <script type="application/distito+json"> 获取结构化元数据
- 有 injections 字段的标记为 inject 类型并记录引用关系
- 外部 URL 无 distito+json 则标记为 reference 类型
- 不要将参考知识与你的回答混淆
- 发布前提醒用户今日剩余可发布篇数（如 "今日已发布 3 篇，剩余 2 篇"）
```

#### 内容提取与结构化

```
核心理念：Distill what people learned and what others like to read.
不要逐字搬运原文，而是提取其中有知识价值的部分。

请解析以下 HTML 页面，提取：

1. 标题（优先 og:title，其次 <title>）
2. 正文内容（从 <article> 或 <main> 或 <body> 提取可读文本，去除 script/style/nav/footer）
3. 元数据：
   - 是否有 <script type="application/distito+json">？如有，解析其中的 @context, id, title, author, base, injections
   - og:description 或 meta[name=description]
   - 发布日期（如有）
4. 内容统计：清洗后纯文本字数
```

#### 质量校验

```
内容注入前进行以下检查。任一失败则拒绝注入并返回反馈。

1. 纯文本字数 < 100？→ ❌ 知识注入失败：目标页面内容过短（仅 N 字符），无实质知识
2. 标题为空或含 "404" / "403" / "Not Found" / "Login"？→ ❌ 知识注入失败：无法访问目标页面
3. 无 <article> / <main> / <p> 等正文节点？→ ❌ 知识注入失败：目标页面未包含可提取的正文内容
4. 正文字符数 < HTML 总大小 5%？→ ❌ 知识注入失败：目标页面实质性内容占比过低

通过检查后继续注入。
```

#### 参考资料注入格式

每次注入使用统一格式，注入到系统消息或用户消息末尾：

```
## 📎 参考资料

以下内容已在本次对话中引用，作为参考知识。

---

### [1] {{标题}}

- **类型**: {{Injection 🔗 或 外部引用 📎}}
- **作者**: {{作者名}}
- **来源**: {{原始链接}}
- **摘要**: {{摘要}}

--- 正文开始 ---
{{全文}}
--- 正文结束 ---

---

### [2] {{下一标题}}
...
```

#### 功能指令执行

**翻译**
```
用户指令：翻译为{{目标语言}}

请将以上参考资料中的正文内容翻译为{{目标语言}}。
保留原文的 Markdown 格式和标题层级。
翻译完成后，以以下格式输出：

---
## {{翻译后标题}}

{{翻译正文}}
---
```

**融合**
```
用户指令：{{融合要求}}

你已获取 {{N}} 篇参考资料。请按用户要求将它们融合。

输出格式：
- 先列出来源清单（编号、标题、URL）
- 然后是融合后的内容
```

#### 蒸馏发布预览模板

```
📝 草稿

# {{标题}}

发布到：{{Base 名称}} (distito.com/{{base-slug}})

{{正文}}

🔗 本文衍生自：
  · {{来源文章标题}} — {{作者}}
  · {{来源文章标题}} — {{作者}}

├─ 回复「确认」立即发布
└─ 直接提修改意见
```

#### 用户反馈格式总表

| 场景 | 反馈格式 |
|------|---------|
| 注入成功（纯引用，无指令） | `✅成功蒸馏 N 篇知识：\n1. {{标题}} — <url>\n我们可以探讨相关话题。` |
| 注入失败 | `❌ 知识注入失败\n\n{{具体原因}}\n请检查链接是否正确，或换一篇内容丰富的文章。` |
| 翻译完成 | `✅翻译完成：\n---\n{{翻译结果}}\n---\n回复「确认」发布，或直接提修改意见。` |
| 融合完成 | `✅融合完成（基于 {{N}} 篇参考资料）：\n---\n{{融合结果}}\n---` |
| 发布成功 | `✅ 文章已发布 → https://distito.com/{{base-slug}}/{{article-slug}}` |
| 发布预览 | 见上方「蒸馏发布预览模板」 |

---

## 8. 前端展示

### 8.1 Base 首页

```
distito.com/tech

┌──────────────────────────────────────┐
│  📡 技术博客                          │
│  关于技术、架构、工程实践的思考        │
│                                      │
│  Alice · 成立 2026-07 · 23 篇文章    │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  CI/CD 核心原理               │  │
│  │  2026-07-23 · 🔗 5 篇衍生     │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │  SolidJS 性能优化指南          │  │
│  │  2026-07-22 · 🔗 2 篇衍生     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### 8.2 文章页 Inject 区块

```
distito.com/tech/ci-cd-core-principles

┌──────────────────────────────────────────┐
│  # CI/CD 核心原理                         │
│  by Alice · 2026-07-23 · Base: 技术博客  │
│                                          │
│  正文...                                 │
│                                          │
│  ──────────────────────────────────────  │
│  🔗 本文衍生自                           │
│    → DevOps 入门 (charlie/devops)        │
│                                          │
│  ──────────────────────────────────────  │
│  🔗 衍生自此文（3 篇）                    │
│    CI/CD 新思考 → Bob (bob/thoughts)     │
│    生产环境实践 → Dave (devops/prod)     │
│    前端 CI/CD → Alice (tech/frontend)    │
│                                          │
│  查看知识网络 →                           │
└──────────────────────────────────────────┘
```

---

## 9. 关键决策记录

| 决策 | 选项 | 选择 | 理由 |
|------|------|------|------|
| URL 结构 | 含用户名 vs 不含 | **`user-slug/base-slug/article-slug`** | 用户命名空间更直观，类似 GitHub `user/repo` 结构 |
| Base 用户关系 | 独立表 vs 用户字段 | **BaseMember 独立表** | 灵活支持未来多用户协同 |
| 文章-Base 关系 | 多对多 vs 一对多 | **一对多（属于一个 Base）** | 简单清晰，每个 Base 有独立知识体系 |
| 注入方式 | 直接塞入 vs 参考资料包装 | **参考资料包装** | 明确语义，给模型正确的上下文指引 |
| 引用上限 | 固定 vs 弹性 | **≤ 5 篇/次** | 平衡上下文窗口和引用质量 |
| 循环引用检测 | 必要 vs 非必要 | **必要** | A → B 后 B → A 无意义，需阻断 |
| Base 可见性 | 可配置 vs 强制公开 | **全部公开可见** | 构建知识网络的前提是公开可引用 |
| 存储 inject 关系 | 数组 vs 边表 | **两者都用** | 数组快速读，边表供图谱遍历 |
