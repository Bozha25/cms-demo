# CMS — Full Stack Management Dashboard

> **Internal management platform** for inventory, orders, business operations, and admin.  
> Designed for teams based in mainland China with full Chinese character support, Pinyin search, and bilingual UI.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Hosting & Infrastructure](#hosting--infrastructure)
- [Chinese String Handling](#chinese-string-handling)
- [Repository Structure](#repository-structure)
- [Backend Architecture](#backend-architecture)
- [Frontend Modules](#frontend-modules)
- [Database Design](#database-design)
- [CI/CD Pipeline](#cicd-pipeline)
- [Demo Project](#demo-project)
- [Local Development](#local-development)
- [Environment Variables](#environment-variables)
- [Security](#security)
- [GitHub Integration](#github-integration)
- [Roadmap](#roadmap)
- [Current Status](#current-status)

---

## Project Overview

| Property | Value |
|---|---|
| Project Name | CMS |
| Type | Internal Management Dashboard |
| Users | Staff in mainland China |
| Languages | English + Chinese (zh-CN) |
| Currency | CNY (¥) |
| Hosting | AWS ap-east-1 — Hong Kong |
| ICP License | Not required (HK hosting, internal use only) |
| Repos | `cms` (private) · `cms-demo` (public) |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14 (App Router) + TypeScript |
| Styling | Tailwind CSS + shadcn/ui (Nova preset — Radix, Lucide icons, Geist font) |
| Backend | C# / .NET 9 — ASP.NET Core Web API |
| ORM | Entity Framework Core 9 + LINQ |
| Relational DB | SQL Server 2022 (AWS RDS Multi-AZ) |
| Document DB | MongoDB Atlas — logs, audit, history snapshots |
| Cache | Redis ElastiCache — sessions, Pinyin index, rate limiting |
| Search | OpenSearch — IK Chinese analyzer + Pinyin plugin |
| Media / Files | AWS S3 — pre-signed URLs, versioned buckets |
| Auth | ASP.NET Core Identity + JWT Bearer + refresh token rotation |
| Containers | Docker → Amazon ECR → Amazon ECS Fargate |
| CI/CD | GitHub Actions |
| IaC | AWS CDK (TypeScript) |

---

## Hosting & Infrastructure

### Region Decision

| Factor | CN (cn-north-1) | HK (ap-east-1) ✅ Chosen |
|---|---|---|
| ICP License | Required (months of process) | Not required |
| Latency to Shanghai | 5–15 ms | 40–80 ms |
| GFW Risk | None | Low (internal tool, acceptable) |
| AWS Services | Reduced catalogue | Full catalogue |
| AWS Account | Separate China partition | Single global account |
| Setup Complexity | High — Chinese entity needed | Low — standard AWS |

> **Decision:** AWS ap-east-1 (Hong Kong) for internal-use phase.  
> If the platform goes public or processes payments, revisit mainland China hosting with ICP Commercial License.

### Dual-AZ Architecture (ap-east-1)

```
Internet
    │
    ▼
ALB (AZ-a + AZ-b)
    │
    ├── ECS Fargate — cms-api        (AZ-a + AZ-b, Auto Scaling)
    └── ECS Fargate — cms-frontend   (AZ-a + AZ-b, Auto Scaling)
         │
         ├── RDS SQL Server 2022     (Multi-AZ — primary AZ-a, standby AZ-b)
         ├── Redis ElastiCache       (primary AZ-a, replica AZ-b)
         ├── OpenSearch              (3-node cluster across both AZs)
         ├── MongoDB Atlas           (cn-east-1 or ap-east-1)
         └── S3 (ap-east-1)         ──CRR──▶ S3 (ap-southeast-1 DR)
```

### Instance Sizing

| Service | Staging / Demo | Production |
|---|---|---|
| ECS API | t3.small | c6i.xlarge |
| RDS SQL Server | db.t3.medium, single-AZ | db.r6i.2xlarge, Multi-AZ |
| Redis | cache.t3.micro | cache.r6g.large |
| OpenSearch | t3.small.search ×1 | r6g.large.search ×3 |
| Estimated cost | ~$80–120/month | ~$600–800/month |

### Disaster Recovery

- S3 Cross-Region Replication: ap-east-1 → ap-southeast-1 (Singapore). RPO ~5 min
- RDS Multi-AZ automatic failover: ~60 seconds
- Route 53 health checks: automatic DNS failover to Singapore if HK unhealthy
- DR Target: AWS ap-southeast-1 (Singapore)

### Early Stage Access

- Local dev: `localhost` (no domain needed)
- Staging: EC2 Elastic IP — `http://<IP>:3000` (frontend) · `http://<IP>:5000` (API)
- Upgrade to domain + HTTPS when going public or for portfolio showcase

---

## Chinese String Handling

### SQL Server Collation

```sql
-- Database collation
CREATE DATABASE CmsDb
  COLLATE Chinese_PRC_CI_AS;

-- Entity columns
Name        NVARCHAR(500)  COLLATE Chinese_PRC_CI_AS NOT NULL,
NamePinyin  NVARCHAR(1000) NULL  -- generated at write-time
```

| Collation | Use Case |
|---|---|
| Chinese_PRC_CI_AS | Pinyin-order sort, case-insensitive. Default for all Chinese text. |
| Chinese_Simplified_Pinyin_100_CI_AS | Explicit Pinyin sort for user-facing sorted lists. |
| Chinese_Simplified_Stroke_Order_100_CI_AS | Stroke-count order — traditional directory style. |

### Pinyin Generation

```csharp
// NuGet: TinyPinyin.Net
product.Name       = dto.Name;
product.NamePinyin = PinyinHelper
    .GetPinyin(dto.Name, " ", PinyinFormat.WITHOUT_TONE)
    .ToLowerInvariant();
// "库存管理" => "ku cun guan li"
```

### LINQ Search (Chinese + Pinyin simultaneously)

```csharp
var results = _db.Products
    .Where(p => p.Name.Contains(q)
             || p.NamePinyin.Contains(q)
             || p.Name.StartsWith(q));
```

### OpenSearch Index Mapping

```json
{
  "mappings": {
    "properties": {
      "name":       { "type": "text", "analyzer": "ik_max_word" },
      "namePinyin": { "type": "text", "analyzer": "pinyin" }
    }
  }
}
```

---

## Repository Structure

### Two Repos

| Repo | Visibility | Purpose |
|---|---|---|
| `cms` | Private | Production codebase |
| `cms-demo` | Public | Portfolio showcase |

### Folder Layout (both repos identical)

```
cms/
├── src/
│   ├── backend/                    # C# / .NET 9 solution
│   │   ├── Cms.Domain/             # Entities, value objects — no dependencies
│   │   ├── Cms.Application/        # CQRS, MediatR, DTOs, FluentValidation
│   │   ├── Cms.Infrastructure/     # EF Core, MongoDB, S3, Redis, OpenSearch
│   │   ├── Cms.Api/                # ASP.NET Core endpoints, JWT, Swagger
│   │   └── Cms.Tests/              # xUnit — unit + integration tests
│   └── frontend/                   # Next.js 14 app
│       ├── src/app/                # App Router pages & layouts
│       ├── src/components/         # shadcn/ui + custom components
│       ├── src/lib/                # API clients, hooks, utils
│       └── public/                 # Static assets (self-hosted fonts)
├── infra/                          # AWS CDK (TypeScript)
├── docs/                           # Architecture docs, ADRs
├── seed/                           # Demo data seeders
├── .github/
│   └── workflows/
│       ├── ci.yml                  # PR: build + test
│       ├── deploy-staging.yml      # develop → staging (auto)
│       └── deploy-prod.yml         # main → production (manual gate)
├── docker-compose.yml              # Local dev stack
├── .env.example                    # Environment variable template
└── PROJECT.md                      # This file
```

### Branch Strategy

```
main        ──▶ Production  (protected, PR + approval required)
develop     ──▶ Staging     (auto-deploy on merge)
feature/*   ──▶ Dev machines only
```

---

## Backend Architecture

### Clean Architecture — Dependency Direction

```
Cms.Api ──▶ Cms.Application ──▶ Cms.Domain
              ▲
Cms.Infrastructure
```

### Layer Responsibilities

| Layer | Responsibility |
|---|---|
| Cms.Domain | Pure C# entities, value objects, domain events. Zero external dependencies. |
| Cms.Application | CQRS with MediatR. Commands, Queries, DTOs, FluentValidation, service interfaces. |
| Cms.Infrastructure | EF Core DbContext, SQL Server migrations, MongoDB driver, AWSSDK S3, Redis StackExchange, OpenSearch.Net. |
| Cms.Api | ASP.NET Core 9 endpoints, middleware, JWT auth, rate limiting, Swagger/OpenAPI. |
| Cms.Tests | xUnit, integration tests per endpoint, test containers for SQL/Mongo. |

### Key NuGet Packages

| Package | Layer | Purpose |
|---|---|---|
| MediatR | Application | CQRS command/query bus |
| FluentValidation | Application | Request validation |
| TinyPinyin.Net | Application | Chinese → Pinyin conversion |
| Microsoft.EntityFrameworkCore.SqlServer | Infrastructure | EF Core SQL Server driver |
| MongoDB.Driver | Infrastructure | MongoDB client |
| StackExchange.Redis | Infrastructure | Redis client |
| OpenSearch.Net | Infrastructure | OpenSearch client |
| AWSSDK.S3 | Infrastructure | S3 pre-signed URLs |
| Microsoft.AspNetCore.Authentication.JwtBearer | Api | JWT auth middleware |

---

## Frontend Modules

| Route | Purpose |
|---|---|
| `/` | Redirect to dashboard or login |
| `/login` | Auth — email/password, guest login (demo only) |
| `/dashboard` | KPI overview — inventory alerts, order volume, revenue (CNY) |
| `/inventory` | Product list — Chinese/Pinyin search, category tree, S3 image upload |
| `/inventory/[id]` | Product detail — variants, stock per warehouse, audit history |
| `/orders` | Order management — status workflow, timeline, assign staff |
| `/orders/[id]` | Order detail — line items, fulfilment, payment status |
| `/suppliers` | Supplier CRM — Chinese company names, contacts, purchase orders |
| `/reports` | Charts (recharts), date-range filters, PDF/Excel export |
| `/admin/users` | User management — RBAC (Admin / Manager / Viewer) |
| `/admin/audit` | Audit log viewer — MongoDB-backed, filterable |
| `/admin/settings` | System config — CNY, zh-CN locale, warehouse config |

### Chinese UI Checklist

- [ ] `<html lang="zh-CN">` in `layout.tsx`
- [ ] Self-host Noto Sans SC — never load from Google Fonts (blocked in China)
- [ ] Tailwind config: extend `fontFamily.sans` with `'Noto Sans SC'`
- [ ] Number format: `Intl.NumberFormat('zh-CN', { style: 'currency', currency: 'CNY' })`
- [ ] Search: `/api/search?q=` — backend handles Chinese + Pinyin matching
- [ ] Remove all Google APIs, reCAPTCHA, YouTube embeds, Meta/Twitter SDKs
- [ ] All CDN assets replaced with local npm builds

---

## Database Design

### SQL Server — Core Tables

| Table | Key Fields |
|---|---|
| Products | Id, Name (NVARCHAR, Chinese_PRC_CI_AS), NamePinyin, SKU, CategoryId, Price, Stock |
| Categories | Id, Name, NamePinyin, ParentId, SortOrder (hierarchical) |
| Orders | Id, OrderNumber, Status, CustomerId, WarehouseId, TotalAmount, CreatedAt |
| OrderItems | OrderId, ProductId, Quantity, UnitPrice, Discount |
| Users | Id, Email, DisplayName, Role, PasswordHash, LastLogin, IsActive |
| Suppliers | Id, CompanyName, CompanyNamePinyin, ContactName, Phone, Address |
| Warehouses | Id, Name, Location, ManagerId, IsActive |

### MongoDB — Collections

| Collection | Purpose |
|---|---|
| audit_events | Immutable audit trail — full before/after snapshots per mutation |
| app_logs | Structured application logs with Chinese text fields |
| order_history | Full order document at each state transition (snapshot pattern) |
| user_sessions | Login analytics — IP, device, actions |
| notifications | In-app notification queue — read/unread per user |

### Redis — Key Usage

| Key Pattern | Purpose |
|---|---|
| `refresh:{userId}` | JWT refresh token store, TTL = token expiry |
| `ratelimit:{ip}` | Sliding window rate limit counter |
| `pinyin:{hash}` | Cached Pinyin mappings for hot products |
| `session:{userId}` | Lightweight user preferences |
| `lock:stock:{sku}` | Distributed lock — prevent concurrent stock deduction |

---

## CI/CD Pipeline

### Workflows

| Workflow | Trigger | Steps |
|---|---|---|
| `ci.yml` | Every PR to main/develop | dotnet build → dotnet test → next build → eslint → tsc |
| `deploy-staging.yml` | Merge to develop | Build Docker → push ECR → deploy ECS staging → smoke test |
| `deploy-prod.yml` | Release tag `v*.*.*` + manual approval | Deploy ECS production (both AZs) → CloudWatch health check |

### GitHub Secrets Required

| Secret | Purpose |
|---|---|
| AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY | IAM credentials — ECR push + ECS deploy |
| AWS_REGION | ap-east-1 |
| ECR_REGISTRY | `*.dkr.ecr.ap-east-1.amazonaws.com` |
| SQL_CONNECTION_STRING | RDS SQL Server connection string |
| MONGODB_URI | MongoDB connection URI |
| REDIS_CONNECTION | ElastiCache Redis endpoint |
| S3_BUCKET_NAME | Primary media bucket |
| JWT_SECRET_KEY | 256-bit JWT signing key |
| OPENSEARCH_ENDPOINT | OpenSearch domain endpoint |

### GitHub Environments

| Environment | Branch | Auto-deploy | Approval |
|---|---|---|---|
| staging | develop | Yes | None |
| production | main | No | Required (manual gate) |
| demo | cms-demo/main | Yes | None |

---

## Demo Project

### Purpose
Standalone public portfolio showcase at `demo.cms.yourdomain.com`.  
Separate repo (`cms-demo`), separate AWS stack, same codebase with demo flag.

### Demo vs Production Differences

| Aspect | Production | Demo |
|---|---|---|
| Repo | `cms` (private) | `cms-demo` (public) |
| Data | Real business data | Seeded fake Chinese/English data |
| Auth | Staff accounts | Guest login — `demo@cms-showcase.com` / `demo1234` |
| Data reset | Never | Every 24 hours |
| Destructive actions | Enabled | Disabled (403 with friendly message) |
| Instance size | Production-grade | Minimal/cheap |

### Demo Mode Flag

```bash
IS_DEMO_MODE=true
```

### Restricted Actions in Demo Mode

- Delete product / bulk delete
- Delete orders
- Change admin passwords
- Clear all data
- Send real emails / notifications

### Demo AWS Sizing

| Service | Demo |
|---|---|
| ECS | t3.small |
| RDS | db.t3.medium, single-AZ |
| Redis | cache.t3.micro |
| OpenSearch | t3.small.search ×1 |
| Estimated cost | ~$80–120/month |

---

## Local Development

### Prerequisites

- .NET 9 SDK
- Node.js 20+
- Docker Desktop
- Git

### Start local infrastructure

```bash
docker compose up -d
```

Services started:

| Service | Port |
|---|---|
| SQL Server 2022 | 1433 |
| MongoDB | 27017 |
| Redis | 6379 |
| OpenSearch | 9200 |

### Start backend

```bash
cd src/backend
dotnet ef database update --project Cms.Infrastructure --startup-project Cms.Api
dotnet run --project Cms.Api
# API running at http://localhost:5000
# Swagger at http://localhost:5000/swagger
```

### Start frontend

```bash
cd src/frontend
npm install
npm run dev
# Running at http://localhost:3000
```

---

## Environment Variables

### Backend (.env / appsettings.Development.json)

```bash
ASPNETCORE_ENVIRONMENT=Development
CONNECTION_STRING=Server=localhost,1433;Database=CmsDb;User=sa;Password=Dev@12345!;TrustServerCertificate=True
MONGODB_URI=mongodb://localhost:27017
REDIS_CONNECTION=localhost:6379
OPENSEARCH_ENDPOINT=http://localhost:9200
JWT_SECRET_KEY=your-256-bit-secret-key-change-this
S3_BUCKET_NAME=cms-dev-media
AWS_REGION=ap-east-1
IS_DEMO_MODE=false
```

### Frontend (.env.local)

```bash
NEXT_PUBLIC_API_URL=http://localhost:5000
```

---

## Security

| Area | Approach |
|---|---|
| Authentication | ASP.NET Core Identity + JWT Bearer. 15-min access tokens + rotating refresh tokens in Redis. |
| Authorization | Role-based: Admin / Manager / Viewer. `[Authorize(Policy=...)]` on all endpoints. |
| API Security | Rate limiting (ASP.NET Core built-in), CORS restricted to frontend domain, HTTPS at ALB. |
| SQL Injection | EF Core parameterizes all queries. Raw SQL strings prohibited. |
| S3 Security | All objects private. Pre-signed URLs (15 min). Bucket policy blocks public access. |
| Secrets | AWS Secrets Manager (ap-east-1). Auto-rotation every 30 days. |
| Audit Logging | All mutations logged to MongoDB `audit_events`. Immutable, indexed by userId + entity. |
| PIPL | Document PII types, apply minimum-necessary principle, support deletion requests. |

---

## GitHub Integration

### CodeRabbit (AI PR Reviews)
Install at coderabbit.ai — automatic PR review comments on every pull request. No config needed.

### Custom Workflow — Project-Specific Checks
Planned GitHub Actions workflow calling Claude API to enforce:

- All new DB string columns use `NVARCHAR` not `VARCHAR`
- New API endpoints have `[Authorize]` attribute
- New EF entities with Chinese text have a corresponding `NamePinyin` field
- No frontend imports from Google-hosted or GFW-blocked domains

---

## Roadmap

| Phase | Description | Status |
|---|---|---|
| 0 — Architecture | Tech stack, hosting, repo strategy, Chinese string handling | ✅ Done |
| 1 — Repo Setup | GitHub repos, monorepo scaffold, .NET solution, Next.js init | ✅ Done |
| 2 — UI Design | Design all screens — dashboard, inventory, orders, suppliers, admin | 🔄 Next |
| 3 — API Design | OpenAPI contracts derived from UI — request/response shapes | ⏳ Pending |
| 4 — Backend | Domain → EF migrations → Application handlers → API endpoints | ⏳ Pending |
| 5 — Frontend Integration | Connect Next.js to live API, auth flow, Chinese/Pinyin search | ⏳ Pending |
| 6 — AWS Deployment | ECS ap-east-1, RDS Multi-AZ, S3 replication, GitHub Actions | ⏳ Pending |
| 7 — Demo Launch | cms-demo public repo, seeded data, guest login, portfolio page | ⏳ Pending |
| 8 — Production | Domain + HTTPS + monitoring + load testing | ⏳ Pending |

---

## Current Status

```
✅ Architecture finalised
✅ Hosting decision — AWS ap-east-1 Hong Kong, no ICP required
✅ PDF architecture document generated
✅ GitHub repos created — cms (private) + cms-demo (public)
✅ Monorepo scaffolded — .NET 9 Clean Architecture + Next.js 14 + shadcn/ui Nova
✅ develop branch created and pushed on both repos
✅ PROJECT.md created

🔄 Next: Frontend UI design
```

---

*Last updated: April 2026*  
*Maintained by: Development Team*
