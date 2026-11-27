## Managing a bought domain: renewal and transfer out

The organization's **OWNER** can do from here exactly what dilaya.eu → Mon organisation → Domaines offers — the same two gestures, run by the same code on dilaya.eu, so the registrar and the Dilaya invoice always move together.

```
set-domain-auto-renew({ domain, enabled: false })    # stop renewing — at the registrar AND on the invoice
set-domain-auto-renew({ domain, enabled: true })     # renew again (both sides)
request-domain-transfer-code({ domain })             # the code is EMAILED to the registrant; never shown here
```

- **Before turning renewal OFF, say what it means and get an explicit yes**: the paid year runs to its end, nothing more is invoiced, and at the expiry date (`expires_at` in `get-domain-ownership`) the domain is **LOST** — the site and email on it stop, and someone else can buy the name. FR: « le domaine ne sera pas renouvelé : l'année payée va jusqu'au bout, plus rien n'est facturé, et à l'échéance le nom est perdu ». It is reversible until that date: `enabled: true` puts both the registrar and the invoice back.
- **The transfer code** goes to the registrant mailbox AS RECORDED AT THE REGISTRAR (`sent_to_masked` names it). Never ask the user to paste it into the conversation. If `transfer_locked` is true, the registry lock must be lifted before the new registrar can pull the name — the email explains it. Asking again is harmless (the code does not change). ICANN forbids a transfer within 60 days of registration (`earliest_transfer_at`).
- **Only the owner.** Anyone else gets `ORG_OWNER_ONLY`: relay that the organization's owner can do it, here or on dilaya.eu → Mon organisation → Domaines (FR: « seul le propriétaire de l'organisation peut le faire »). Members still read the facts with `get-domain-ownership`.
- Errors: `DOMAIN_NOT_FOUND` (not bought by this organization through Dilaya), `DOMAIN_NOT_REGISTERED` (purchase not finished), `NO_REGISTRANT_EMAIL`, `TRANSFER_CODE_MAIL_FAILED` (safe to retry), `RATE_LIMITED` (at most 3 transfer codes per domain per hour, 10 renewal changes — relay the wait), `DOMAIN_BUSY` (another change to that name is running, on dilaya.eu or elsewhere — nothing changed; wait a minute, read it back), `DOMAIN_ACTION_UNAVAILABLE` — for the renewal this means the outcome is UNKNOWN: read `auto_renew` back with `get-domain-ownership` before trying again.
- These two tools are listed with the domain-purchase option; an organization whose option was switched off keeps managing its names on dilaya.eu.
