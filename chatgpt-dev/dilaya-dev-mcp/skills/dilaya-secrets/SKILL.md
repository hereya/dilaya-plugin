---
name: dilaya-secrets
description: "Use when a Dilaya app's backend needs a third-party API key, entered by the user out of the chat."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# Integration secrets

Store a **3rd-party API key** (an OpenAI key, a Stripe key, a webhook secret, …) that an app's **web-frontend handler reads at runtime** to call an external service — no agent in the loop. The key is entered by the USER in a browser form and stored in **SSM SecureString**; it is **never** pasted into chat and is **never** visible to you or any MCP tool.

> **Frontend feature.** Secrets are read by a deployed frontend handler (`secrets.get`), so they pair with the `frontend` topic. An agent-only app that never deploys a handler has nothing to read them.

## Flow

`get-secret-setup-url` → hand the user the URL → they open it and enter the key → in your handler `const key = await secrets.get("<name>")`. Inspect with `list-secrets`; remove with `delete-secret`.

## 1. Declare the secret + get the setup URL

```
get-secret-setup-url({ schema: "myapp", name: "OPENAI_KEY", description: "OpenAI API key for the summariser" })
# → { url: "https://<customDomain>/o/<orgId>/myapp/secrets/setup?token=…", name: "OPENAI_KEY", expiresIn: 900 }
```

- `name` is an identifier — letters, digits and `_`, starting with a letter or `_` (max 64). It is what the handler reads it back by, so pick a stable, code-friendly name (e.g. `OPENAI_KEY`, `STRIPE_SECRET`).
- **Give the user the `url`** and walk them through opening it — it's single-use and expires in 15 minutes. **Never ask the user to paste the key into the chat.** Re-run to rotate the value (mint a fresh link).
- `description` (optional) is shown on the form so the user knows exactly which key to paste.

## 2. The user enters the value

They open the link, type the key into a single password field, and submit. The value goes **browser → SSM SecureString** at `/dilaya/<orgId>/apps/<app>/secrets/<name>`; it never touches this conversation. The link is single-use (a re-submit is rejected). Once submitted, `list-secrets` shows the secret as `configured: true`.

## 3. Read it in the frontend handler

The runtime layer exposes `secrets.get(name)` — reads the decrypted value from SSM (the per-app Lambda's IAM role can read ONLY this app's own `/secrets/*`):

```js
const { secrets, parseRequest } = require("hereya");

exports.handler = async (event) => {
  const req = parseRequest(event);
  const key = await secrets.get("OPENAI_KEY"); // throws a clear error if unset
  // → call OpenAI with `key`
  return { statusCode: 200, headers: { "content-type": "text/html" }, body: "…" };
};
```

`secrets.get` **throws** if the secret was never set (`secret '<name>' is not set for this app. Ask the app's agent to run get-secret-setup-url first.`) — so declare + have the user fill it before the handler relies on it. There is no write path in the handler: values are set only through the browser form.

## Inspect + remove

```
list-secrets({ schema: "myapp" })                                   # name / description / configured / created_at — NO values
delete-secret({ schema: "myapp", name: "OPENAI_KEY", confirm: true }) # removes the SSM value + metadata (idempotent)
```

`list-secrets` returns metadata only — it can NEVER return a value. `delete-secret` requires `confirm: true`. Dropping the app removes its secret values too.
