## Manage

```
list-custom-domains({ schema? })                        # org's custom domains + status + the routing target
check-custom-domains({ schema? })                       # poll + promote; give schema to switch that app's mail + OTP senders
remove-custom-domain({ schema, domain, confirm: true }) # stop serving on the domain (vanity + path URL keep working)
```

- The first `set-custom-domain` of an org triggers a background distribution deploy (~5-10 min) — the DNS records are available immediately; the domain simply starts serving when both the deploy and the DNS are in place.
- Certificate renewal is automatic as long as the validation CNAME stays in place — tell the user to keep it.
- `CUSTOM_DOMAINS_NOT_CONFIGURED` → the deployment doesn't have the app-content/BYOD feature enabled; ask the platform admin.
- `CAA_BLOCKED` on `set-custom-domain` → the domain's CAA policy (often inherited through a CNAME still pointing at the previous hosting provider, e.g. Vercel/Netlify) forbids Amazon from issuing the certificate. Relay the error's fix verbatim: repoint the routing record (or add `issue "amazon.com"`), wait out the old TTL, retry.
- A certificate that still ends up `FAILED` (e.g. the CAA changed after the request) is recovered AUTOMATICALLY: the next `check-custom-domains` deletes it and requests a fresh one (`cert_reissued: true`, with `cert_failure_reason`/`cert_failure_hint` explaining what happened) — standing validation CNAMEs stay valid, so it converges without manual AWS action. `dns_pending` is verified against LIVE DNS (records already in place are not re-listed).
