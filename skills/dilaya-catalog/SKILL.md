---
name: dilaya-catalog
description: "Use when publishing a Dilaya app to the app catalog, or managing a published listing."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# App catalog — publish and install ready-to-use apps

The **catalog** ("catalogue d'apps") lets an org share a ready-to-install app with every Dilaya user (`visibility: "public"`) or with its own org only (`"org"`). A published app is a **package**: a versioned set of files with a mandatory `dilaya.md` doc at its root. The connector only stores and serves packages — **installation is executed by the agent** (you), following the package's doc with the ordinary tools. Published versions are **immutable**: fixing anything means publishing a new semver.

## Consuming: find and install an app

1. `catalog-search({ q?, tag? })` — the listings your org can see (public + your org's own). Each row: `name`, `title`, `summary`, `tags`, `author_org`, `latest_version`, `requires` (human-readable requirement lines).
2. `catalog-get({ name, version? })` — full detail: the `dilaya.md` doc inline + download URLs for the package's other files + all published versions.
3. the `dilaya-catalog-install` skill — THE install path: a self-contained prompt (pre-flight + scope rules + the package's doc). Follow it directly when the user asks to install, or hand the text to the user. Non-negotiables it enforces (mirror them if you improvise): present the app + its requirements and get the user's explicit OK **before creating anything**; touch ONLY the newly created app; pass `installed_from: "<name>@<version>"` to `create-schema` (provenance, shows up in `list-applications.installedFrom`); guide the user through each missing connector (consent/setup URLs — secret values never transit the chat).
4. `catalog-copy-files({ schema, name, version, files: [{from, to}] })` — server-side S3 copy from the package into the target app's folder. Use it for pre-built artifacts (e.g. `frontend/deployment.zip` → `backend/deployment.zip` then `deploy-backend`) — it works even from environments that can't build or upload files.

## Publishing: share an app (org owners/admins only)

**Public visibility is curated**: publishing with `visibility: "public"` (or switching a listing to public) additionally requires your org's curation flag, toggled per organization in the dilaya.eu admin (Organizations page). The connector verifies it live against dilaya.eu at publish time (fail-closed). Without it you get `CATALOG_FORBIDDEN` — `visibility: "org"` (private to your org) always remains available.

Publication is an act of **authorship**, not a snapshot: you WRITE the package from the live app (`describe-schema`, `get-skill`, `list-cowork-dashboards`, …). **NEVER include user data** — structure + generic seed rows only. Steps:

1. **Write `dilaya.md`** — the whole install story, addressed to the installing agent:
   - what the app is, for whom, how it works (precise, no fluff);
   - a `## Installation` section with numbered, executable steps: `create-schema` (suggest the catalog name), the full DDL, skills to `save-skill` (either inline or referencing packaged `skills/*.md` files), dashboards/agents to create (`save-cowork-dashboard`, `set-agent` — remind that set-agent returns an install prompt to hand over), frontend deployment if any (`catalog-copy-files` the packaged `deployment.zip`, then `deploy-backend` + `test-backend`), crons (`set-app-cron`), and a final verification step;
   - which connectors the app needs and why (mirror of `requires`). For every OPTIONAL connector, describe the without-it behavior in the doc itself: what the app does when it's absent (feature simply missing) and/or how the installer can adapt (e.g. "sans Zoom : le RDV est confirmé sans lien de visio ; adaptable en remplaçant l'appel `mcp.call('zoom', …)` du handler par un autre MCP de visio que l'utilisateur possède, p. ex. Google Meet").
   The publish gate refuses a doc under 500 chars, without headings, or without an Installation section.
2. **Stage the files** — `catalog-upload-urls({ name, version, files: ["dilaya.md", ...] })` (claims the name for your org; name rules = app-name rules, global across Dilaya), then HTTP PUT each file.
3. **Publish** — `catalog-publish({ name, version, files, title, summary, visibility, tags?, requires?, notes? })`. `requires` is the machine-readable requirements block ({ telegram/mail/auth/frontend: "required"|"optional", secrets: [{name, description}], mcp_connections: [{name, why, tools, url?, target?: "backend"|"claude"|"both", level?: "required"|"optional", without?}], environment }) — it drives the install pre-flight and the catalog badges, so declare it carefully. An MCP connection defaults to required; mark it `level: "optional"` when the app installs and works without it, and say in `without` what is lost / how to adapt (the install prompt tells the installing agent to offer the choice and, if declined, to install without it following that guidance). `target` says WHERE the connection must exist — do not conflate the two: `backend` (the default) = the app's Lambda calls the server at runtime via `add-mcp-connection` + `grant-mcp-tools`; `claude` = the USER connects the server in Claude itself (claude.ai Settings → Connectors) so the chat and the cowork/code agents can use its tools — `add-mcp-connection` is NOT involved; `both` = both places. ALWAYS set `url` (the MCP server URL) when the target server has a stable public URL — it lets the installing agent guide the user without asking; omit it only when the URL is per-user/per-deployment (publish then returns a warning, and the install prompt tells the agent to ask the user for it). The install prompt makes the installing agent confirm EACH MCP connection with the user before adding it (sensitive; usually requires an authentication) — never silently. Versions are strict `x.y.z` and immutable once published (`CATALOG_VERSION_EXISTS` on republish — bump instead).
4. **Iterate** — `catalog-listing-update` patches metadata (visibility/title/summary/tags) without a new version; `catalog-unpublish({ name, version?, confirm: true })` removes a version or the whole listing (already-installed apps are unaffected).

## The catalogue console

`setup-catalog-console()` (org-scoped, idempotent) seeds the `catalogue` Cowork dashboard into the reserved `_console` app — browse/search the catalog, see each app's requirements, and copy its install prompt in one click. After seeding, hand the `dilaya-dashboard-install` skill to the user (same flow as the management console).

## Tools

- `catalog-search({ q?, tag?, mine? })` — listings visible to your org (`mine: true` = yours incl. staged drafts). Org-scoped, read-only.
- `catalog-get({ name, version? })` — listing + versions + `dilaya.md` inline + file download URLs.
- the `dilaya-catalog-install` skill — the self-contained install prompt (pre-flight + doc). Default: latest.
- `catalog-copy-files({ schema, name, version, files: [{from,to}] })` — server-side copy package → target app folder. Package files only, own-app destination only.
- `catalog-upload-urls({ name, version, files })` — stage a version (presigned PUTs; claims the name; refused for published versions). Owners/admins.
- `catalog-publish({ name, version, files, title, summary, visibility, tags?, requires?, notes? })` — validate + freeze the immutable version + update the listing. Owners/admins.
- `catalog-listing-update({ name, visibility?, title?, summary?, tags? })` — metadata-only patch. Owners/admins.
- `catalog-unpublish({ name, version?, confirm: true })` — remove a version / the listing (files + index). Owners/admins.
- `setup-catalog-console()` — seed/refresh the `catalogue` Cowork dashboard in `_console`.
