# HeavyMeta — Claude Code Context

## What this project is
A public-facing MTG community website with card browsing, individual card pages, deck building, deck browsing, and community deck pages. User accounts, comments, and social features are in scope.

## Current status
- Solution scaffolded, all 14 projects build successfully
- React frontend scaffolded with Vite + TypeScript
- **Next step: Identity Service** — domain models, JWT infrastructure, register/login endpoints

## Repo structure
```
src/
  shared/
    HeavyMeta.Shared.Contracts/       # Domain events, shared DTOs, common interfaces
    HeavyMeta.Shared.Messaging/       # Service Bus abstractions, outbox interfaces
  services/
    identity/
      HeavyMeta.Identity.API/         # ASP.NET Web API — auth endpoints
      HeavyMeta.Identity.Core/        # Domain models, interfaces, command/query handlers
      HeavyMeta.Identity.Infrastructure/ # Dapper repos, JWT, password hashing, outbox poller
    catalog/
      HeavyMeta.Catalog.API/          # ASP.NET Web API — card/set endpoints
      HeavyMeta.Catalog.Core/         # Domain models, interfaces, command/query handlers
      HeavyMeta.Catalog.Infrastructure/  # Dapper repos, Redis cache, Search integration
    deck/
      HeavyMeta.Deck.API/             # ASP.NET Web API — deck/comment endpoints
      HeavyMeta.Deck.Core/            # Domain models, interfaces, command/query handlers
      HeavyMeta.Deck.Infrastructure/  # Dapper repos, outbox poller
    sync/
      HeavyMeta.Sync.Worker/          # .NET Worker Service — scheduled Scryfall import
      HeavyMeta.Sync.Core/            # Import pipeline interfaces
      HeavyMeta.Sync.Infrastructure/  # Scryfall HTTP client, JSON streaming, blob storage
  frontend/
    heavy-meta-web/                   # React 18 + TypeScript + Vite
```

## Tech stack
| Layer | Technology |
|---|---|
| Backend services | ASP.NET Core 9, C# |
| Data access | Dapper + raw SQL (no EF Core) |
| Database | Azure SQL / MSSQL (one DB per service) |
| Async messaging | Azure Service Bus |
| Cache | Azure Cache for Redis |
| Search | Azure Cognitive Search |
| Card images / static | Azure Blob Storage + Azure CDN |
| Auth | ASP.NET Core Identity + custom JWT issuance |
| Frontend | React 18, TypeScript, Vite, React Query, Zustand, React Router v6 |
| Hosting | Azure Container Apps (services), Azure Static Web Apps (frontend) |
| Observability | Azure Application Insights, structured logging via ILogger<T> |
| CI/CD | GitHub Actions |

## Architecture patterns in use
- **CQRS** via MediatR in every service (commands write, queries read)
- **Repository pattern** — service-specific interfaces, no generic IRepository<T>
- **Outbox pattern** — `outbox_messages` table in every service DB; background poller publishes to Service Bus
- **Competing consumers** — multiple service instances subscribe to Service Bus topics
- **Database-per-service** — no cross-service joins; data denormalized at write time where needed
- **Cache-aside** — Redis, invalidated via Service Bus events

## Service Bus topics
| Topic | Publisher | Subscriber |
|---|---|---|
| `card.synced` | Sync | Catalog (cache invalidation, search reindex) |
| `deck.published` | Deck | (future notification service) |
| `user.registered` | Identity | Deck (provision user profile) |

## Key data decisions
- `deck_entries` stores `card_name` denormalized from Catalog — avoids cross-service calls on deck reads
- `NEWSEQUENTIALID()` for all primary keys (GUIDs, index-friendly)
- Outbox poller uses READ COMMITTED SNAPSHOT isolation; sync upserts use MERGE statements
- Card search via Azure Cognitive Search (not SQL full-text) — supports fuzzy match and faceted filters

## Identity Service design (implement this next)
- ASP.NET Core Identity for user storage + password hashing
- Custom JWT issuance: 15-minute access tokens, 7-day refresh tokens (HTTP-only cookie, stored in DB)
- Refresh token rotation on every use
- JWT claims: `sub` (userId), `email`, `role` (user/moderator/admin), `jti`
- APIM validates JWT at the gateway; services trust pre-validated claims
- Endpoints: POST /api/v1/auth/register, POST /api/v1/auth/login, POST /api/v1/auth/refresh, POST /api/v1/auth/logout
- Publishes `user.registered` event to outbox on successful registration

## Coding conventions
- Nullable reference types enabled, warnings as errors (see Directory.Build.props)
- Implicit usings enabled
- All handlers take CancellationToken
- DTOs are records (immutable)
- Repository interfaces live in Core; implementations live in Infrastructure
- No EF Core anywhere — Dapper only
- Structured logging: _logger.LogInformation("User {UserId} created deck {DeckId}", userId, deckId)

## Build order
1. ✅ Solution scaffold
2. 🔜 Identity Service (domain → infrastructure → API → tests)
3. Catalog Service (schema → Dapper repos → card browser API → Redis caching)
4. Sync Worker (Scryfall download → JSON streaming → MERGE upsert → Service Bus publish)
5. React frontend (card browser, card detail, auth flows)
6. Deck Service (CQRS with MediatR → outbox → comments)
7. Deck frontend (builder UI, browser, comments)
8. Search (Azure Cognitive Search integration)
9. Hardening (APIM rate limiting, Testcontainers integration tests, App Insights)
