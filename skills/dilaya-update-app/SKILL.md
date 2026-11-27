---
name: dilaya-update-app
description: "Use when changing an existing Dilaya app's structure: new tables or columns, migrations, skill updates."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# How to Update an Existing App on Dilaya

## Steps

### 1. Load the current state
```
get-skill({ schema: "recipes", name: "main" })
```
This gives you the current skill instructions AND schema structure.

### 2. Make schema changes
Use `execute` for ALTER TABLE, CREATE TABLE, etc. — SQLite dialect, unqualified table names:
```
execute({ sql: "ALTER TABLE recipes ADD COLUMN difficulty TEXT" })
execute({ sql: "CREATE INDEX idx_recipes_category ON recipes (category)" })
execute({ sql: "CREATE TABLE tags (id INTEGER PRIMARY KEY AUTOINCREMENT, recipe_id INTEGER REFERENCES recipes(id), tag TEXT)" })
```
SQLite's `ALTER TABLE` supports `ADD COLUMN`, `RENAME`, and `DROP COLUMN` — but not changing an existing column's type. To change a type, add a new column and copy the data across (or rebuild the table).

If `execute` answers `MIGRATION_REQUIRED`, the app has separate development and production databases: structure changes are migrations — see "Apps with separate development and production databases" below.

### 3. Update the skill
After changing the schema, always update the skill to reflect the new structure:
```
save-skill({
  schema: "recipes",
  name: "main",
  description: "Manage recipes with ingredients, portions, and tags",
  content: "... (updated instructions reflecting new columns/tables)"
})
```

### 4. Verify
```
describe-schema({ schema: "recipes" })
```
Confirm the schema matches your intent.

## Destructive changes
- `DROP TABLE`: Use `execute({ sql: "DROP TABLE tags" })`
- `DROP SCHEMA`: Use the dedicated `drop-schema` tool (requires `confirm: true`). This also deletes all files and skills for the app. (`DROP SCHEMA` / `DROP DATABASE` via `execute` is rejected.)
- Column removal: `execute({ sql: "ALTER TABLE recipes DROP COLUMN difficulty" })`

Always update the skill after destructive changes.

## Apps with separate development and production databases

Some apps have TWO databases: a **development** one (fictitious data — what `query`, `execute`, `bulk-insert`, `describe-schema` and the **staging** site use) and the **production** one (the real data — the production site). You never write production data; `schema: "<app>:prod"` reads it with `query` / `describe-schema` only when the platform opened it (else `PROD_DATA_CLOSED`). Signs: `execute` answers `MIGRATION_REQUIRED` to a CREATE / ALTER / DROP, `list-secrets` shows `dev_configured`.

Turning it on or off: in an organization whose tools include `set-app-db-mode`, the app's creator or an owner/admin of the organization calls `set-app-db-mode({ schema, mode: "separated" })` (an empty development database beside the unchanged production one; the staging site moves to it at its next staging deploy) or `mode: "single"` (back to one database; refused with `MIGRATION_NOT_PROMOTED` while development has migrations production lacks). Ask the user first: it changes where the staging site and your data tools work.

The structure changes ONLY through migrations, so the two never drift apart:
1. Write `migrations/NNNN_<what>.sql` in the deployment zip — the next free number (`0001_add_orders.sql`, `0002_…`), plain SQL, several statements allowed. Keep every earlier file in the zip, unchanged.
2. `deploy-backend({ schema, env: "staging", confirm: true })` applies the new ones to the development database (each in its own transaction), then serves the code. The answer's `migrations` lists what was applied and what is `to_promote`.
3. Add fictitious rows with `bulk-insert` / `execute` (INSERT, UPDATE, DELETE work) and test the staging site.
4. `promote-deployment({ schema, confirm: true })` applies them to production one by one BEFORE switching the code; the first failure stops it with the code not switched — fix the cause and promote again, it resumes at that migration. `schema_drift` in the answer means the two structures differ (say so to the user).

Rules (refusal codes): an applied migration is never edited, renamed or removed (`MIGRATION_IMMUTABLE` — write a new one to undo it); a new one numbered below the last applied is refused (`MIGRATION_OUT_OF_ORDER`); a production deploy-backend whose zip carries a migration production has not applied is refused (`MIGRATION_PENDING` — go through staging then promote); `rollback-deployment` restores CODE only, never the structure. Tables whose name starts with `_` belong to the platform (`SYSTEM_TABLE`).

A migration that applied on development but FAILS on production can no longer be edited (development recorded it): ask the user to have the platform **reset the development database** (Dilaya support) — it then has production's structure and history, no data, and the fixed migration is applied again at the next staging deploy.

Not separated: files, the site's sign-in pool, Telegram, organization keys. Staging mail only reaches organization members (others are withheld and logged), and staging cannot schedule crons (`CRON_PROD_ONLY`). Each secret has a production and a development value; the staging site reads only the development one.

## Apps delivered from Git

Some apps are delivered from their Git repository instead of a deployment zip (only in an organization whose tools include `commit-app-files`). Sign: `deploy-backend` answers `GIT_MANAGED_APP`. On such an app:
- **Code**: `commit-app-files({ schema, … })` writes several files (added, replaced or deleted) as ONE commit on main. The platform builds it and serves it in the development environment; it never changes production. Follow it with `get-dev-status({ schema, commit })` (the build, the error it met, what development serves).
- **Structure**: a NEW file `migrations/NNNN_<what>.sql` (the next free number) sent with `commit-app-files` — applied before the new code is served. `execute` refuses CREATE / ALTER / DROP (`MIGRATION_REQUIRED`); INSERT, UPDATE and DELETE still work. A migration already committed cannot be changed: write a new one.
- Tables whose name starts with `_` belong to the platform (e.g. `_dilaya_migrations`): `execute` and `bulk-insert` refuse to write them (`SYSTEM_TABLE`).
- `deploy-backend` (both environments), `promote-deployment`, `rollback-deployment` and `disable-staging` answer `GIT_MANAGED_APP`: production is released from a version confirmed by a person — an owner of the organization, not you. Tell the user; `list-deployments` shows the versions.
