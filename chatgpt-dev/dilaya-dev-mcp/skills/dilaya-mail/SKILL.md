---
name: dilaya-mail
description: "Use when a Dilaya app must send transactional email (receipts, notifications, confirmations)."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# Transactional email (mail)

Send transactional email (receipts, notifications, magic links, confirmations) from an app via a dedicated per-app Postmark server. This is **separate** from `enable-auth` — mail is its own feature — though the two **share the app's sender domain** (`<app>-<orgslug>.<contentDomain>` — the same host as the app's vanity URL), auto-provisioned the first time either is enabled.

## Flow

`enable-mail` (once) → `send-mail` → `list-mail` (audit). Tear down with `disable-mail`.

## 1. Enable mail

```
enable-mail({ schema: "myapp" })
# → { from_email: "noreply@myapp-<orgslug>.<contentDomain>", mail_enabled: true }
```

Provisions a dedicated per-app Postmark server and verifies the sender domain (find-or-create — reuses the domain `enable-auth` already created, or creates it). Idempotent. **The sender domain can take ~1 min to verify the first time** — the very first send in that window may be delayed; retry if it errors. You do NOT need `enable-auth` or a frontend to use mail.

## 2. Send

```
send-mail({ schema: "myapp", to: "marie@example.com", subject: "Your receipt", html: "<h1>Merci!</h1>", text: "Merci!" })
# → { message_id: "..." }
```

- `to`, `subject`, and **at least one of** `html` / `text` are required (else `INVALID_BODY`).
- The `From` is fixed to the app's provisioned `noreply@…` sender — you cannot override it.
- Transactional (outbound) stream only — no bulk/broadcast. Requires `enable-mail` first (else `MAIL_NOT_ENABLED`).
- `RECIPIENT_REJECTED`: Postmark refuses that address (invalid, or inactive after a hard bounce / spam complaint). Fix the address; retrying will not help.

**Attachments** — reference files already in the org's storage (upload with `get-upload-url` first if needed):

```
send-mail({ schema: "myapp", to: "marie@example.com", subject: "Facture", html: "<p>Ci-joint.</p>",
            attachments: [{ path: "myapp/invoices/2026-07.pdf" }] })
```

- Each attachment: `{ path, name?, content_type?, content_id? }` — `path` is the org-storage path (same as the file tools / `list-files`); `name` defaults to the basename, `content_type` is inferred from the extension.
- Max **10 files, 7 MB total** per message (`ATTACHMENT_TOO_LARGE` / `TOO_MANY_ATTACHMENTS`); Postmark caps the whole message at 10 MB encoded.
- Inline images: set `content_id: "logo"` and reference it in the html as `<img src="cid:logo">`.

## 3. Audit

```
list-mail({ schema: "myapp", limit: 50 })   # recent sends, newest first (default 50)
```

Every send is logged to `_mail_log` (`to_email, subject, message_id, status, error, created_at`) — `status` is `sent` or `error`. Query it with `query`, or build a cowork dashboard on it.

## Send from a frontend handler (runtime helper)

A deployed **frontend handler** can send mail itself — no agent in the loop — via the runtime layer's `mail` helper (same fixed sender, same `enable-mail` requirement). It pairs with the `frontend` topic:

```js
const { mail, parseRequest } = require("hereya");

exports.handler = async (event) => {
  const req = parseRequest(event);
  const { message_id } = await mail.send({
    to: "marie@example.com",
    subject: "Your receipt",
    html: "<h1>Merci!</h1>",
    text: "Merci!",
  });
  return { statusCode: 200, headers: { "content-type": "text/html" }, body: "sent " + message_id };
};
```

`mail.send({ to, subject, html?, text?, attachments? })` → `{ message_id }`. It **throws** if mail was never enabled (`mail is not enabled for app '<app>'. Ask the app's agent to run enable-mail first.`) — run `enable-mail` once first. The From is the same fixed `noreply@…` sender and cannot be overridden. This lean handler path does **not** write to `_mail_log` (only the MCP `send-mail` tool logs).

Handler attachments take `{ path }` (a file in THIS app's storage, app-relative — e.g. `"reports/q4.pdf"`) **or** `{ content, name }` (base64, for files generated in the handler), plus optional `content_type` / `content_id`. Same limits: 10 files, 7 MB total.

## Teardown

`disable-mail({ schema: "myapp", confirm: true })` deletes the per-app Postmark mail server + SSM token and clears the config (the `_mail_log` history is kept). The shared sender domain is left in place (per-app auth may still use it).
