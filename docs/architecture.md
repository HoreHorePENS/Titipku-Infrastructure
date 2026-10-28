# Titipku Architecture

This document describes the runtime and code architecture of the Titipku
backend. The deployable unit is **`core-service`**, a Go modular monolith.

## System context

```text
 Owner web app (PUBLIC_WEB_URL)
        |  Bearer JWT (HS256)
        v
 +------------------+          +----------------------+
 |    core-service  |--------->|   PostgreSQL 17      |
 |   Go 1.26, :8080 |  pgx     |  schema + functions  |
 +------------------+          +----------------------+
        ^
        |  no auth (token link, rate limited)
        v
 Warung partner browser -> /i/{public_token} -> public invoice page
```

- One service, one database (`titipku`).
- Every tenant (business) is isolated by `business_id` carried in the JWT
  claims; repositories always filter by it.
- The only unauthenticated application route is the public invoice view,
  protected by an unguessable token plus a per-IP rate limit.

## Service layout

```text
Titipku-Backend
├── Dockerfile                      # multi-stage build, ARG SERVICE selects the binary
├── internal/platform/              # reusable technical building blocks
│   ├── apperror/                   # error kinds -> HTTP status + stable codes
│   ├── config/                     # env parsing
│   ├── database/                   # pgx pool wiring
│   ├── health/                     # /healthz and /readyz registration
│   ├── httpserver/                 # server lifecycle, graceful shutdown
│   ├── httpx/                      # router, middleware, JSON envelope helpers
│   ├── identity/                   # JWT manager, principal, password hashing, roles
│   └── logger/                     # slog setup
└── services/core-service
    ├── cmd/server/                 # composition root: config -> infra -> app -> HTTP
    └── internal/
        ├── domain/                 # entities, validation, invariants
        ├── usecase/                # workflows + ports (Store, PasswordHasher, TokenIssuer)
        ├── adapter/
        │   ├── httpapi/            # handlers, DTOs, router
        │   └── repository/postgres/# pgx repositories + PG function calls
        └── migrations/             # embedded SQL migrations (run at startup)
```

Dependency rule: `cmd -> adapter -> usecase -> domain`. The usecase layer
defines the ports it needs (`usecase.Store`); the PostgreSQL adapter
implements them. Domain code imports nothing from other layers.

The `auth-service`, `user-service`, and `order-service` directories are the
original marketplace scaffold. They are kept for reference and are **not**
part of the deployed stack.

## Layer responsibilities

| Layer | Contains | Must not contain |
| --- | --- | --- |
| `domain` | Entities, validation helpers, sentinel errors | I/O, SQL, HTTP |
| `usecase` | Workflows, input validation, port interfaces | SQL, HTTP types |
| `adapter/httpapi` | Request decoding, DTO mapping, status codes | Business rules |
| `adapter/repository/postgres` | Queries, PG function calls, error mapping | HTTP concerns |
| `cmd` | Wiring only | Logic |

## Request lifecycle

1. Middleware chain (in order): `RequestID` -> CORS -> `Recoverer` ->
   `SecurityHeaders` -> `AccessLog` -> route handler.
2. `httpx.Handle` wraps every handler; returned errors are translated by
   `apperror.From` into `{ "error": { code, message, fields } }` plus the
   matching HTTP status.
3. Protected routes add `Authenticate` (JWT verification) and
   `RequireRoles(owner)`. Handlers then resolve the business scope
   (`business_id`, `member_id`) from the principal.
4. Success responses are written as `{ "data": ..., "meta"?: ...,
   "request_id": ... }`.

### Error mapping

| Domain error | HTTP | Code |
| --- | --- | --- |
| validation failure | 400 | `VALIDATION_ERROR` (with `fields`) |
| bad credentials | 401 | `INVALID_CREDENTIALS` |
| missing/expired token, missing business scope | 401 | `UNAUTHORIZED` / `BUSINESS_SCOPE_REQUIRED` |
| non-owner role | 403 | `FORBIDDEN` |
| `ErrNotFound` | 404 | `NOT_FOUND` |
| `ErrEmailTaken` | 409 | `EMAIL_TAKEN` |
| `ErrWhatsAppTaken` | 409 | `WHATSAPP_TAKEN` |
| `ErrIdempotencyMismatch` | 409 | `IDEMPOTENCY_MISMATCH` |
| `ErrStockMismatch` | 409 | `STOCK_MISMATCH` |
| `ErrPaymentExceedsReceivable` | 409 | `PAYMENT_EXCEEDS_RECEIVABLE` |
| `ErrPaymentAllocation` | 409 | `PAYMENT_ALLOCATION_FAILED` |
| `ErrInconsistentData` | 409 | `INCONSISTENT_DATA` |
| public rate limit exceeded | 429 | `RATE_LIMITED` |
| dependency down (`/readyz`) | 503 | `DEPENDENCY_UNAVAILABLE` |
| anything else | 500 | `INTERNAL_ERROR` (cause logged, never returned) |

## Consistency model: PostgreSQL functions

The critical write flows run entirely inside PostgreSQL functions within a
single transaction, so stock, invoices, and payments can never diverge.

### `record_visit(business_id, member_id, payload jsonb, request_hash)`

1. **Idempotency**: look up `(business_id, client_request_id)`. If found and
   the stored `request_hash` matches, return the existing visit/invoice
   (`duplicate = true`); if the hash differs, raise `IDEMPOTENCY_MISMATCH`.
2. Lock the member (must be active) and the business (`visit_sequence`
   `FOR UPDATE`), then the partner (`FOR UPDATE`).
3. For each item: re-read the product (active, `FOR SHARE`) for the name,
   consign price, and shelf life; compute `prev_stock` as the sum of open
   batch quantities; derive `sold = prev_stock - remaining - returned`
   (negative -> `STOCK_MISMATCH`).
4. Consume stock **FIFO** across `batches` ordered by
   `dropped_at, created_at, id` for `sold + returned` units; the remaining
   sum must equal `remaining + new_drop`, otherwise `STOCK_MISMATCH`.
5. `new_drop > 0` inserts a new batch with
   `expires_at = dropped_at + shelf_life_days`.
6. Insert `visit_items` with the re-read unit price and
   `subtotal = sold * unit_price`; the invoice total is their sum.
7. If the total is positive, insert the invoice (`UNIQUE (visit_id)`).
8. If an embedded payment with `amount > 0` is present, delegate to
   `record_payment` inside the same transaction.
9. Visit number: `N-YYMMDD-SSSS` where the date is the visit date in
   Asia/Jakarta and `SSSS` is the business sequence zero-padded to 4.

### `record_payment(business_id, member_id, payload jsonb, request_hash)`

Same idempotency pre-check, then: lock the active partner, validate the
amount and method, compute the receivable
(`SUM(total - paid) WHERE paid < total`), reject overpayment with
`PAYMENT_EXCEEDS_RECEIVABLE`, insert the payment, and allocate the amount to
the partner's oldest unpaid invoices (`ORDER BY issued_at, id`), inserting a
`payment_allocations` row per touched invoice. Leftover allocation raises
`PAYMENT_ALLOCATION_FAILED`.

### `void_payment(business_id, member_id, payment_id)`

Locks the payment, reverses each allocation from `invoices.paid`, and marks
the payment `is_void = TRUE` with `voided_at`/`voided_by`. Voiding is
idempotent and never deletes rows, so history is preserved.

### Derived state

- `invoices.status` is a stored generated column:
  `paid = 0 -> unpaid`, `paid < total -> partial`, else `paid`.
- `partners.receivable` is computed on read from open invoices.
- `updated_at` is maintained by the `titipku_set_updated_at` trigger on all
  mutable tables.

## Authentication and authorization

- JWT HS256, issuer `titipku` (configurable), TTL `24h` (configurable),
  30s leeway, expiry required.
- Claims: `sub` (user id), `email`, `name`, `role`, `business_id`,
  `member_id`, `iss`, `iat`, `nbf`, `exp`.
- All `/v1` routes except `auth/register`, `auth/login`, and the public
  invoice route require role `owner` and a business-scoped token.
- Passwords are hashed with bcrypt (Go's `identity` package, 8-72 chars).

## Idempotency and retry safety

Mutating flows (`POST /v1/visits`, `POST /v1/payments`, and the embedded
visit payment) accept a client-generated `client_request_id` (UUID):

- Unique per business, enforced by a unique index.
- A SHA-256 `request_hash` of the canonical payload is stored; replays with
  the same key **and** the same payload return the original result, while a
  different payload returns `409 IDEMPOTENCY_MISMATCH`.

## Cross-cutting HTTP behaviour

- `X-Request-ID`: accepted from the client (max 128 chars) or generated,
  always echoed in the response header and in the JSON envelope.
- CORS: explicit allow-list from `CORS_ALLOWED_ORIGINS`; preflight cached
  600s.
- Security headers: `X-Content-Type-Options: nosniff`,
  `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`.
- Body limits: 1 MiB, single JSON object, unknown fields rejected.
- Pagination: `limit` (1-100, default 20) and `offset`
  (0-100000, default 0) with `meta.total`.
- Public invoice rate limit: in-memory per-IP token bucket,
  `PUBLIC_RATE_LIMIT_PER_MINUTE` (default 60, `0` disables), responds
  `429` with `Retry-After`.

## Data ownership rules

- Every table carries `business_id`; all queries filter by it.
- A user can hold at most one active membership
  (`members_active_user_unique` partial unique index).
- Partners are unique per business by WhatsApp number.
- Visits and invoices reference partners with `ON DELETE RESTRICT`, so
  historical financial records survive partner deactivation.

## Configuration

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `APP_ENV` | No | `development` | Log verbosity and environment label |
| `HTTP_PORT` | No | `8080` | HTTP listen port |
| `DATABASE_URL` | Yes | - | PostgreSQL connection string |
| `JWT_SECRET` | Yes | - | HMAC secret, at least 32 characters |
| `JWT_ISSUER` | No | `titipku` | Expected JWT issuer |
| `JWT_TTL` | No | `24h` | Access token lifetime |
| `HTTP_SHUTDOWN_TIMEOUT` | No | `10s` | Graceful shutdown deadline |
| `PUBLIC_WEB_URL` | No | `http://localhost:3000` | Frontend base URL for invoice links (`{PUBLIC_WEB_URL}/i/{token}`) |
| `CORS_ALLOWED_ORIGINS` | No | `http://localhost:3000` | Comma-separated browser origins |
| `PUBLIC_RATE_LIMIT_PER_MINUTE` | No | `60` | Per-IP public invoice limit; `0` disables it |

## Deployment topology (local, Docker Compose)

- `postgres`: PostgreSQL 17 with a healthcheck; data in a named volume.
- `core-service`: built from the root `Dockerfile` with
  `--build-arg SERVICE=core-service`; runs as a non-root user; migrations
  run automatically at startup; `depends_on` healthy Postgres; exposes
  `8080` with `/healthz`/` readiness` checks.

Production notes: terminate TLS at a reverse proxy or load balancer, inject
`JWT_SECRET` from a secret manager, run at least two `core-service`
replicas (stateless; the public rate limiter is per-instance and in-memory),
and back up PostgreSQL with PITR.

## API contract

The full HTTP contract lives in
[`specs/openapi/core-service.yaml`](../specs/openapi/core-service.yaml).
The ERD is documented in [`docs/erd.md`](erd.md).
