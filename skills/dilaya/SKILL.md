---
name: dilaya
description: "Use whenever the user wants to work in or with Dilaya (their own software: apps, data, files, websites, Telegram bots, automation) or calls any Dilaya tool. The entry point: what the connector can do and which dilaya-* skill holds each recipe."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).


# Working with Dilaya

Dilaya is reached through this plugin's MCP connector. Core idea: an **app** = a per-app SQLite database + an S3 folder + per-app skills + (optionally) a web frontend Lambda. This skill is your single entry point: skim the big functions below, then open the matching detailed instructions via the matching `dilaya-*` skill before acting.

**Golden rule:** before touching an app that may already exist, call `list-skills()`, then `get-skill({ schema, name })` to load its instructions + live schema in one call. That is also how you discover the connector's live inventory of apps — this skill describes the connector's *capabilities*, not your specific apps.

**Multi-tenant — the `org` argument.** One connector serves many organizations, and the caller's org(s) live in the OAuth token. Every app-scoped tool takes an optional `org`: a **single-org** session defaults it (omit it), a **multi-org** session **requires** it (else `ORG_REQUIRED`) and it must be one the token allows (else `FORBIDDEN`). `org` is the organization **ID — the `orgId` UUID from `list-organizations` — never its display name** (a uniquely-matching name is auto-resolved as a courtesy, but the id is the contract; an ambiguous name fails `ORG_AMBIGUOUS`). App names are unique **within** an org, so the same name can exist in two orgs without colliding. `set-name` / `get-name` name the **organization** (and seed its vanity-host slug) — never call `set-name` to label the connector.

## Big functions (what this connector can do)

- **User help & onboarding (the `dilaya-guide` skill)** — plain-language help you relay TO the user (most users are non-technical — no jargon). It holds the "Bienvenue sur Dilaya" overview + tutorial menu, the method that turns a need in the user's words into a plan, and each user-voice tutorial. This is *user documentation*; the matching `dilaya-*` skill below is the *builder-facing* recipe book YOU execute.
- **Apps (data + storage)** — create a schema with tables, store files in its S3 folder, query/mutate via `execute` / `query` / `bulk-insert`. Detail: the `dilaya-create-app` skill, `dilaya-use-app`, `dilaya-update-app`.
- **Skills** — per-app instruction docs so any agent can use an app. Detail: the `dilaya-write-skill` skill; manage with `save-skill` / `get-skill` / `list-skills` / `delete-skill`.
- **Cowork dashboards** — a third view type: a live artifact in Claude Cowork, defined by a prompt, with its source code uploaded to S3 + attached as the source of truth so it can be reproduced on another machine. Detail: the `dilaya-dashboards` skill. Tools: `save-cowork-dashboard` / the `dilaya-dashboard-install` skill / `get-cowork-dashboard-source-upload-urls` / `attach-cowork-dashboard-source` / `get-cowork-dashboard` / `list-cowork-dashboards` / `delete-cowork-dashboard`.
- **Frontend (standalone web app)** — a per-app Lambda served at its vanity host `https://<app>-<orgslug>.<contentDomain>/` (or a custom domain), optionally behind passwordless email-OTP auth. Detail: the `dilaya-frontend` skill. Tools: `enable-frontend` / `deploy-backend` / `test-backend` / `disable-frontend` / `enable-auth` / `add-user` / `list-users` / `remove-user-access` / `disable-auth`.
- **App templates (curated starters — check BEFORE hand-writing a frontend)** — begin a web frontend from a **curated starter repo** (framework, styling, build step and Dilaya packaging already wired) instead of scaffolding a site by hand. **Call `list-app-templates` before writing any `handler.js` or HTML** — starting from a template routinely saves hours; write from scratch only when none fits. Detail: the `dilaya-frontend` skill (App templates section). Tools: `list-app-templates` / `create-app-from-template` (plus `get-app-sources` — the call every work session starts from, which says whether the app's sources travel as a **zip** or through a **git** repo — `get-app-template-archive`, `check-git-access`, `create-app-repo` / `get-app-repo`, `set-app-source-mode`).
- **App host (vanity domain)** — additionally serve an app's frontend at a flat, friendly host `https://<app>-<orgslug>.<contentDomain>/` (cert + DNS are a platform-managed wildcard — nothing to configure). Additive to the path URL. Detail: the `dilaya-custom-domains` skill. Tools: `set-app-host` / `list-app-hosts` / `disable-app-host` / `check-app-hosts`.
- **Customer custom domain (BYOD)** — serve an app's frontend on a domain the customer OWNS (`commandes.acme.com`): the tools return the DNS records to add at their provider (cert validation, routing, email DKIM), then `check-custom-domains` promotes it live and can switch the app's mail sender to the domain. Detail: the `dilaya-custom-domains` skill. Tools: `set-custom-domain` / `check-custom-domains` / `list-custom-domains` / `remove-custom-domain`.
- **Transactional email (mail)** — send email (receipts, notifications, confirmations) from an app via a dedicated per-app Postmark server; separate from auth but shares the app's sender domain. Sendable from chat (`send-mail`) or at runtime from a frontend handler (`mail.send(...)`). Detail: the `dilaya-mail` skill. Tools: `enable-mail` / `send-mail` / `list-mail` / `disable-mail`.
- **Integration secrets** — store a 3rd-party API key (OpenAI, Stripe, …) the USER enters in a browser form (never in chat) into SSM SecureString, read at runtime by an app's frontend handler with `secrets.get(name)`. Detail: the `dilaya-secrets` skill. Tools: `get-secret-setup-url` / `list-secrets` / `delete-secret`.
- **Vector search & embeddings (semantic search)** — turn text into vectors on Dilaya's platform key and run KNN search inside an app's db (sqlite-vec), no backend or API key needed: `embed` / `vector-search` / `vector-index-list` / `vector-index-drop`. Enabled PER ORGANIZATION by a Dilaya admin (`VECTOR_TOOLS_NOT_ENABLED` otherwise). Detail: the `dilaya-vectors` skill.
- **App catalog (ready-to-use apps)** — search ready-made app packages published by Dilaya orgs and install one end-to-end (`catalog-search` / `catalog-get` / the `dilaya-catalog-install` skill / `catalog-copy-files`), or publish YOUR app for every Dilaya user or your org only (`catalog-upload-urls` / `catalog-publish` / `catalog-listing-update` / `catalog-unpublish`; immutable semver versions). Console: `setup-catalog-console`. Detail: the `dilaya-catalog` skill.
- **MCP connections (deterministic multi-tool workflows)** — connect the ORG to an external MCP server once (user-consented OAuth), grant an app an explicit tool allowlist, and its backend handler calls those tools directly with `mcp.call(connection, tool, args)` — instant, no agent, no LLM tokens (e.g. confirm a booking AND send the email via a calendar/email MCP from the HTTP handler). Detail: the `dilaya-mcp` skill. Tools: `add-mcp-connection` / `check-mcp-connection` / `grant-mcp-tools` / `revoke-mcp-tools` / `list-mcp-connections` / `remove-mcp-connection`.
- **App crons (scheduled deterministic runs)** — EventBridge invokes the app's deployed backend on a cron or at an instant (`req.cron` in the handler; the handler can pose its own one-shots with `cron.at(...)`, e.g. booking reminders at confirm time). No agent, no LLM. Detail: the `dilaya-crons` skill. Tools: `set-app-cron` / `list-app-crons` / `remove-app-cron`.
- **Telegram bot** — attach a bot to an app to send/receive messages with RBAC roles; inbound is stored in `_telegram_messages`. Detail: the `dilaya-telegram` skill.
- **Agents (automation)** — one **code** agent (a local poller running Claude in tmux, woken only on work) plus many **scheduled** agents (scheduled tasks run by a runtime — ChatGPT or Claude Cowork). Detail: the `dilaya-agents` skill. Tools: `set-agent` / the `dilaya-agent-setup` skill / `list-agents` / `set-agent-schedule`.
- **Managing all apps (cross-app status + archive + console)** — org-scoped tools over your whole app inventory: `list-applications` (one row per app with its lifecycle status — active | archived — plus `archived` / `createdAt` / `archivedAt`, web-frontend state (`frontendEnabled` / `frontendDeployed` / `defaultRoute`), public URLs (`publicUrl` / `vanityHostUrl` / `customDomains`), and active/archived counts; the registry-backed superset of `list-schemas`), `archive-app` / `unarchive-app` (reversible put-aside — an archived app stays listed and readable but writes are refused until unarchived; `confirm: true` to archive), and `setup-management-console` (seed a ready-made interactive Claude Cowork dashboard, `gestion-apps`, that boards every app and acts on them safely — then hand off its Cowork install text). Detail: the `dilaya-management` skill. Tools: `list-applications` / `archive-app` / `unarchive-app` / `setup-management-console`.
- **Consumption metering (billing / tiers)** — `get-usage-report` gives a per-app + org-rollup snapshot: each app's database size and S3 storage, plus org totals (total DB, per-org system db, attributed vs unattributed storage, app counts). `include_storage: false` skips the S3 walk. One connector = one client; a control plane can poll each and aggregate. Detail: the `dilaya-management` skill (Consumption metering).
- **Organization identity & coordination** — `list-organizations` (which org(s) can this session act on — the id to pass as `org`); `set-name` / `get-name` set/read your **organization's** friendly display name (this name also seeds the org's vanity-host slug, so don't repurpose it to label the connector); DynamoDB-backed `acquire-lock` / `renew-lock` / `release-lock` / `lock-status`; and per-agent `notify` / `list-notifications` / `ack-notifications` / `clear-notifications`.

## Choosing the right pattern (IMPORTANT — read before building)

Users describe OUTCOMES ("confirmation immédiate", "un rappel la veille", "un rapport chaque
lundi") — they don't know the architecture. YOU pick the pattern, and the rule is:
**deterministic first, agent only where judgment is needed.** An agent run costs tokens and
minutes; a handler runs in seconds for free. Same discipline on wording: ask **business**
questions only (what, for whom, when) and NEVER make the user arbitrate technical choices or
read jargon ("statique ou dynamique ?", "cache", "Lambda", "vanity host") — translate to
benefits ("plus rapide", "mieux référencé sur Google") and decide yourself.

| The user wants… | Build it as… |
| --- | --- |
| A showcase site / landing / site whose content is the same for every visitor | **Static-first frontend** (the `architecture` section of the `dilaya-frontend` skill): `static_prefixes: ["/"]` + pre-built `site/` served from the CDN on the app's vanity host (`set-app-host` — free, immediate; the customer's own domain is optional, attached later), `handler.js` only for `/api/*` (forms). Faster + better SEO than per-request rendering; dynamic only for personalized/authenticated pages (hybrid) |
| A multi-step action when THEY (or their visitors) act — confirm + email + calendar + Zoom, process a form, sync on submit | **Inline in the backend handler**: `db` + `mcp.call` (external MCP tools, the `dilaya-mcp` skill) + `mail.send` / `secrets.get` — instant, zero LLM. Connections are ORG-LEVEL: `list-mcp-connections` FIRST and reuse before creating; if none fits, PROACTIVELY offer the user the consent link (they don't know this exists) |
| Something at a FIXED TIME with fixed rules — reminders, reports, purges, syncs | **App crons** (the `dilaya-crons` skill): recurring `set-app-cron`, or one-shots the handler poses itself (`cron.at`) — e.g. booking reminders posed at confirm time |
| Live data on a page (availability, stock…) | **Compute per request** in the handler (live `mcp.call` reads over stored state) — don't materialize caches an agent must refresh |
| Understanding free-form language, judgment calls, triage, conversation (Telegram) | **An agent** (the `dilaya-agents` skill): code (reactive local) or scheduled (cloud, in ChatGPT or Claude Cowork) |
| A scheduled job that needs judgment EACH run (editorial digest, anomaly triage) | **Scheduled agent** — the only scheduled-agent case; deterministic schedules are crons |

**Failure doctrine** (bake it into every handler): transient error → ONE bounded immediate retry;
still failing → fail LOUDLY and STRUCTURED (a `last_error` column + a log table row + an owner
email) — never a silent retry loop, never a cron that re-drives failures (a repeating failure is a
BUG). The agent is the RESCUE with judgment: it reuses partial artifacts (ids stored on the row —
NEVER recreate) and decides finish-vs-apologize.

**Channel rule (security)**: agent notifications (`notify`) and Telegram are OWNER↔AGENT channels
— a public-facing handler must NEVER write content into them (public input would flow into an
LLM's context or the owner's pocket). A handler may only: write structured rows in its OWN db,
raise content-free S3 flags, and email via `mail.send`/granted mail tools.

## Router — pick by intent

- User is new / asks "what can I do here / how does this work" → the welcome of the `dilaya-guide` skill; a user describes a goal in plain words → its need → plan method; a user asks about one task → its tutorial
- Create/build an app → the `dilaya-create-app` skill
- Use / list / query an app → `list-skills` then `get-skill`; deeper context → the `dilaya-use-app` skill
- `get-skill` shows `backend_actions` → CALL the action (`run-app-action` / `-write`) instead of recomputing it in SQL; freeze a treatment as an action (with or without a web page) → the `actions` section of the `dilaya-frontend` skill
- Add a column / evolve a table → the `dilaya-update-app` skill
- Write or improve an app's skill → the `dilaya-write-skill` skill
- Build a website / showcase site / blog / landing / docs for an app → **`list-app-templates` FIRST** (a curated starter may scaffold the whole thing), then `create-app-from-template`; details in the `dilaya-frontend` skill
- Frontend / deploy / auth → the `dilaya-frontend` skill (its **App templates** section first — check `list-app-templates` before hand-writing any `handler.js`; write from scratch only if no template fits)
- Give an app a friendly vanity URL (`<app>-<org>.<contentDomain>`) → the `dilaya-custom-domains` skill, then `set-app-host`
- Serve an app on the customer's own domain → the `dilaya-custom-domains` skill, then `set-custom-domain` + `check-custom-domains`
- Send transactional email from an app → the `dilaya-mail` skill, then `enable-mail` + `send-mail`
- Store a 3rd-party API key an app's frontend reads at runtime → the `dilaya-secrets` skill, then `get-secret-setup-url` (user opens the link) + `secrets.get(name)` in the handler
- Let an app's backend call external MCP tools itself (deterministic workflow, no agent) → the `dilaya-mcp` skill, then `add-mcp-connection` (user opens the consent link) + `grant-mcp-tools` + `mcp.call(...)` in the handler
- Run the app's backend on a schedule (reminders, reports, purges — no agent) → the `dilaya-crons` skill, then `set-app-cron` (recurring) or `cron.at(...)` in the handler (one-shot)
- Semantic search / embeddings / RAG over an app's data ("retrouve par le sens", reformulations, synonymes) → the `dilaya-vectors` skill, then `embed` + `vector-search` (org access required — `VECTOR_TOOLS_NOT_ENABLED` = ask a Dilaya admin to enable it at dilaya.eu)
- Telegram bot (attach, send, receive, RBAC) → the `dilaya-telegram` skill
- Cowork dashboard (live Cowork artifact + S3 source of truth) → the `dilaya-dashboards` skill
- Create or update an app's agent(s) → the `dilaya-agents` skill, then `set-agent` + the `dilaya-agent-setup` skill
- See / manage ALL apps in one place (status, archive) → the `dilaya-management` skill, then `list-applications` / `archive-app` / `unarchive-app`
- Give the user a visual, interactive board over all apps → the `dilaya-management` skill, then `setup-management-console` + hand off the `dilaya-dashboard-install` skill
- Measure consumption (DB / storage / apps) for billing or tiers → `get-usage-report`; context → the `dilaya-management` skill
- Drop an app → `drop-schema({ schema, confirm: true })` (destructive — confirm with the user first)

## Hard rules (these surface as cryptic errors otherwise)

- Schema names match `^[a-z][a-z0-9]{0,62}$` — lowercase alphanumeric, no hyphens/underscores/leading digit.
- Use `:param_name` placeholders; never string-concat values into SQL.
- `query` is SELECT/WITH only (≤1000 rows); everything else via `execute`. Table names are unqualified — one SQLite database per app (you cannot JOIN across apps).
- `enable-auth` requires `enable-frontend` first.
- Multi-org sessions must pass `org` on app-scoped tools (`ORG_REQUIRED` otherwise); a single-org session defaults it. `org` = the organization **ID** (the `orgId` UUID from `list-organizations`), never its name. `set-name` / `get-name` are your **org's** display name, not the connector's — never call `set-name` to label the connector.

When unsure, call the matching `dilaya-*` skill for the closest topic, or `list-skills` + `describe-schema` to orient.
