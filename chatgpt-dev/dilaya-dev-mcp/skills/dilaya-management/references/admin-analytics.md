## Platform-admin analytics (admin orgs only)

If (and only if) your session includes a **platform-admin org** (the `platformAdmin` flag, granted by a platform super-admin at dilaya.eu), three extra org-scoped tools are listed — restricted to that org's owners, cross-ORG by design (they observe the whole platform, no app data):

- `admin-usage-stats({ from?, to?, target_org? })` — daily usage series from the connector's chokepoint counters: calls, denied calls, active users, top tools. Platform-wide by default (+ per-org lifetime totals and last activity); `target_org` drills into one org. Defaults to the last 14 days (max 92). Counters start at the feature's ship date.
- `admin-org-overview()` — one row per organization: name, status, app counts, deployed frontends, lifetime calls, last activity — sorted by recency (actives first, dormants last).
- `setup-analytics-console()` — seed/refresh the `analytics` Cowork dashboard in the reserved `_console` app: the platform cockpit (activity curves, top tools, active vs dormant orgs, per-org drill-down) fed by the two tools above. After seeding, hand the `dilaya-dashboard-install` skill to the user (same flow as the management console). Idempotent — re-run to ship an updated prompt.

If these tools are not in your list, the session has no admin org — there is nothing to enable from chat.
