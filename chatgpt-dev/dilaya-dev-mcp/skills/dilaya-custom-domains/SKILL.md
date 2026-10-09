---
name: dilaya-custom-domains
description: "Use when putting a Dilaya app's website on its own address: a Dilaya host, the customer's domain, or a domain bought through Dilaya."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# App host (vanity domain)

Give an app's web frontend a **friendly vanity host** on the shared app-content domain, **in addition** to the path URL. Instead of only `https://<customDomain>/o/<orgId>/<app>/site/`, the app is also reachable at a **flat** host:

```
https://<app>-<orgslug>.<contentDomain>/     e.g. https://smartcal-novopattern.dilaya-apps.eu/
```

- `<app>` is the app name; `<orgslug>` is a stable, DNS-safe slug of your organization's friendly name (set with `set-name`); a single `-` joins them (app names never contain `-`, so it's unambiguous). (A single hyphen — not `--` — is used so the same host doubles as a clean email sender domain; `--` in a DNS label is punycode-reserved and gets mail rejected by strict filters like iCloud.)
- **Both URLs stay live** — the vanity host is additive; the path URL keeps working. (For SEO on custom domains, a canonical 301 between HOSTS is available — see `redirect_to` in the BYOD section below.)
- The **TLS certificate and DNS are platform-managed** — a single wildcard `*.<contentDomain>` covers every app/org, so there is **nothing to configure** (no cert request, no DNS record). Provisioning just updates an edge routing map and takes **~seconds** to go live.

> **Availability.** This only works when the deployment was configured with an app-content domain. If it wasn't, the tools return `CUSTOM_DOMAINS_NOT_CONFIGURED` — ask the platform admin to enable it.

## Sections

This is the table of contents, not the recipe: read the sections a step needs before acting on it.

- [Flow](references/vanity-flow.md) — `vanity-flow`
- [Manage](references/vanity-manage.md) — `vanity-manage`
- [Customer-owned custom domain (BYOD)](references/byod.md) — `byod`
- [Flow](references/byod-flow.md) — `byod-flow`
- [Canonical domain (`redirect_to`) — avoid duplicate content](references/byod-redirect.md) — `byod-redirect`
- [Manage](references/byod-manage.md) — `byod-manage`
- [Migrating a domain that is ALREADY live elsewhere](references/byod-migrate.md) — `byod-migrate`
- [Buying a NEW domain through Dilaya (billed org option)](references/buy-domain.md) — `buy-domain`
