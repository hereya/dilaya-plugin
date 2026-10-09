## Canonical domain (`redirect_to`) — avoid duplicate content

When SEVERAL hosts serve the same app (e.g. `acme.com` + `www.acme.com`), pick ONE canonical host and 301 the others to it:

```
set-custom-domain({ schema, domain: "acme.com", redirect_to: "www.acme.com" })
```

- The domain is provisioned normally (certificate, DNS, routing — TLS is needed to serve the redirect) but answers **301** to `https://<redirect_to><path>?<query>` instead of serving the app. Rendered at the edge (no Lambda), works in static and dynamic modes, cached 1h.
- `redirect_to` must be another host of the SAME app (an active custom domain or the vanity host); chains are refused (`REDIRECT_CHAIN`), unknown targets too (`REDIRECT_TARGET_NOT_FOUND`).
- Apex→www and www→apex both work — the canonical choice belongs to the customer.
- Re-run with `redirect_to: null` to clear (the domain serves again); omitting the field leaves the redirect unchanged. `list-custom-domains` shows it; removing a domain that others redirect TO is refused (`REDIRECT_TARGET_IN_USE`) until they're repointed.
- Handlers can read the actually-called host via `parseRequest(event).host` (the plain `Host` header never carries it behind the CDN) — useful for per-domain logic; `req.baseUrl` already reflects it.
