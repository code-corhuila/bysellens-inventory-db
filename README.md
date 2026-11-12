# bysellens-inventory-db

Inventory-owned PostgreSQL schema and Flyway migrations for By_Sellens.

The current Inventory boundary owns product identity, descriptive information,
prices, immutable registration stock and logical removal. Product Service owns
current stock and stock changes. Legacy reservations are outside the current
Inventory contract.

## Database contract

- Engine: PostgreSQL; the current API's local Compose selects `postgres:17-alpine`.
- Database: `inventory_db`; use the API's configured default schema (`public` in
  its local setup), rather than the ZIP's shared-database `inventory` schema.
- Migration: [`db/migration/V1__create_products_and_outbox.sql`](db/migration/V1__create_products_and_outbox.sql).
- Tables: `products` and Inventory's technical publication table `outbox_events`.
- Seeds: none in this delivery; there is no confirmed Inventory seed contract.

The migration is an exact copy of Inventory API `develop` at
`9d2889cee517f2c5f644244f8ba5f8ff4068eb01`. It preserves the API's Flyway filename
and SQL bytes. See [the source review and pending differences](docs/source-review.md)
for the ZIP inventory, authoritative references and unresolved decisions.

## Migration handling

This repository prepares SQL for review. It does not provision databases or
automatically apply migrations. The API currently owns execution through Flyway
at `classpath:db/migration`; this repository is not wired into that runtime.
Do not run both copies through separate migration histories. Packaging or execution
ownership changes require a separate reviewed integration.

V1 is for an empty Inventory database under the current contract, not an upgrade
of the ZIP's schema. Never replace an already applied V1 or bypass its checksum.
Future compatible changes require a new numbered migration. Legacy identifier
mapping and data migration remain pending; no legacy records are imported here.

UUIDs and audit timestamps are supplied by the API. Non-blank text validation and
`initial_stock` immutability are application rules, not SQL triggers in this V1.
Monetary precision/rounding and image storage remain pending. No provisioning,
credentials, personal environment files, real data or backups are included.

Governance lives in [`bysellens-docs`](https://github.com/code-corhuila/bysellens-docs).

## Branching

Three permanent branches. **None of them accepts a direct commit** — you enter through a child
branch and leave through a Pull Request.

```text
develop  <--PR--  feat/... fix/... chore/...
qa       <--PR--  qa/...
main     <--PR--  release/...  hotfix/...
```

Promotion happens **by re-application** (`git cherry-pick -x`), never by merging one permanent
branch into another: `merge develop -> qa` and `merge qa -> main` do not exist in this model.

`main` requires **1 approval from `ariel5253`**. On `develop` and `qa` the team sets its own review
rule.

Full policy: [`00-governance/branching-policy.md`](https://github.com/code-corhuila/bysellens-docs/blob/main/00-governance/branching-policy.md).
