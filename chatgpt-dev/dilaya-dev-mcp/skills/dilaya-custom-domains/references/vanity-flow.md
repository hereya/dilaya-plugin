## Flow

1. (Recommended) `set-name({ name: "Novopattern" })` first, so the org slug reads well — the slug is derived from this name and is **stable for the life of the org** (it does not change if you rename later).
2. `set-app-host({ schema: "smartcal" })` → `{ host, url }`. Provisions the vanity host. Idempotent — safe to re-run.
3. Make sure the frontend is actually deployed (`deploy-backend`) so the host serves. `enable-frontend` / `deploy-backend` now also report the vanity `host_url` alongside the path `url`.
