## 2. Write the handler

Export a `handler` function from a file named `handler.js`, importing the runtime layer as the module `hereya`:

```js
const { query, sql, parseRequest, storage } = require("hereya");

exports.handler = async (event) => {
  const req = parseRequest(event);            // SYNCHRONOUS — no await
  const r = await query("SELECT count(*) AS n FROM items");
  return {
    statusCode: 200,
    headers: { "content-type": "text/html" },
    body: `<h1>${r.rows[0].n} items</h1> path=${req.path}`,
  };
};
```

SQL is **unqualified** — one SQLite db per app, no schema prefix: `FROM items`, not `FROM myapp.items`.

### Runtime layer (`hereya`)

- `parseRequest(event)` → **synchronous** `AppRequest` (no `await`): `{ path, method, headers, query, body, cookies, auth: { authenticated, email, dilaya_user_id, cognito_sub, agent? }, orgId, appId, appName, schema, baseUrl, host, clientIp }`. A HEAD request (uptime monitors, link checkers) arrives as `method: "GET"` with `head: true` — route it like any GET, the body is dropped for you; `path` is relative to the site root; `baseUrl` is the public origin for absolute links; `host` is the host the visitor actually called (vanity host / custom domain — the plain `Host` header never carries it behind the CDN); `clientIp` is the visitor's TRUSTWORTHY IP (the CDN-seen X-Forwarded-For entry — never parse XFF yourself: the first entry is client-forgeable).
- `query(sql, params?)` → `{ columns, rows, row_count }` (rows as objects), for SELECTs.
- `sql(sql, params?)` → the raw Data API result, for writes / when you need `.records` or `.numberOfRecordsUpdated`.
- Params: both take a plain object — `await query("SELECT * FROM items WHERE id = :id", { id })` — converted internally. `convertParams({ id })` still exists and pre-built `SqlParameter[]` still passes through unchanged (older handlers keep working).
- `cacheHeaders(seconds)` → `{ "cache-control": "public, max-age=<s>" }` — opt-in edge caching for PUBLIC responses (see §3, static assets + cache).
- `storage` — org+app-scoped S3 helpers (`getFileContent`, `putFileContent`, `getUploadUrl`, `getDownloadUrl`, `listFiles`, `deleteFile`, …), all app-relative paths. `listFiles` `truncated`→`{cursor:next_cursor}`.
- `mail.send({ to, subject, html?, text?, attachments? })` → `{ message_id }` — send transactional email from the app, optionally with attachments from the app's storage or generated base64 (needs `enable-mail`; see the `dilaya-mail` skill).
- `secrets.get(name)` — read a 3rd-party API key the user entered out of band (see the `dilaya-secrets` skill). Both throw a clear error when their feature isn't set up yet.
- `mcp.call(connection, tool, args?)` / `mcp.listTools(connection)` — call tools on an external MCP server the org connected and this app was granted (see the `dilaya-mcp` skill). Throws `McpError` with a machine-readable `.code` when refused or failing.
- `cron.at(name, when)` / `cron.every(name, expr, tz?)` / `cron.remove(name)` — schedule the app's OWN future invocations (one-shot reminders, recurring jobs); the firing arrives as `parseRequest(event).cron === "<name>"` (see the `dilaya-crons` skill).
- `llm.complete({ input, instructions?, model?, maxOutputTokens?, jsonSchema? })` → `{ text, model, usage }` and `llm.embed(texts, { model?, dimensions? }?)` → `{ vectors, ... }` — run an LLM / embeddings on the platform key via the connector's gateway (no key in the app). Requires the org's « IA pour les apps » opt-in at dilaya.eu; a monthly budget is enforced server-side (throws `LlmError` code `LLM_BUDGET_EXCEEDED` when exhausted, `LLM_NOT_ENABLED` when off). Treat the reply as untrusted text — render or store it, never execute it.
- `claude.complete({ input, instructions?, model?, maxTokens?, timeoutMs? })` → `{ text, model, stopReason, usage }` — Claude on the app OWNER's own Claude API key, called directly from the handler (fastest path, no Dilaya budget: the owner's Claude Console bills and caps it). The key is the app secret `_ANTHROPIC_API_KEY` (the one a managed agent of the app uses), entered by the owner on the page from `get-secret-setup-url({ schema, name: "_ANTHROPIC_API_KEY" })` — or, when the app has none, the organization's keyless federation (`name: "_ANTHROPIC_FEDERATION", scope: "org"`), else its key (same call with `scope: "org"`). Default model claude-haiku-5-5; output ≤ 8192 tokens; 25 s timeout by default (a site request has 30 s — longer work goes to a cron). Throws `ClaudeError` with `code`: `CLAUDE_KEY_MISSING`, `CLAUDE_KEY_INVALID`, `CLAUDE_CREDIT_EXHAUSTED`, `CLAUDE_RATE_LIMITED`, `CLAUDE_TIMEOUT`… — tell the owner which, the fix is in their Claude Console. Metered in `_llm_calls` (kind `claude`). Same rule: the reply is untrusted text.
- `claude.run({ input, instructions?, model?, maxTokens?, tools, maxTurns?, timeoutMs?, confirmed? })` → `{ text, model, stopReason, usage, actions, pending?, error? }` — same key and errors, but Claude can ACT: a bounded loop (6 API calls, 10 at most; 24 s) calling functions of YOUR code. Recipe:
  - **A tool is an app function with the identity closed over.** Build `tools` per request from `req.auth`, never with a user id or e-mail as an argument, so the model cannot choose who it acts for: `tools: { list_orders: { description, input_schema, handler: async () => ordersOf(email) }, cancel_order: { description, input_schema, confirm: true, handler: async ({ numero }) => cancelOrder(email, numero) } }`. A thrown message reaches the model.
  - **`confirm: true` on every irreversible action.** The run stops unexecuted: `stopReason: "confirmation_required"`, `pending: { name, args }`, a `text`. Show a button YOUR code labels from `pending.args`; its own route calls `claude.run({ tools, confirmed: { [name]: args } })`: the handler runs with no API call (`stopReason: "confirmed"`, outcome in `actions[0].ok`; add `input` to have Claude word it). "Yes" typed in the chat confirms nothing. Those args came through the browser: the handler re-checks identity and state in its SQL (`UPDATE … WHERE email = :e AND numero = :n AND statut = 'open'`).
  - Other stops: `max_turns`, `timeout`, `api_error`: `text` may be empty, `actions` says what ran. Audit: `_llm_actions`.
- `telegram.notifyOwner(text)` → `{ delivered }` — escalate to the app OWNER on Telegram (recipients resolved server-side from the bot allowlist's owner role; needs `attach-telegram` + token setup). Rate-limited per day; message arrives prefixed with the app name. This is the ONLY Telegram surface for handlers — no arbitrary recipients, and the agent `notify` channel stays closed to handlers.
- `antibot.check(req, data, opts?)` → `{ pass, reason }` — see the anti-bot section below. `antibot.verifyTurnstile(token)` for the optional Cloudflare Turnstile add-on.

### Protecting a PUBLIC form (anti-bot) — use this on every public POST

Any public endpoint gets spammed. The runtime ships the full defense — never hand-roll it (hand-rolled versions get XFF parsing wrong: the FIRST X-Forwarded-For entry is client-forgeable; `req.clientIp` reads the CDN-seen entry):

**Page side** — add two things to the form: a hidden honeypot field real users never fill, and an elapsed-ms stamp:
```html
<input type="text" name="website" style="position:absolute;left:-9999px" tabindex="-1" autocomplete="off">
<script>const t0 = Date.now();
  // on submit: body.website = form.website.value; body.elapsed = Date.now() - t0;
</script>
```

**Handler side** — branch on `reason`; the reject doctrine is DIFFERENTIATED:
```js
const { parseRequest, antibot, sql } = require("hereya");
const req = parseRequest(event);
const data = JSON.parse(req.body ?? "{}");
const bot = await antibot.check(req, data); // honeypot + timing + rate limit (default 3/h/IP)
if (!bot.pass) {
  if (bot.reason === "rate_limited") // possibly a HUMAN behind a shared IP — answer honestly
    return { statusCode: 429, headers: {"content-type":"application/json"}, body: JSON.stringify({ ok: false, error: "Trop de demandes depuis votre réseau — réessayez dans quelques minutes." }) };
  // honeypot / too_fast: near-certain bot → SILENT REJECT (normal success, persist nothing — never inform a bot)
  return { statusCode: 200, headers: {"content-type":"application/json"}, body: JSON.stringify({ ok: true }) };
}
await sql("INSERT INTO messages …");
```

- **Why the split**: `honeypot`/`too_fast` are near-certain bots — silent-reject them. `rate_limited` can be a LEGITIMATE human: the limit is per client IP, and one public IP often fronts many people (café/campus NAT, mobile CGNAT) — a silent fake success would drop a real person's message without anyone knowing. Show that person a real retry-later message (and make sure the page displays it).
- Defaults: honeypot field `website`, elapsed field `elapsed` (min 3000 ms), 3 submissions/hour/IP/endpoint. Override via `opts` (`{ honeypot, elapsedField, minElapsedMs, ratePerHour, endpoint }`).
- **Tune `ratePerHour` to the form's audience — the 3/h default is deliberately conservative** (sized for one visitor): for a public form likely reached from shared IPs (cafés, campuses, mobile networks), set `antibot.check(req, data, { ratePerHour: 10 })` or higher (10–20/h). Honeypot + timing do the real bot filtering; the rate limit is only the backstop, so raising it costs little.
- The rate limit lives in a runtime-managed table `_antibot_hits` (auto-created, sliding 1-hour purge) — your business tables are untouched. IPs are stored ONLY as truncated SHA-256 hashes (GDPR by design) — still mention form-abuse protection in the site's privacy policy.
- **Optional Turnstile** (BYO key, free tier): the user creates a Cloudflare Turnstile widget (sitekey in the page), stores the SECRET key via `get-secret-setup-url({ schema, name: "turnstile" })`, and the handler adds `const ts = await antibot.verifyTurnstile(data.turnstileToken); if (!ts.pass) …`. Cloudflare's test keys (`1x00…AA`/`2x00…AB`) work for E2E.
- NO WAF in the base platform (fixed per-ACL cost — incompatible with zero-fixed-cost distributions). An org under a REAL attack can ask the platform admin about attaching a Web ACL to ITS distribution as a billed option.
