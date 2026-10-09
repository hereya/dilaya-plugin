## Deployment versions & rollback

Every successful `deploy-backend` archives the exact deployed zip as a **version** (the last 10 are kept; the live deployment is always the highest version). Nothing to opt into — the response includes `version`.

- `list-deployments({ schema })` — the archived versions, newest first (timestamp, sha256, size, what the zip carried).
- `rollback-deployment({ schema, to_version })` — restores that archived zip and re-runs the standard deploy pipeline. The rollback becomes a **new** version (linear history, like `git revert` — you can roll back a rollback). Everything the zip produced is restored together: Lambda code, `site/` bundle, `assets/`. Edge-cached HTML refreshes within ~60 s.

Note: the archive is the ZIP, not the app config — `static_prefixes` (and auth) apply as they are configured at rollback time.
