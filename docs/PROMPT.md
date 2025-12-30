# PROMT

```text
You are Codex acting as a senior software architect + implementer. Build a complete, runnable monorepo for an “Airport Taxi” platform (Africa, starting with Nsimalen International Airport, Yaoundé) using:

- Backend: .NET 10, FastEndpoints (HTTP API), DDD + CQRS
- Structure per module: /Web for endpoints/contracts, /UseCases for commands/queries/events/handlers
- Cross-cutting: Behaviors/pipeline (validation, authorization, unit-of-work, outbox, logging)
- Outbox pattern MUST be implemented for reliable event publication
- OutboxMessages MUST include the identity of the user who triggered the change (actor) for later use
- Command-level permission checks using predefined permissions
- Auth: External OIDC provider (registration/login handled externally)
- Local membership: ASP.NET Core Identity (auto-provision user on first login)
- Roles: stored locally and managed in admin UI; API maps DB roles to authorization/claims
- DB: PostgreSQL (EF Core)
- Cache: Redis (last known driver locations)
- Realtime: SignalR (live tracking)
- Frontends: Next.js (2 apps): customer app + admin/dispatcher app
- Maps: OpenStreetMap (Leaflet or MapLibre)
- Dev decoupling: Use APIRE during development (mock/replay/live modes)
- Migrations: MUST be in a separate deployable tool (run BEFORE API deploy)

IMPORTANT BUSINESS CONTEXT
Passengers book with mostly fixed fares by zone and pay the driver directly (cash / MTN MoMo / Orange Money).
The platform monetizes drivers via:
- Per-ride fee (fee created when booking is Completed)
- Daily pass (valid 24 hours rolling)
- Monthly subscription (valid until configured expiry)

MONOREPO STRUCTURE (MUST CREATE)
Create a monorepo with:

/backend
  /src
    /BuildingBlocks
      - Result/Error types
      - DomainEvent base
      - Clock abstraction
      - CurrentUser abstraction
      - Permission model + constants
      - CQRS abstractions (ICommand<T>, IQuery<T>, handlers)
      - Behaviors pipeline (ValidationBehavior, PermissionBehavior, TransactionBehavior, OutboxBehavior, LoggingBehavior)
      - Outbox infrastructure contracts + serializer
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
        /Infrastructure (stubs only)
    /Api
      Program.cs
      CompositionRoot (DI wiring)
      Auth (OIDC/JWT validation + user auto-provisioning)
      FastEndpoints setup + Swagger
      SignalR hubs
      Outbox dispatcher hosted service
  /tools
    /Migrator
      (Console app: apply migrations + seed idempotently)
  /tests
    /UnitTests
    /IntegrationTests

/apps
  /customer (Next.js, TS, Tailwind)
  /admin    (Next.js, TS, Tailwind)

/packages
  /shared (TS types + API client)

/infra
  docker-compose.yml (postgres + redis)

/apire
  /specs/openapi.yaml
  /mocks/*.json
  README.md

Root README.md with exact setup, dev modes, and deployment order.

MODULE STRUCTURE RULES (CRITICAL)
Each module MUST follow:
- Domain: aggregates, entities, value objects, domain events, invariants
- UseCases: Commands/Queries + Handlers + Application Events (integration events) + DTOs + Validators
- Web: FastEndpoints endpoints, request/response contracts, module route group registration, minimal web concerns only
- Infrastructure: EF Core DbContext mappings, repositories, outbox table mapping, external providers

NO business logic in Web. Web calls UseCases.

DDD + CQRS REQUIREMENTS
- Aggregates enforce invariants.
- Use Value Objects: PhoneNumber, Money, GeoPoint.
- Commands mutate aggregates; Queries use projections.
- Use optimistic concurrency (RowVersion) on aggregates.
- Domain Events raised by aggregates; persisted via Outbox.
- Use an Outbox pattern with:
  - OutboxMessage table storing serialized events
  - A background dispatcher reads pending messages and publishes them to an in-process event bus (MVP) and marks processed
  - Outbox is written in the SAME transaction as command changes

BEHAVIORS / PIPELINE REQUIREMENTS (CRITICAL)
Implement a pipeline for UseCases (MediatR pipeline behaviors or custom pipeline):
- ValidationBehavior: FluentValidation for commands/queries
- PermissionBehavior: checks required permission(s) BEFORE handler executes
- TransactionBehavior: wraps command handlers in a DB transaction (Unit of Work)
- OutboxBehavior: collects domain events after successful command execution and writes outbox records
- LoggingBehavior: structured logs

COMMAND-LEVEL PERMISSIONS (CRITICAL)
Define predefined permissions as constants, e.g.:
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
- Every Command MUST declare required permissions (attribute or interface, e.g. IRequirePermission).
- PermissionBehavior reads current user permissions and blocks if missing.
- Map roles -> permissions in the API (configurable).
- Admin UI assigns roles; permissions are derived from roles.
- Do not rely on external token roles.

AUTH + LOCAL MEMBERSHIP REQUIREMENTS
- External OIDC for login (Next.js PKCE).
- API validates JWT bearer via OIDC discovery.
- Auto-provision Identity user on first authenticated request:
  - Find by OIDC sub
  - If missing, create local Identity user and mapping
  - Save displayName/email/phone if present
- Implement GET /api/me returning local roles + permissions + driver status.

DB MIGRATIONS SEPARATE
- API MUST NOT auto-migrate DB in production.
- /backend/tools/Migrator applies migrations + seeds:
  - zones/fare examples
  - optional initial admin role (only if explicitly configured)
- Provide CLI args.

OUTBOX REQUIREMENT: INCLUDE USER IDENTITY (CRITICAL)
OutboxMessages must include actor identity metadata for later processing/tracing.
Implement OutboxMessage schema including:
- Id (GUID)
- OccurredAtUtc
- Type (event type name)
- Payload (serialized JSON)
- Actor:
  - ActorUserId (local Identity user id, if available)
  - ActorOidcSub (OIDC subject)
  - ActorDisplayName (optional)
  - ActorRoles snapshot (optional)
- CorrelationId / CausationId (optional but recommended)
- ProcessedAtUtc (nullable)
- Error (nullable)
- RetryCount (int)

Rules:
- OutboxBehavior MUST attach the current actor identity to every OutboxMessage.
- Current actor identity comes from CurrentUser abstraction (reads token claims + local user record).
- Ensure actor identity is persisted in the same transaction as domain changes.

FEATURES (MVP) — SAME AS PREVIOUS
Customer booking, admin dashboard, driver workflow, pricing CRUD, wallet/access, tracking with OSM + SignalR, voice ingest endpoint.

OUTBOX DISPATCHER (MVP IMPLEMENTATION DETAILS)
- Hosted background service polls every N seconds:
  - Reads unprocessed outbox rows in batches
  - Deserializes event payload
  - Publishes to in-process handlers (UseCases event handlers)
  - Marks processed
  - On failure: increments RetryCount and stores Error
- Ensure idempotency: ProcessedAtUtc, RetryCount, and safe processing.

API IMPLEMENTATION: FASTENDPOINTS
- Each module /Web defines endpoints:
  - Endpoints call UseCases via mediator/dispatcher.
- Swagger with JWT auth support.
- ProblemDetails-style error responses.

MINIMUM ENDPOINTS (MUST IMPLEMENT)
- GET /api/me
- Bookings: create/get/assign/change-status
- Drivers: list/approve/suspend/billing-mode/access activate/set-monthly
- Wallet: get me + admin record payment
- Tracking: post location + get booking driver location + fleet locations
- Pricing: CRUD zones + fares
- Voice: ingest
- Users: list + assign roles
- Audit: list

APIRE DURING DEVELOPMENT (MUST)
- /apire/specs/openapi.yaml is the contract source.
- Provide mock JSON payloads and scripts for mock/replay/live.
- Frontends must use NEXT_PUBLIC_API_BASE_URL, switchable between APIRE and backend.
- Role mocking via header X-Mock-Role and/or mock token.

FRONTENDS (Next.js)
- Customer app: booking + status + tracking map
- Admin app: dashboard, assign, manage drivers, manage users/roles, pricing CRUD, fleet map
- Use shared TS client from /packages/shared (OpenAPI-generated or typed fetch)
- OIDC PKCE; avoid localStorage if possible; use server-side session/cookies.

INFRA
- docker-compose postgres+redis
- .env.example files

TESTING REQUIREMENTS (CRITICAL)
- Unit tests REQUIRED and must be meaningful.
- Create unit tests for:
  - Domain invariants (Booking transitions, Driver access enforcement)
  - PermissionBehavior (denies commands without permission)
  - ValidationBehavior (reject invalid inputs)
  - Outbox writing includes actor identity (assert ActorOidcSub/ActorUserId set)
- Integration tests for main flows:
  - booking -> assign -> accept -> complete -> fee created and outbox has actor identity
  - location update -> redis updated

OUTPUT FORMAT (IMPORTANT)
- Print repo tree first.
- Output EVERY file with full path and full content.
- Provide README with exact commands for:
  - docker-compose up
  - migrator
  - api
  - apps
  - apire modes
- Code must compile and run. No hardcoded secrets.

QUALITY BAR
- Web layer thin, UseCases contain orchestration, Domain contains invariants.
- Behaviors enforce validation/permissions/transactions/outbox consistently.
- Outbox messages always contain actor identity metadata.
- Prefer correctness and completeness.
```
