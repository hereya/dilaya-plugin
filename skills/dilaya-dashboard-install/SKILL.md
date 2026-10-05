---
name: dilaya-dashboard-install
description: "Use when installing or reproducing a live dashboard of a Dilaya app (after save-cowork-dashboard or a setup-*-console call, or on another machine)."
---

# Install a Dilaya dashboard

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

1. Read the dashboard: `get-cowork-dashboard({ schema, name })` — its `prompt` and `description` (the consoles live in the app `_console`).
2. Build the install text from [references/install-text.md](references/install-text.md). Replace every `<…>` placeholder with the value named inside it, and keep the rest verbatim.
3. Give that text to the user to paste where live artifacts are created (Claude Cowork) — or, if you can create live artifacts yourself, follow it directly.
