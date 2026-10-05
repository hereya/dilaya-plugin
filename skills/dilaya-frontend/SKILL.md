---
name: dilaya-frontend
description: "Use when giving a Dilaya app a website: enable it, write and deploy its handler, sign-in, static pages, staging, named actions."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# Per-app web frontend

Give an app a **standalone web page** real people open in a browser — no Claude, no AI in the loop. It is served by a **per-app Lambda** that runs Node handler code you write and reaches ONLY its own app's SQLite database + files (isolated — see the end). **A site is PRIVATE by default**: the first `enable-frontend` provisions passwordless e-mail login and the platform closes the site, unless you pass `public: true` — a showcase site, a landing page, a menu, anything meant for everyone, needs that explicitly (§5).

> **What a frontend is.** A real website other people visit on their own URL, with no AI in the loop — as opposed to answering in the conversation, which is what you do for the person you are chatting with.

## Sections

This is the table of contents, not the recipe: read the sections a step needs before acting on it.

- [App templates — check these FIRST (before writing any `handler.js` or HTML)](references/templates.md) — `templates`
- [Choosing the architecture — YOU decide, don't ask the user](references/architecture.md) — `architecture`
- [Dynamic pages, static sections — mix freely](references/mix.md) — `mix`
- [Flow (dynamic)](references/flow.md) — `flow`
- [1. Turn the frontend on](references/enable.md) — `enable`
- [2. Write the handler](references/handler.md) — `handler`
- [3. Package + upload](references/package.md) — `package`
- [4. Deploy + test](references/deploy.md) — `deploy`
- [5. Login — private by default, public on request (per-app auth)](references/login.md) — `login`
- [6. Static sections — pre-built pages served from the edge](references/static.md) — `static`
- [How sources travel between sessions — zip or git (the app remembers)](references/sources.md) — `sources`
- [Deployment versions & rollback](references/versions.md) — `versions`
- [Optional staging environment](references/staging.md) — `staging`
- [Actions — freeze an exact treatment in the backend, and CALL it](references/actions.md) — `actions`
- [Isolation](references/isolation.md) — `isolation`
