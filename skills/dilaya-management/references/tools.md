## Tools

- `list-applications()` — cross-app status, one row per app (`status` / `archived` / `createdAt` / `archivedAt` / `frontendEnabled` / `frontendDeployed` / `defaultRoute` / `publicUrl` / `vanityHostUrl` / `customDomains`) + active/archived counts. Org-scoped, registry-backed. Excludes the reserved `_console` app.
- `setup-management-console()` — seed/refresh the `gestion-apps` Cowork dashboard in the reserved `_console` app, then hand the `dilaya-dashboard-install` skill to the user. Org-scoped, idempotent.
- `archive-app({ schema, confirm: true })` — reversible put-aside (registry status `archived`; writes refused, data kept). `unarchive-app({ schema })` — back to `active`.
- `get-usage-report({ include_storage?: boolean })` — per-app DB + S3 usage + org rollups for billing / tiers. `include_storage: false` skips the S3 walk. Read-only.
- `set-app-visibility({ schema, visibility })` — who INSIDE the organization may use the app (`organisation` | `personal`). `reassign-app({ schema, to_user_id? })` — change its owner. See below.
