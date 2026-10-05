## Optional staging environment

For impactful changes, deploy to **staging** first, test, then promote:

```
deploy-backend({ schema: "myapp", env: "staging" })   # 1st call provisions staging lazily
# → test on https://myapp-<orgslug>--stg.<contentDomain>/ (or test-backend({env:'staging'}))
promote-deployment({ schema: "myapp" })               # the EXACT staged zip goes live (new version)
```

- Staging = **same app, same DATA** (SQLite, files, secrets, auth) — only the CODE is the candidate. It is a preview of new code on real data, not a data sandbox.
- Upload flow is unchanged: same `<app>/backend/deployment.zip` slot; `env` picks the target. Each staging deploy snapshots its exact zip, so what you promote is what you tested — never a rebuild.
- One candidate at a time (a new staging deploy replaces the previous candidate). Promotion enters the normal version history (`promoted_from_staging`) — rollback works as usual.
- The staging host rides the existing wildcard domain; auth (when enabled) protects it like production. Crons, Telegram and mail always target production.
- `disable-staging({ schema, confirm: true })` tears staging down; production is untouched.
