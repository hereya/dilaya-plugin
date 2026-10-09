## Isolation

Each app's frontend is sandboxed to that app. The per-app Lambda runs under its **own IAM role** scoped to `<orgId>/<app>/*` in S3 (it cannot read another app's files), and its database calls carry a **capability token** bound to `(orgId, appId)` that the SQLite VM enforces (missing/invalid → denied). A handler therefore reaches only its own app's data and files.
