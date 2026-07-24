# distito

> **A Knowledge Collaboration Network Based on AI Conversations.**

Distill what you learn in AI conversations. Build a knowledge network without context boundaries.

---

## What is distito

distito is an **AI-native knowledge collaboration platform**. It transforms AI conversations into structured, shareable, and citable knowledge nodes — forming a living knowledge network that grows with every conversation.

### Core Loop

```
User chats with AI
  → Types => to distill insights
  → Publishes as a knowledge node
  → Others reference via => URL
  → Inject, reuse, build upon
```

## Why distito

**Knowledge shouldn't be trapped in conversation history.**

- **`=>`** — A universal command to distill, cite, and publish knowledge from any AI conversation
- **`=> <URL>`** — Inject any web page as structured reference knowledge into your current conversation
- **Injection Network** — Every article tracks its sources (`injections`) and descendants (`injected_by`), forming a directed knowledge graph
- **Custom Domains** — Own your namespace. Map `fei.me/cn` → your Chinese base, `fei.me/en` → your English base
- **Open Protocol** — `application/distito+json` standard makes any page AI-citable

## Tech Stack

| Layer | Technology | Deployment |
|-------|-----------|------------|
| Public Site | Static HTML + widget.js (SolidJS) | Cloudflare Pages |
| Admin SPA | SolidJS + StyleX + Vite | Cloudflare Pages |
| API | Supabase Edge Functions (TypeScript/Deno) | Supabase (Free Tier) |
| Database | PostgreSQL 16 (Supabase) | Supabase (Free Tier) |
| Auth | Supabase Auth (GitHub OAuth) | Built-in |
| MCP Server | Python (mcp SDK) | PyPI (`pip install distito-mcp`) |
| Static Builder | Python (Jinja2) | Cloudflare Pages (build time) |
| DNS/CDN | Cloudflare | — |

**Monthly cost: $0** (all services on free tiers)

## Project Structure

```
distito/
├── builder/         Static site generator (Python)
├── webapp/          Admin dashboard SPA (SolidJS)
├── website/         Landing & discovery SPA (SolidJS)
├── api/             Supabase Edge Functions (TypeScript)
├── mcp-server/      MCP protocol handler (Python, pip install)
├── database/        DDL schemas & seed data
├── ops/             Cloudflare Pages configs
├── AGENT.md         Chinese product overview (detailed)
├── injection-design.md  Full feature design document (Chinese)
├── site-architecture.md  Architecture & deployment design (Chinese)
├── reserved-slugs.json   Slug reservation list
└── .gitignore
```

## Architecture Overview

```
                    ┌─────────────┐
                    │  Cloudflare │
                    │    CDN      │
                    └──────┬──────┘
                           │
         ┌─────────────────┼──────────────────┐
         ▼                 ▼                   ▼
  distito.com         app.distito.com    api.distito.com
  (Static HTML)       (SolidJS SPA)      (Edge Functions)
  Cloudflare Pages    Cloudflare Pages   Supabase
         │                                   │
         │                                   ▼
         │                            ┌──────────────┐
         │                            │  Supabase DB  │
         │                            │  PostgreSQL   │
         │                            └──────────────┘
         │
    ┌────┴────┐
    │  User   │
    │  Local  │
    │ mcp-    │
    │ server  │
    └─────────┘
```

## Quick Start (MCP Server)

```bash
pip install distito-mcp && distito-mcp setup
```

Then type `=>` in your AI conversation to distill, cite, and publish.

---

**distito = distill + to**

*Built from AI conversations. For AI conversations.*
