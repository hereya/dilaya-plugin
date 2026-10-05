# Customer-owned custom domain (BYOD)

An app's frontend can ALSO be served on a **domain the customer owns** (e.g. `commandes.acme.com` or `proposition.acme.fr`), in addition to the path URL and the vanity host. Unlike the vanity host (zero-config), a customer domain needs the customer to **add DNS records at their own provider** — the tools give you the exact records; nothing sensitive is exchanged.

Under the hood the organization gets ONE dedicated CloudFront distribution (created lazily at its first custom domain; no fixed cost) and ONE certificate covering all its custom domains. You never manage those directly.
