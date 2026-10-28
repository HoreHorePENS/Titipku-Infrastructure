# Titipku ERD

Entity-relationship diagram for the `titipku` PostgreSQL 17 schema, generated
from `services/core-service/internal/migrations/0001_create_core_schema.sql`.

## Diagram

```mermaid
erDiagram
    users ||--o{ members : "belongs to (via user_id)"
    businesses ||--o{ members : "has"
    businesses ||--o{ products : "owns"
    businesses ||--o{ partners : "owns"
    businesses ||--o{ visits : "issues"
    businesses ||--o{ batches : "tracks"
    businesses ||--o{ invoices : "issues"
    businesses ||--o{ payments : "receives"

    partners ||--o{ visits : "receives"
    members ||--o{ visits : "records"
    partners ||--o{ batches : "holds stock at"
    products ||--o{ batches : "stocked as"
    visits ||--o{ batches : "drops stock in"
    visits ||--o{ visit_items : "contains"
    products ||--o{ visit_items : "sold as"
    visits ||||--o| invoices : "bills (1:0..1)"
    partners ||--o{ invoices : "owes"
    partners ||--o{ payments : "pays"
    members ||--o{ payments : "records"
    members ||--o{ payments : "voids (voided_by)"
    payments ||--o{ payment_allocations : "allocated to"
    invoices ||--o{ payment_allocations : "settled by"

    users {
        uuid id PK
        text email UK "unique on lower(email)"
        text full_name
        text password_hash "bcrypt"
        text role "CHECK: owner | admin"
        timestamptz created_at
        timestamptz updated_at
    }

    businesses {
        uuid id PK
        text name "2-120 chars"
        text owner_name "2-100 chars"
        text whatsapp "nullable, normalized 62..."
        text address "nullable"
        bigint visit_sequence "counter for invoice numbering"
        timestamptz created_at
        timestamptz updated_at
    }

    members {
        uuid id PK
        uuid business_id FK
        uuid user_id FK
        text role "CHECK: owner | staff"
        boolean is_active "one active membership per user"
        timestamptz created_at
        timestamptz updated_at
    }

    products {
        uuid id PK
        uuid business_id FK
        text name "1-120 chars"
        text photo_path "nullable"
        bigint consign_price ">= 0, unit price at visits"
        bigint retail_price ">= 0"
        integer shelf_life_days "1-3650, drives batch expiry"
        boolean is_active
        timestamptz created_at
        timestamptz updated_at
    }

    partners {
        uuid id PK
        uuid business_id FK
        text name "2-120 chars"
        text owner_name "2-100 chars"
        text whatsapp "62..., unique per business"
        text address "nullable"
        boolean is_active
        timestamptz created_at
        timestamptz updated_at
    }

    visits {
        uuid id PK
        uuid business_id FK
        uuid partner_id FK "RESTRICT"
        uuid member_id FK "RESTRICT"
        text number "N-YYMMDD-SSSS, unique per business"
        uuid client_request_id "idempotency key, unique per business"
        text request_hash "sha256 of canonical payload"
        text public_token "UK, unguessable invoice share token"
        timestamptz visited_at
        text received_by
        timestamptz created_at
        timestamptz updated_at
    }

    visit_items {
        uuid id PK
        uuid business_id FK
        uuid visit_id FK
        uuid product_id FK "RESTRICT"
        text product_name "denormalized at visit time"
        bigint prev_stock "sum of open batches before visit"
        bigint remaining "counted stock left"
        bigint returned "units returned"
        bigint new_drop "fresh units dropped"
        bigint sold "computed: prev - remaining - returned"
        bigint stock_after "computed: remaining + new_drop"
        bigint unit_price "consign price re-read at visit time"
        bigint subtotal "sold * unit_price"
        timestamptz created_at
    }

    batches {
        uuid id PK
        uuid business_id FK
        uuid partner_id FK "RESTRICT"
        uuid product_id FK "RESTRICT"
        uuid visit_id FK "the drop that created the batch"
        date dropped_at "FIFO order key"
        date expires_at "dropped_at + shelf_life_days"
        bigint qty_in "> 0"
        bigint qty_left "0..qty_in, open stock at partner"
        timestamptz created_at
    }

    invoices {
        uuid id PK
        uuid business_id FK
        uuid partner_id FK "RESTRICT"
        uuid visit_id FK "UNIQUE, 1:1 with visit"
        bigint total "> 0"
        bigint paid "0..total"
        text status "generated: unpaid | partial | paid"
        timestamptz issued_at
        timestamptz created_at
        timestamptz updated_at
    }

    payments {
        uuid id PK
        uuid business_id FK
        uuid partner_id FK "RESTRICT"
        uuid member_id FK "RESTRICT"
        bigint amount "> 0"
        text method "CHECK: cash | transfer | qris | other"
        timestamptz paid_at
        uuid client_request_id "idempotency key, unique per business"
        text request_hash "sha256 of canonical payload"
        boolean is_void
        timestamptz voided_at "nullable"
        uuid voided_by "FK members, nullable"
        timestamptz created_at
        timestamptz updated_at
    }

    payment_allocations {
        uuid id PK
        uuid payment_id FK
        uuid invoice_id FK "RESTRICT"
        bigint amount "> 0"
        timestamptz created_at
    }
```

## Table notes

| Table | Key constraints and indexes |
| --- | --- |
| `users` | `users_email_unique` on `lower(email)` |
| `businesses` | `visit_sequence` is locked `FOR UPDATE` during `record_visit` to generate visit numbers |
| `members` | `UNIQUE (business_id, user_id)`; `members_active_user_unique` partial unique index guarantees one active membership per user |
| `products` | Indexes on `business_id` and `(business_id, is_active)` |
| `partners` | `UNIQUE (business_id, whatsapp)`; WhatsApp CHECK `^62[0-9]{8,15}$` |
| `visits` | `UNIQUE (business_id, number)`, `UNIQUE (business_id, client_request_id)`, `UNIQUE (public_token)` |
| `visit_items` | `UNIQUE (visit_id, product_id)`; all quantities CHECK `>= 0` |
| `batches` | FIFO index `(partner_id, product_id, dropped_at, created_at, id)`; `qty_left <= qty_in` |
| `invoices` | `UNIQUE (visit_id)`; `status` is a stored generated column from `paid` vs `total`; CHECK `paid <= total` |
| `payments` | `UNIQUE (business_id, client_request_id)`; method CHECK enum; void columns keep history |
| `payment_allocations` | `UNIQUE (payment_id, invoice_id)`; partial payments create one row per invoice touched |

## Invariants enforced in the database

- **Stock**: for every `(partner, product)`, the sum of `batches.qty_left`
  equals the partner's current stock. `record_visit` consumes batches FIFO
  for `sold + returned` units and verifies the result equals
  `remaining + new_drop`, raising `STOCK_MISMATCH` otherwise.
- **Invoice totals**: `invoices.total = SUM(visit_items.subtotal)` for the
  owning visit, and an invoice exists only when the total is positive.
- **Payments**: a payment never exceeds the partner's receivable
  (`SUM(total - paid) WHERE paid < total`); allocations always sum to the
  payment amount.
- **Timestamps**: `updated_at` is maintained by the
  `titipku_set_updated_at` trigger on `businesses`, `members`, `products`,
  `partners`, `visits`, `invoices`, and `payments`.
- **Extensions**: `pgcrypto` provides `gen_random_uuid()` and
  `gen_random_bytes()` (public invoice tokens).
