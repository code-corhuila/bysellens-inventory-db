# Inventory database source review

Review date: 2026-11-11. Only `bysellens-inventory-db` is changed. Reference
repositories and the local archive were read without modifying them.

## Sources and precedence

| Source | Reviewed revision | Role |
|---|---|---|
| `BY_SELLENS Copia 3.zip` | Local archive, Inventory source paths below | Legacy implementation evidence |
| [bysellens-docs][docs] | `main` at `05f12ed39c0855b5e8f79ea9e3a552244bb203f7` | Current ownership and domain/API contract |
| [bysellens-inventory-api][api] | `develop` at `9d2889cee517f2c5f644244f8ba5f8ff4068eb01` | Current physical SQL contract |
| `bysellens-inventory-db` | `develop` at `f19ab0863b143edc5a7d05c723687c729fe33231` | Feature branch base |

Documentation determines Inventory ownership; the current API migration determines
the exact DDL delivered here. ZIP differences are retained as pending work, not
silently translated into new tables or imports. Both reference repositories are
consultation-only sources.

## ZIP findings

Paths are relative to `BY_SELLENS Copia/BACKEND/` inside the archive.
Inventory uses PostgreSQL, Flyway and Hibernate schema validation. The shared
Compose selects `postgres:17-alpine`. Inventory's Flyway and Hibernate default
schema is `inventory`, unlike the current API's separate `inventory_db` setup.

| Path | Findings |
|---|---|
| `inventory-service/pom.xml` | PostgreSQL JDBC, Flyway core and PostgreSQL Flyway support |
| `inventory-service/src/main/resources/application.properties` | `inventory` schema; Hibernate `validate` |
| `inventory-service/src/main/resources/db/migration/V1__initial.sql` | Creates `existencias`, `reservas`, `movimientos` |
| `inventory-service/src/main/resources/db/migration/V2__rabbit_outbox.sql` | Creates `rabbit_outbox` |
| `postgres/init.sh` | Shared initializer creates per-service schemas/users; not copied |

| Legacy table | Keys and rules |
|---|---|
| `existencias` | `producto_id BIGINT` PK; non-negative `stock`, `stock_minimo`; `reservado` defaults to 0 and stays between 0 and stock |
| `reservas` | `id VARCHAR(255)` PK; required `detalles JSONB`; `estado` limited to NUEVA, RESERVADA, CONFIRMADA, CANCELADA |
| `movimientos` | Identity BIGINT PK; local FK to `existencias(producto_id)`; quantity, type, optional reference and TIMESTAMPTZ date |
| `rabbit_outbox` | `id VARCHAR(100)` PK; routing key, text payload, created/sent TIMESTAMPTZ; created time defaults to `now()` |

Explicit ZIP indexes are `movimientos_producto_idx` on `(producto_id, id DESC)`
and `rabbit_outbox_pending_idx` on `(created_at) WHERE sent_at IS NULL`.
Primary keys also provide their implicit unique indexes.

No Inventory seed class, seed SQL or separate Inventory database script was found
in the source tree. Inventory's application initialization inserts stock during a
business operation; it is not a seed. Compiled `target/` copies, other service
migrations, snapshots and backups are excluded from the delivery.

## Delivered physical contract

[`db/migration/V1__create_products_and_outbox.sql`](../db/migration/V1__create_products_and_outbox.sql)
matches the [API migration][migration] exactly, with Git blob
`077eb02e5cf2ef6409507f49d697ea69576263ba`. No legacy V1/V2 is placed in its Flyway
location, and no applied migration is rewritten.

| Object | Current definition |
|---|---|
| `products` | UUID PK; required text code/name/category; nullable shade; NUMERIC purchase/sale prices; non-negative INTEGER initial_stock; required created/updated TIMESTAMPTZ and nullable deleted_at |
| Product checks | `purchase_price >= 0`, `sale_price > 0`, `initial_stock >= 0` |
| `uk_products_code` | Unique code across active and logically removed rows |
| `idx_products_active_code_id` | `(code ASC, id ASC) WHERE deleted_at IS NULL`, matching active-list ordering |
| `outbox_events` | UUID event_id PK; required event_type VARCHAR(64), routing_key VARCHAR(128), JSONB payload and created_at; attempts INTEGER defaults to 0 and stays non-negative; nullable published_at |
| `idx_outbox_pending_created` | `(created_at ASC) WHERE published_at IS NULL` |

Both primary keys and the code constraint also create implicit unique indexes.
There are no cross-service foreign keys. Current stock belongs to Product Service;
`initial_stock` records only the immutable registration quantity. The outbox is
Inventory's technical storage for local transactional publication, already present
in the current API; general docs leave its concrete schema unspecified.

## Pending differences and decisions

| Pending item | Evidence and required follow-up |
|---|---|
| Legacy ownership | ZIP Inventory owns stock, reservations and movements; current [ownership][ownership] assigns Stock to Product. Keep these legacy objects outside Inventory DB; review disposition separately. |
| Legacy identifiers and data | ZIP uses numeric product IDs; current [model][model] uses UUID. A reviewed identifier mapping, reference reconciliation and import plan are required before any legacy conversion. No mapping or data is invented here. |
| Database separation | ZIP uses service schemas in a shared PostgreSQL database. Current sources select service-owned databases. Provisioning, privileges and any transition require separate review; the shared initializer is not reused. |
| Minimum stock | ZIP persists `stock_minimo`; current model excludes minimum_stock. Retain as a pending domain/migration decision, not a new Inventory column. |
| Reservations and event flow | ZIP consumes sale confirmations and writes `rabbit_outbox`; current Inventory publishes ProductCreated/ProductUpdated into `outbox_events`. Legacy states, event IDs and text payloads have no approved conversion. |
| Money and optional images | Current model leaves monetary precision/rounding and image representation pending. Preserve unconstrained NUMERIC and omit image columns until reviewed. |
| SQL versus application rules | V1 has no non-blank checks, immutable-stock trigger, generated UUID defaults or timestamp defaults. The API supplies IDs/timestamps, validates text and rejects initialStock updates. Any database hardening needs a new reviewed migration. |
| Documentation consistency | Inventory [service README][service] still describes draft 0.1.0 and no collection endpoint; [OpenAPI][openapi] is 1.0.0 and defines active pagination. The file at [Inventory decisions][decisions] contains a Customer data model. Reference corrections remain pending in their owning repository. |
| Migration execution ownership | API executes its classpath copy; this repository only prepares the matching artifact. Artifact packaging, single execution ownership and any provisioning/recovery procedure need separate integration review. |

## Verification scope

Static checks cover the exact source blob, PostgreSQL syntax parsed with pglast
8.7 (four DDL statements, without execution), expected table/column definitions,
partial-index columns and predicates, whitespace, local Markdown links,
credential patterns, changed-file scope and the 400-line PR limit. The SQL's final
newline behavior is preserved from the authoritative API blob.

No seed records or external service code are included. SQL execution, Flyway
validation against a database, application integration and data migration are
outside this preparation and were not run.

[docs]: https://github.com/code-corhuila/bysellens-docs/tree/05f12ed39c0855b5e8f79ea9e3a552244bb203f7
[api]: https://github.com/code-corhuila/bysellens-inventory-api/tree/9d2889cee517f2c5f644244f8ba5f8ff4068eb01
[migration]: https://github.com/code-corhuila/bysellens-inventory-api/blob/9d2889cee517f2c5f644244f8ba5f8ff4068eb01/src/main/resources/db/migration/V1__create_products_and_outbox.sql
[model]: https://github.com/code-corhuila/bysellens-docs/blob/05f12ed39c0855b5e8f79ea9e3a552244bb203f7/06-data/models.md
[ownership]: https://github.com/code-corhuila/bysellens-docs/blob/05f12ed39c0855b5e8f79ea9e3a552244bb203f7/09-microservices/data-ownership-matrix.md
[service]: https://github.com/code-corhuila/bysellens-docs/blob/05f12ed39c0855b5e8f79ea9e3a552244bb203f7/09-microservices/services/04-inventory-service/README.md
[openapi]: https://github.com/code-corhuila/bysellens-docs/blob/05f12ed39c0855b5e8f79ea9e3a552244bb203f7/07-api/contracts/openapi/inventory-service.yaml
[decisions]: https://github.com/code-corhuila/bysellens-docs/blob/05f12ed39c0855b5e8f79ea9e3a552244bb203f7/09-microservices/services/04-inventory-service/decisions.md
