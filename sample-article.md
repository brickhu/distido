# 为什么我把后端从 FastAPI 换成了 Supabase

做 distito 的时候，我一开始选了 Python FastAPI + Railway。

理由很充分：MCP Server 是 Python 写的，API 也用 Python，模型可以两边共用，省事。Docker 部署，标准流程，没什么毛病。

结果用了两周发现一个问题：**两个月前注册的 Fly.io 账号过期了。**

换 Railway。$5/月信用金，够用。但又觉得不对劲——我为了一个 MVP，要维护两个服务（API + DB），跨平台排查问题，还要自己搞 JWT、GitHub OAuth 回调。

然后我认真看了一眼 Supabase Free Tier：Edge Functions 是免费包含的。

意味着什么？数据库 + API + Auth + 部署，全部在同一平台。**少一个服务维度，少一份运维。**

代价是什么？API 要从 Python 重写成 TypeScript（Deno）。

但想想：MCP Server 是用户本地的 pip 包，API 是服务端的 Edge Functions，它们本来就独立部署。共享 Pydantic 模型这个"收益"，实际上三个月没给我省过任何事。

于是就切了。API 重写花了两天，但之后每个月 Railway 那 $5 不用付了。

---

**一些具体的发现：**

| 我之前担心的 | 实际怎么样 |
|------------|-----------|
| Edge Functions 有执行超时 | 重活都在本地 MCP Server，API 只做 CRUD，毫秒级完成 |
| Deno 性能不如 Python | 冷启动比 Python Docker 容器还快，用了几天没人感知差异 |
| 后台任务触发不了 Webhook | Supabase Database Webhooks 完美替代 |

最爽的是 Auth。之前要配 `JWT_SECRET`、`GITHUB_CLIENT_ID`、`GITHUB_CLIENT_SECRET` 三个环境变量，写回调路由。现在 Supabase Dashboard 里打开 GitHub OAuth 开关就好。Edge Functions 里一行 `supabase-js.getUser()` 就鉴权完了。

---

一个反思：**"代码通用"的诱惑，经常让人忽略"少一个服务"的价值。** 这次经验之后，我对于 MVP 的架构原则变成了：优先选能合并到同一平台的方案，哪怕要换语言。

当然也不是没有新问题——Edge Functions 在复杂聚合查询时会不会有性能瓶颈？文件上传用 Edge Functions + Storage 够不够？这些还没遇到，但值得留意。

---

*这篇来自 distito 产品设计对话，同一对话还讨论了 URL 结构设计和注入协议。*
