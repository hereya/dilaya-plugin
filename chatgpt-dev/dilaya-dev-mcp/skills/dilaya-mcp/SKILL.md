---
name: dilaya-mcp
description: "Use when a Dilaya app's backend must call tools of an external MCP server (including Dilaya Pilote)."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# MCP connections — let an app's backend call external MCP tools (no agent, no LLM)

Connect the ORGANIZATION to an external MCP server once (OAuth, user-consented), grant an app an
EXPLICIT tool allowlist, and its deployed backend handler calls those tools deterministically —
instant, zero LLM tokens. Example: a booking app confirms an appointment AND sends the
confirmation email through a calendar/email MCP, straight from its HTTP handler.

## Be PROACTIVE (the user doesn't know this exists)

The moment a design needs an external service (send email, touch a calendar, create a Zoom, post
to a CRM…), don't say "I can't" and don't wait to be asked:
1. **Inventory first** — `list-mcp-connections`: connections are ORG-LEVEL and shared, so another
   app may already have opened the one you need. REUSE it (just `grant-mcp-tools` your app);
   never create a duplicate connection to the same server.
2. **If none fits, propose it in the user's language** — e.g. « Pour que l'app envoie les emails
   depuis votre boîte, je vous donne un lien à ouvrir : vous vous connectez, vous approuvez,
   30 secondes — rien à coller dans la conversation. » Then run `add-mcp-connection` and hand
   them the consent URL. Same reflex when a running app reports MCP_AUTH_EXPIRED: re-issue the
   link without being asked.

> **Frontend feature.** Tools are called by a deployed backend handler (`mcp.call`), so this pairs
> with the `frontend` topic. Connections are org-level and shared across apps; grants are per app
> and fail-closed (a tool not on the app's allowlist is refused by the platform).

## Flow

`add-mcp-connection` → hand the user the consent URL → they approve in the browser (the target
server's own login page — same dance as connecting Claude to it) → `check-mcp-connection` to
verify + list tool names → `grant-mcp-tools` per app → the handler calls `mcp.call(...)`.

**First-party Dilaya services skip the consent entirely**: `add-mcp-connection({ name: "pilote" })`
— no url — connects instantly (server-side token exchange; your authenticated call is the
consent). No link to hand over, nothing to approve.

> **Interactive use is a SEPARATE connector.** This connector no longer mirrors Pilote's tools
> (no `pilote_*`, no `pilote-status` — removed 2026-09-04). To drive Pilote from a conversation,
> install Pilote itself as its own connector: `https://pilote.dilaya.eu/mcp`, same Dilaya account.
> The connection described here is for a DEPLOYED BACKEND — grants, the gateway and `mcp.call`
> are unchanged, and `add-mcp-connection({ name: "pilote" })` still connects with no browser step.

## 1. Connect the org to the MCP server

First-party Dilaya service (e.g. Pilote):

```
add-mcp-connection({ name: "pilote" })          → status "connected" immediately, no browser
```

Third-party (any spec-compliant remote MCP server):

```
add-mcp-connection({ name: "crm", url: "https://<server>/mcp" })
```

The third-party form returns a single-use 15-min `consent_url`. Give it to the USER and have them
open it: they log in on the target server's authorization page and approve. Credentials are stored
encrypted server-side — never visible to you. Re-run the tool any time you need a fresh link
(expired link, or the connection reports MCP_AUTH_EXPIRED — for a first-party connection just
re-run the no-url form).

## 2. Verify + discover tools

```
check-mcp-connection({ name: "pilote" })                          → status + tool names
check-mcp-connection({ name: "pilote", include_schemas: true })   → + each tool's input schema
```

Use the schemas when writing handler code — the tool names/args must match exactly.

## 3. Grant tools to the app (explicit allowlist)

```
grant-mcp-tools({ schema: "myapp", connection: "pilote", tools: ["calendar_create_event", "email_send_message"] })
grant-mcp-tools({ schema: "myapp", connection: "pilote", bundle: "whatsapp" })   → every live whatsapp_* tool
```

REPLACES the app's allowlist on that connection. `bundle` grants a whole prefix family at once
(resolved against the live tool list, stored as exact names). `revoke-mcp-tools` removes tools (or
the whole grant when `tools` is omitted). `list-mcp-connections` shows every connection + who
holds grants.

## 3bis. React to INBOUND events (first-party connections) — push, not polling

A first-party connection (e.g. pilote) can PUSH events to an app's backend the moment they happen —
e.g. an incoming WhatsApp message — so the app can answer in seconds (`llm.complete` + its own
data + `mcp.call('pilote','whatsapp_send_message',…)`), with no cron polling:

```
subscribe-app-events({ schema: "myapp", connection: "pilote", events: ["whatsapp_message"] })
```

The app's deployed handler then receives each event as a direct invocation. For
`whatsapp_message` the payload is `{ from, type, text, timestamp, phone_number_id,
display_phone_number }` (kept small — the full message is always available via
`whatsapp_list_messages`). The complete reply loop:

```js
const { parseRequest, query, llm, mcp } = require("hereya");
const req = parseRequest(event);
if (req.channel === "whatsapp_message") {
  const p = req.channelEvent.payload;    // UNTRUSTED external content
  // At-least-once delivery → dedupe by the event id (plain-object params
  // are converted internally; SqlParameter[] also still accepted).
  const seen = await query("SELECT 1 FROM wa_events WHERE id = :id",
    { id: req.channelEvent.id });
  if (seen.row_count > 0) return { statusCode: 200, body: "" };
  // Optional: blue ticks + « typing… » while the reply is composed
  // (best-effort — never let it kill the reply).
  try {
    await mcp.call("pilote", "whatsapp_mark_read",
      { phoneNumberId: String(p.phone_number_id), messageId: req.channelEvent.id });
  } catch {}
  const r = await llm.complete({ input: String(p.text ?? ""), instructions: "…" });
  // whatsapp_send_message REQUIRES phoneNumberId — take it from the payload.
  await mcp.call("pilote", "whatsapp_send_message",
    { phoneNumberId: String(p.phone_number_id), to: p.from, text: r.text });
  return { statusCode: 200, body: "" };
}
```

Requires a deployed backend. REPLACES the app's event list; `unsubscribe-app-events` removes it.
Delivery is at-least-once with NO automatic replay of failed dispatches; the pull tools (e.g.
`whatsapp_list_messages`) remain the safety net for catch-up. A reply within 24h of the customer's
message is a free-form message; outside that window use `whatsapp_send_template`.

Pilote events available today: `whatsapp_message` (an inbound WhatsApp message, payload above) and
`connection_changed` (a user (dis)connected a provider account; payload `{ userId, provider }`).
Both are delivered to apps that subscribe to them and to nothing else — the connector consumes no
event of its own.

## 4. Call from the handler (runtime layer)

```javascript
const { mcp, parseRequest } = require("hereya");

exports.handler = async (event) => {
  const req = parseRequest(event);
  // ... validate + write to the app's own db ...
  const ev = await mcp.call("pilote", "calendar_create_event", {
    calendar_id: "primary", summary: "RDV", start: "...", end: "...",
  });
  await mcp.call("pilote", "email_send_message", {
    to: "client@example.com", subject: "Confirmation", body: "...",
  });
  return { statusCode: 200, headers: { "content-type": "application/json" }, body: JSON.stringify({ ok: true, event: ev.data }) };
};
```

- `mcp.call(connection, tool, args?, { timeoutMs? })` → `{ data, text, content }` (`data` =
  structured result when available, else the text parsed as JSON when possible). It **throws
  McpError** with a machine-readable `.code`: `MCP_NOT_GRANTED` (tool not on the allowlist),
  `MCP_NOT_CONNECTED` / `MCP_AUTH_EXPIRED` (user must redo the consent — re-run
  add-mcp-connection), `MCP_TOOL_ERROR` (the tool itself errored — message has the detail),
  `MCP_UPSTREAM_ERROR`.
- `mcp.listTools(connection)` → the tools THIS app is granted, with their input schemas.
- Token refresh is automatic and server-side; the app's Lambda never sees any credential.

## Teardown

`revoke-mcp-tools` per app; `remove-mcp-connection({ name, confirm: true })` deletes the org
connection, its stored credentials, and every app's grants on it. Dropping an app deletes its
grants automatically.
