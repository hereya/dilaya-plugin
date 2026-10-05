---
name: dilaya-crons
description: "Use when a Dilaya app's backend must run on a schedule or at a given time."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# App crons — scheduled deterministic runs of the app's backend (no agent, no LLM)

Schedule the app's deployed backend Lambda to run on a cron (recurring) or at an instant
(one-shot) — EventBridge invokes it directly with `req.cron === "<name>"`. Use it for reminders,
reports, purges, syncs… anything with a fixed time and a deterministic recipe. The agent stays out
of the loop entirely.

> **Frontend feature.** Crons invoke the deployed backend handler, so this pairs with the
> `frontend` topic (`deploy-backend` first). Doctrine: a cron does SCHEDULED BUSINESS WORK — it
> is NOT a retry mechanism for failures (those must fail loudly and be fixed).

## Recurring cron (agent-managed)

```
set-app-cron({ schema: "myapp", name: "rapport", cron: "0 8 * * ? *", tz: "Europe/Paris" })
```

EventBridge 6-field syntax: `min hour day month weekday year` (use `?` for day-or-weekday you
don't constrain). Upserts by name. `list-app-crons` / `remove-app-cron` round it out.

## One-shot (usually runtime-managed)

The handler can pose its OWN schedules — e.g. booking reminders at confirm time:

```javascript
const { cron, parseRequest } = require("hereya");
// at confirm time:
await cron.at("rappel24-" + bookingId, new Date(slotStartMs - 24 * 3600e3));
// EventBridge fires it once, the schedule self-deletes; the handler receives:
exports.handler = async (event) => {
  const req = parseRequest(event);
  if (req.cron && req.cron.startsWith("rappel24-")) {
    // ... send the reminder deterministically (db + mcp.call/mail.send) ...
    return { statusCode: 200 };
  }
  // ... normal HTTP routes ...
};
```

- `cron.at(name, when)` (one-shot, UTC instant, self-deleting) · `cron.every(name, expr, tz?)`
  (recurring) · `cron.remove(name)` · `cron.list()`. All throw `CronError` with a `.code`.
- The cron invocation arrives by IAM (never through the public URL) — `req.cron` cannot be forged
  by a visitor. For any HTTP request it is `undefined`.
- Retries are short and bounded (2 attempts, ≤5 min): a persistently failing cron is a BUG — log
  loudly (`reconcile_log`-style table + last_error columns), don't re-drive it.

## Teardown

`remove-app-cron` by name; `disable-frontend` and `drop-schema` remove ALL the app's schedules.
