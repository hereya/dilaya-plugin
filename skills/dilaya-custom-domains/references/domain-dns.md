## Your own DNS records on a bought domain (outside mailbox, verification)

The organization's **OWNER** can add their own records to a domain bought through Dilaya, for example to run Zoho Mail on it, next to the records Dilaya keeps for the site and the app mail.

```
list-dns-records({ domain })                                # every record; managed_by_dilaya marks the ones off limits
set-dns-record({ domain, name, type, value, ttl? })         # ADD one value; the others at that name stay
delete-dns-record({ domain, name, type, value })            # remove ONE value; the record goes with its last value
```

- `name` is the FULL name (`zmail._domainkey.moncafe.fr`), or `@` for the domain itself. A provider's short form (`zmail._domainkey`) must be completed with the domain.
- Types: TXT, MX, CNAME, SRV. Values: TXT the plain text (no need for quotes or for cutting long values: both are handled); MX `"10 mx.zoho.eu"`, one call per server; CNAME a host name; SRV `"10 5 5060 sip.example.com"`. TTL 300 unless set; it applies to all the values at that name and type.
- **Zoho Mail, in order**: the verification TXT at `@`, the MX records at `@`, the SPF TXT at `@`, then the DKIM TXT at `zmail._domainkey.<domain>`. The app mail Dilaya sends keeps working: it has its own DKIM and bounce records, and leaves the root SPF and MX free.
- **Before changing MX or SPF on a name already receiving mail, say what it means and get an explicit yes**: mail follows the MX records from then on.
- **One SPF only**: a second `v=spf1` at one name is refused (`DNS_SPF_DUPLICATE`). Merge the senders into one value: delete the old one, add the merged one.
- **Refused to protect the site and the mail** (`DNS_NAME_PROTECTED`): at the root anything but TXT and MX, certificate validation records, Dilaya's mail records (`<digits>pm._domainkey`, `pm-bounces`), and any name serving a Dilaya site. `DNS_TYPE_NOT_ALLOWED`: CAA (it can block the certificate's renewal), A, AAAA, NS, SOA, ALIAS.
- `DNS_CNAME_CONFLICT`: a CNAME must be alone at its name, both ways. `DNS_NAME_OUTSIDE_ZONE`: the name is not under the domain. `DNS_CONCURRENT_CHANGE`: the name kept changing, nothing was written; read it back and retry. `DOMAIN_NOT_FOUND`: not bought by this organization through Dilaya, its DNS is held elsewhere. `ORG_OWNER_ONLY`: relay that the owner can do it.
- A name used here is not free for a site: attaching it later with `set-custom-domain` replaces its CNAME, and Route 53 refuses the site's record while a TXT or MX sits at that name. Delete those first.
- These tools are listed with the domain-purchase option.
