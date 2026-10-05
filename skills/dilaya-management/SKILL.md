---
name: dilaya-management
description: "Use when looking across all the apps of a Dilaya organization: status, archive, usage, the management console."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# Managing all apps — cross-app status, archive, usage, and the console

A Dilaya "app" is its own SQLite database + file folder; the org's app inventory and lifecycle live in a DynamoDB **registry**. These org-scoped tools give you a board over ALL your apps at once — status, reversible archive, consumption metering, and a ready-made interactive **management console** — without touching any single app's data. The status tools you call directly; the console you seed once with `setup-management-console`.

## Sections

This is the table of contents, not the recipe: read the sections a step needs before acting on it.

- [Cross-app status: `list-applications`](references/status.md) — `status`
- [Interactive console: `setup-management-console`](references/console.md) — `console`
- [Reversible archive: `archive-app` / `unarchive-app`](references/archive.md) — `archive`
- [Consumption metering: `get-usage-report`](references/metering.md) — `metering`
- [Tools](references/tools.md) — `tools`
- [Who in the organization can use an app](references/access.md) — `access`
- [Platform-admin analytics (admin orgs only)](references/admin-analytics.md) — `admin-analytics`
