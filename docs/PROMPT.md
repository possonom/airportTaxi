Understood — **drop APIRE بالكامل**. You meant **.NET Aspire** for the dev experience (orchestration, service discovery, env wiring), not API mocking/replay.

Here is the **complete prompt** rewritten with **.NET Aspire (required)** and **no APIRE**.

```text
You are Codex acting as a senior software architect + implementer.

BUILD A COMPLETE RUNNABLE MONOREPO for an “Airport Taxi” platform (Africa; starting with Nsimalen International Airport, Yaoundé) with:

- Backend: .NET 10 + FastEndpoints (HTTP API) + SignalR
- Architecture: DDD + CQRS
- Per module folder structure: /Domain, /UseCases, /Web, /Infrastructure
- Cross-cutting pipeline Behaviors (validation, permission checks, transactions, outbox, logging)
- Outbox pattern REQUIRED; OutboxMessages MUST include actor identity (user info)
- Command-level permissions REQUIRED (predefined permissions checked at command level via pipeline behavior)
- Auth: External OIDC provider (login/registration handled externally)
- Local membership: ASP.NET Core Identity; auto-provision local user on first login
- Roles assigned via admin UI; API derives permissions from roles (do NOT rely on token roles)
- Database: PostgreSQL (EF Core)
- Cache: Redis (store last known driver locations)
- Realtime: SignalR (live tracking)
- Frontends: Next.js apps (2): customer app + admin/dispatcher app (TypeScript + Tailwind)
- Maps: OpenStreetMap via Leaflet or MapLibre
- DB migrations: separate deployable tool “Migrator” to run BEFORE API deploy
- Dev orchestration: .NET Aspire AppHost (required) for local dev, wiring Postgres, Redis, Migrator, API

IMPORTANT BUSINESS CONTEXT
Passengers book with mostly fixed fares by zone and pay the driver directly (cash / MTN MoMo / Orange Money).
Platform monetizes drivers via:
- Per-ride fee (fee created when booking is Completed)
- Daily pass (valid 24h rolling)
- Monthly subscription (valid until configured expiry)

MONOREPO STRUCTURE (MUST CREATE)
Create a monorepo with:

/backend
  /src
    /BuildingBlocks
      - Result/Error types
      - DomainEvent base
      - Clock abstraction
      - CurrentUser abstraction (reads token + local user)
      - Permission model + constants + role->permission mapping
      - CQRS abstractions (ICommand<T>, IQuery<T>, handlers)
      - Behaviors pipeline:
        ValidationBehavior
        PermissionBehavior
        TransactionBehavior
        OutboxBehavior
        LoggingBehavior
      - Outbox contracts + serializer + dispatcher interfaces
    /Modules
      /Identity
        /Domain
        /UseCases
        /Web
        /Infrastructure
      /Bookings
        /Domain
        /UseCases
        /Web
        /Infrastructure
      /Drivers
        /Domain
        /UseCases
        /Web
        /Infrastructure
      /Pricing
        /Domain
        /UseCases
        /Web
        /Infrastructure
      /Wallet
        /Domain
        /UseCases
        /Web
        /Infrastructure
      /Tracking
        /Domain
        /UseCases
        /Web
        /Infrastructure
      /Notifications
        /UseCases
        /Infrastructure (stubs only: SMS/WhatsApp/Telegram/Email)
    /Api
      Program.cs (FastEndpoints + Swagger + Auth + DI)
      Hubs (SignalR hubs)
      HostedServices (Outbox dispatcher)
  /tools
    /Migrator (Console app: apply migrations + seed idempotently)
  /aspire
    /AppHost (Aspire AppHost project)
    /ServiceDefaults (Aspire defaults project)
  /tests
    /UnitTests
    /IntegrationTests

/apps
  /customer (Next.js, TS, Tailwind, OIDC PKCE)
  /admin    (Next.js, TS, Tailwind, OIDC PKCE)

/packages
  /shared (TS types + API client)

/infra
  docker-compose.yml (OPTIONAL fallback; Aspire is primary for dev)

Root README.md with:
- Dev run using Aspire
- Production deploy order: Migrator -> API -> apps
- Env var examples for OIDC and API URLs

MODULE STRUCTURE RULES (CRITICAL)
Each module MUST have:
- Domain: aggregates, entities, value objects, domain events, invariants
- UseCases: Commands/Queries + Handlers + DTOs + Validators + internal event handlers
- Web: FastEndpoints endpoints + request/response contracts only
- Infrastructure: EF Core mapping, repositories, outbox mapping, external providers

No business logic in Web.

DDD + CQRS REQUIREMENTS
- Use Value Objects: PhoneNumber, Money, GeoPoint.
- Aggregate invariants enforced in Domain.
- Commands mutate aggregates; Queries return read models via EF projections.
- Optimistic concurrency (RowVersion) on aggregates.
- Domain events raised by aggregates and persisted via outbox in same transaction.

BEHAVIORS / PIPELINE (CRITICAL)
Implement a pipeline for UseCases (MediatR pipeline behaviors or custom pipeline):
- ValidationBehavior uses FluentValidation
- PermissionBehavior enforces predefined permissions for every command
- TransactionBehavior ensures a single DB transaction for a command
- OutboxBehavior persists domain events to outbox (same transaction) including actor identity
- LoggingBehavior logs command execution (structured)

PERMISSIONS (CRITICAL)
Define constants:
- Permissions.Bookings.Create
- Permissions.Bookings.Assign
- Permissions.Bookings.ChangeStatus
- Permissions.Drivers.Approve
- Permissions.Drivers.Suspend
- Permissions.Drivers.SetBillingMode
- Permissions.Drivers.ActivateDailyPass
- Permissions.Pricing.Manage
- Permissions.Users.ManageRoles
- Permissions.Tracking.ViewFleet
- Permissions.Audit.View

Rules:
- Every Command declares required permissions (e.g., IRequirePermissions { string[] Permissions }).
- PermissionBehavior checks CurrentUser permissions (derived from LOCAL roles in DB).
- Do not rely on external token roles.

AUTH + LOCAL MEMBERSHIP (CRITICAL)
- External OIDC provider:
  - Next.js apps use Authorization Code + PKCE
  - API validates JWT bearer via OIDC discovery
- On first authenticated API request:
  - If local user mapping by OIDC sub missing, create Identity user and mapping.
  - Save displayName/email/phone if present.
  - Idempotent.
- GET /api/me returns: profile, roles, permissions, driver status, access validity.

DB MIGRATIONS SEPARATE (CRITICAL)
- API MUST NOT auto-migrate DB in production.
- /backend/tools/Migrator:
  - Applies EF Core migrations + seeds (idempotent)
  - CLI args: --connection "<conn>" --seed true
- Seed at least:
  - Sample zones + fares (FCFA)
  - Minimal system configuration

OUTBOX REQUIREMENT: INCLUDE ACTOR IDENTITY (CRITICAL)
OutboxMessage schema MUST include:
- Id (GUID)
- OccurredAtUtc
- Type
- Payload (JSON)
- ActorUserId (local Identity user id, if available)
- ActorOidcSub (OIDC subject)
- ActorDisplayName (optional)
- ActorRolesSnapshot (optional)
- CorrelationId / CausationId (optional but recommended)
- ProcessedAtUtc (nullable)
- Error (nullable)
- RetryCount (int)

Rules:
- OutboxBehavior must attach actor identity from CurrentUser to every OutboxMessage.
- Outbox records are written in same transaction as the command.

OUTBOX DISPATCHER (MVP)
- Hosted background service in API:
  - Polls DB for unprocessed outbox messages
  - Deserializes payload
  - Publishes to in-process event handlers (UseCases)
  - Marks processed; retries with backoff, stores errors

FEATURES (MVP)

Customer app:
- Create booking (pickup Nsimalen + meeting point, dropoff zone, schedule time, passengers, phone required)
- Show fixed fare and “pay driver directly”
- View booking status
- If assigned: live map tracking (OSM)

Admin/Dispatcher app:
- Dashboard (today’s bookings, unassigned, active drivers)
- Manual assignment/reassignment
- Drivers management (approve/suspend, set billing mode, activate daily pass, set monthly expiry)
- Users/roles management (assign Admin/Dispatcher/Driver)
- Pricing CRUD (zones/fare)
- Fleet map (all online drivers)
- Audit log view

Driver UI (can be a section in admin app):
- Profile
- Online/Offline toggle
- Accept/decline booking
- Update booking status (Assigned -> Accepted -> OnTheWay -> PickedUp -> Completed/Cancelled)
- Wallet/access view
- Send location periodically

Monetization rules:
- BillingMode: PerRide | DailyPass | Monthly
- DailyPass: validUntil = now + 24 hours rolling
- Monthly: validUntil = configured date
- PerRide: fee transaction created when booking completed
- Enforcement: cannot accept new bookings if access expired OR wallet negative
- Accepted bookings can be completed even if access expires mid-trip (default TRUE)

Tracking rules:
- Driver POST /api/drivers/me/location
- Redis stores last known location
- SignalR groups:
  - booking:{bookingId} -> customer map
  - admin:fleet -> fleet map

Voice booking endpoint (for n8n):
- POST /api/voice-bookings/ingest (transcript + extracted fields + confidence)
- Create Draft booking if missing fields; else Created

FASTENDPOINTS API (MUST)
- Implement endpoints in each module’s /Web
- Use FastEndpoints conventions; add Swagger auth
- Use ProblemDetails-style error responses

MINIMUM ENDPOINTS (MUST IMPLEMENT)
- GET /api/me
- POST /api/bookings
- GET /api/bookings/{id}
- POST /api/bookings/{id}/assign
- POST /api/bookings/{id}/status
- GET /api/drivers
- POST /api/drivers/{id}/approve
- POST /api/drivers/{id}/suspend
- POST /api/drivers/{id}/billing-mode
- POST /api/drivers/{id}/access/activate-daily
- POST /api/drivers/{id}/access/set-monthly
- GET /api/wallet/me
- POST /api/wallet/{driverId}/payments (admin records payment)
- POST /api/drivers/me/location
- GET /api/bookings/{id}/driver-location
- GET /api/drivers/locations
- CRUD /api/zones
- CRUD /api/fares
- POST /api/voice-bookings/ingest
- GET /api/admin/users
- POST /api/admin/users/{userId}/roles
- GET /api/audit

.NET ASPIRE (REQUIRED)
Use .NET Aspire to orchestrate local development:
- Create AppHost that starts:
  - Postgres container resource
  - Redis container resource
  - Migrator (runs once on startup OR via a separate Aspire profile; must be possible to run independently too)
  - API service
- Provide connection strings and env vars via Aspire configuration binding.
- Include /backend/aspire/ServiceDefaults project and apply it to API + Migrator.
- Provide run instructions in README:
  - dotnet run --project backend/aspire/AppHost
  - and how to run Migrator alone for production pipelines

FRONTENDS (Next.js)
- Implement OIDC PKCE login for both apps (admin required; customer optional feature flag)
- Use env vars:
  NEXT_PUBLIC_OIDC_ISSUER
  NEXT_PUBLIC_OIDC_CLIENT_ID
  NEXT_PUBLIC_OIDC_REDIRECT_URI
  NEXT_PUBLIC_API_BASE_URL
- Use shared TS client in /packages/shared
- Customer: booking flow + map tracking
- Admin: dashboard + management UIs + fleet map
- Leaflet OSM map with markers and last-updated timestamps

TESTS (CRITICAL)
Unit tests REQUIRED (meaningful):
- Booking status transition rules (domain)
- Driver access enforcement rules (domain/use cases)
- PermissionBehavior denies when missing permission
- ValidationBehavior works
- OutboxBehavior writes messages containing actor identity (ActorOidcSub/ActorUserId set)
Integration tests:
- booking -> assign -> accept -> complete -> fee created + outbox message with actor identity
- location update stores in Redis

OUTPUT FORMAT (IMPORTANT)
- Print repo tree first.
- Output EVERY file with full path + full content.
- Provide README with exact commands:
  - dotnet run --project backend/aspire/AppHost
  - how to run migrator alone
  - how to run API alone
  - how to run apps
- Code must compile and run.
- No hardcoded secrets; provide .env.example.

QUALITY BAR
- Web thin, UseCases orchestrate, Domain enforces invariants.
- Behaviors enforce validation/permissions/transactions/outbox consistently.
- Outbox always contains actor identity metadata.
- Prefer correctness and completeness over brevity.
```

If you want, I can also add an explicit requirement that **Aspire wires the Next.js apps too** (as external endpoints) — or keep Aspire only for backend services (Postgres/Redis/API/Migrator), which is the most common pattern.
