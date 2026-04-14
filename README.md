# 🏗️ Software Project Team

A team of 13 specialized AI agents for building production-grade software projects. Built on Vertical Slice Architecture + Clean Architecture + Mediator pattern with .NET, PostgreSQL, RabbitMQ, Redis, Elasticsearch, and more.

## Installation

```bash
/team install https://github.com/mkurak/agent-workshop-software-project-team.git
```

> Requires [Agent Team Manager](https://github.com/mkurak/agent-workshop-agent-team-manager-skill) to be installed first.
> Dependency: [Core](https://github.com/mkurak/agent-workshop-core) (auto-installed via team.json).

## Agents (13)

### 🧠 API Agent (`api-agent`) — 17 children

The brain of every project. All business logic lives in Domain + Application layers. Other services are bridges.

**Topics:** Architecture layers, audit trail, caching (pipeline behavior + Redis), concurrency (optimistic locking), dynamic settings (DB + Redis), error handling, file storage (MinIO/S3), idempotency, logging (virtual debug), multi-tenancy (User + Profile + Tenant), naming conventions, notification (HTTP + RMQ), pagination (cursor-based), RMQ topology, seed data, soft delete, workflows.

---

### ⚡ Socket Agent (`socket-agent`) — 17 children

SignalR WebSocket bridge. Real-time communication between clients and API. No business logic.

**Topics:** Hub design, auth (JWT query string), internal endpoints, API client pattern, group management, connection tracking, reconnection handling, CORS, rate limiting, event conventions, data boundary, multi-device, presence/typing, heartbeat, error handling, event versioning, hub method blueprint.

---

### ⏰ Worker Agent (`worker-agent`) — 6 children

Scheduled background jobs with Cronos. Calls API via HTTP. No business logic.

**Topics:** Job blueprint, cron scheduling, API client pattern, distributed locking, graceful shutdown, health monitoring.

---

### 📱 Flutter Agent (`flutter-agent`) — 21 children

Mobile/tablet specialist. iOS + Android from a single Dart codebase. UI bridge — no business logic.

**Topics:** Screen blueprint, state management (Riverpod), routing (go_router), API integration (Dio), responsive design, i18n, theme system, component design, form handling, offline-first, push notifications, auth flow, image handling, list patterns, navigation patterns, testing, deep linking, permissions, app lifecycle, error/loading states, WebView.

---

### 🌐 React Agent (`react-agent`) — 23 children

Web UI specialist. TypeScript + Vite (or Next.js for SSR). Admin panels, dashboards, web apps. No business logic.

**Topics:** Component blueprint, component design, state management (React Query + Zustand), routing (React Router), API integration (Axios), form handling (React Hook Form + Zod), styling (Tailwind CSS), auth flow, table patterns (TanStack Table), list patterns, modal/dialog, error/loading states, layout patterns, i18n, file upload, testing (Vitest + RTL + MSW), WebSocket (SignalR), permission guard, chart/dashboard, keyboard shortcuts, print/export, SEO/meta, white-label.

---

### 🐳 Infra Agent (`infra-agent`) — 16 children

Docker, Compose, CI/CD, environment management. Everything runs in containers.

**Topics:** Compose blueprint, compose architecture, Dockerfile patterns, volume strategy, env management, health checks, port management, hot reload, logging infra, MinIO setup, CI/CD (GitHub Actions), production Dockerfile, backup/restore, multi-project coexistence, resource limits, SSL/TLS local.

---

### 🗄️ Database Agent (`database-agent`) — 9 children

PostgreSQL + EF Core specialist. Schema design, migrations, indexing, query optimization.

**Topics:** Schema design blueprint, migration management, index strategy, query optimization, naming conventions, relationships, data types, constraint patterns, monitoring.

---

### 🔴 Redis Agent (`redis-agent`) — 8 children

Cache, session, distributed lock, rate limiting specialist. Ephemeral data only.

**Topics:** Cache blueprint, key naming, data structures, TTL strategy, pub/sub, memory management, connection management, distributed patterns.

---

### 🐇 RMQ Agent (`rmq-agent`) — 8 children

RabbitMQ messaging specialist. Exchange/queue topology, producer/consumer patterns, reliability.

**Topics:** Consumer blueprint, topology design, exchange patterns, retry/DLX, idempotency, message serialization, connection management, monitoring.

---

### 🔍 Code Reviewer (`code-reviewer`) — 8 children

Reviews code changes for quality, security, performance, and convention compliance.

**Topics:** Review blueprint, SOLID check, naming review, security scan, performance check, error handling review, logging review, API review.

---

### 📋 Project Reviewer (`project-reviewer`) — 7 children

Reviews overall project health, architecture, dependencies, and technical debt.

**Topics:** Review blueprint, architecture review, dependency audit, tech debt, documentation check, configuration review, scalability assessment.

### 🎨 Design System Agent (`design-system-agent`) — 10 children

Visual foundation of every project. Colors, typography, spacing, icons, component tokens, accessibility. One design system, all platforms.

**Topics:** Design blueprint, color system, typography, spacing system, icon strategy, elevation/shadow, component tokens, accessibility (WCAG 2.1 AA), dark mode, animation/motion.

---

### 🧑‍🎨 UX Agent (`ux-agent`) — 10 children

User experience specialist. Screen flows, navigation patterns, form design, data presentation, feedback systems.

**Topics:** Screen flow blueprint, navigation UX, form UX, data presentation, feedback patterns, onboarding UX, mobile vs tablet vs web, notification UX, error UX, accessibility UX.

## Architecture Overview

```
┌─────────────────────────────────────────┐
│                  API                     │  ← Brain (all logic)
│  Domain → Application → Infrastructure  │
│  Minimal API Endpoints (bridges)        │
└──────────┬──────────┬───────────────────┘
           │          │
    ┌──────┴──┐  ┌────┴────┐
    │ Socket  │  │ Worker  │  ← Bridges (HTTP to API)
    │ SignalR │  │  Cronos │
    └─────────┘  └─────────┘
           │
    ┌──────┴──────────────────┐
    │      RabbitMQ           │  ← Async messaging
    ├─────────┬───────────────┤
    │LogIngest│  MailSender   │  ← Consumers
    │→Elastic │  →SMTP        │
    └─────────┴───────────────┘
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| API | .NET 9, Minimal API, Mediator (source gen) |
| Database | PostgreSQL 17, EF Core 9 |
| Messaging | RabbitMQ 3 (fanout exchanges) |
| Cache | Redis 7 |
| Logging | Serilog → RMQ → Elasticsearch 8 + Kibana |
| Email | MailSender consumer → Mailpit (dev) |
| Auth | JWT HS256, BCrypt, X-Internal-Token |
| Storage | MinIO (dev) / S3 (prod) |
| Mobile | Flutter (Riverpod, go_router) |
| Web UI | React + TypeScript + Vite (or Next.js) |
| Infrastructure | Docker Compose, dotnet watch |

## Recommended Companion Skills

- [Brainstorm Skill](https://github.com/mkurak/agent-workshop-brainstorm-skill) — structured brainstorming with persistent state
- [Rule Skill](https://github.com/mkurak/agent-workshop-rule-skill) — coding rule management with guided wizard
- [Core](https://github.com/mkurak/agent-workshop-core) — memory system, journal, save-learnings (auto-installed as dependency)

## Key Rules

- **Everything runs in Docker** — no local SDK installations (except Flutter)
- **Minimal API only** — no Controllers
- **martinothamar/Mediator** — source generator, not MediatR
- **LogIngest logs to console only** — prevents infinite RMQ loop
- **X-Internal-Token** for system-to-system auth
- **API is the brain** — all logic lives there, everything else is a bridge

## License

MIT
