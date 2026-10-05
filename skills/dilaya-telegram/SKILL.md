---
name: dilaya-telegram
description: "Use when attaching a Telegram bot to a Dilaya app, or sending and reading its messages."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# Telegram bot

Attach a Telegram bot to an app so the app's **agent** can send and receive Telegram messages. Everything is driven through the **MCP tools** below (send/setup/RBAC/webhook) — there is no runtime `telegram` helper and no per-app-backend push path. Inbound messages are stored in `_telegram_messages` (a table in the app's own SQLite database); the bot's key is typed by the user into a Dilaya web page and kept server-side, never in the chat. No domain setup is needed: the bot's webhook is registered for you, and `attach-telegram` returns its address as `webhook_url`.

## One-time setup

1. Create a bot with **@BotFather** in Telegram → it gives you the **bot's key**.
2. `attach-telegram({ schema, owner: "<owner telegram id>" })` — creates the `_telegram_messages` table (in the app's SQLite database) and wires up the static public webhook route. **Bots are private by default**: pass `owner` with the owner's Telegram numeric id — ASK THE USER for it if you don't have it (they can get it from **@userinfobot**). The owner gets the `owner` role. Only make the bot open to everyone on the user's explicit request, by passing `visibility: "public"` (then `owner` is optional but recommended).
3. `get-telegram-setup-url({ schema })` — returns a single-use, 15-min URL. The user opens it in a browser and pastes the bot's key there, never in the chat; Dilaya keeps it server-side **and the webhook is registered with Telegram automatically** — no dashboard step.

Check anytime with `get-telegram-status({ schema })` (shows bot username, allowlist, counts). Re-register the webhook if needed with `register-telegram-webhook({ schema })`.

## Sending (via MCP)

- Text: `send-telegram({ schema, to: "123456789", text: "Hello!" })` — `to` is a numeric chat id (or @channelusername).
- Rich: pass a `message` object:
  - photo: `{ type: "photo", photo: "<public image URL or file_id>", caption: "…" }`
  - document: `{ type: "document", document: "<public file URL or file_id>" }`
  - location: `{ type: "location", latitude: 48.85, longitude: 2.35 }`
  - buttons / markdown: a text message with extras, e.g. `{ type: "text", text: "Pick one", reply_markup: { inline_keyboard: [[{ text: "Yes", callback_data: "yes" }]] } }`
  - escape hatch: `{ type: "raw", method: "sendDice", params: {} }`
- A bot can only message users who have **started it** (`/start`) or chats it belongs to — Telegram doesn't allow cold outreach.
- Download an inbound file with `get-telegram-file({ schema, file_id })` (returns a presigned URL).
- Show "is typing…" before a slow reply with `send-telegram-chat-action({ schema, to, action: "typing" })`. It clears when your message arrives or after ~5s.

## Receiving & replying — use an autonomous agent (no polling)

Inbound Telegram messages surface through the **agent loop**, not a backend push. On each inbound update the webhook stores the message in `_telegram_messages` and **appends it to the agent's notification inbox** (a low-level `has_new` flag is also set), so the agent's runner wakes it on a DM — it never polls the database on a timer. Wire this up with an **autonomous agent** (see `set-agent` / the `dilaya-agent-setup` skill): on wake the agent reads the new rows from `_telegram_messages`, replies with `send-telegram` (optionally `send-telegram-chat-action` first for a typing hint), and acknowledges the notifications. (The low-level `check-telegram-new` / `clear-telegram-new` flag still exists for bespoke poll loops, but you don't need it — the notification is the wake.)

There is **no runtime `telegram` helper and no per-app-backend push path**: inbound messages do not invoke a frontend `handler.js`, and `require("hereya")` exposes no `telegram` namespace, so a frontend handler can neither send nor receive Telegram. All Telegram send/receive goes through the MCP tools + the agent loop. (`set-telegram-notifications` / `attach-telegram`'s `notify_backend` only record a preference — backend Lambdas are not wired in this connector.)

## Access control & RBAC: private by default

- **private (default)** — only allow-listed users/chats may interact. Inbound from anyone else is **dropped** (not stored, not delivered); `send-telegram` to a non-allowed target is **rejected**. Seed it at attach time via `owner`. Always default to this.
- **public** — open to anyone who starts the bot. Only choose this when the user **explicitly asks** for an open bot; pass `visibility: "public"` to `attach-telegram` (or `set-telegram-access`).

**Roles (RBAC):** every allow-listed user has a role — `owner` (full control), `admin` (elevated), or `member` (basic). Non-members on a public bot are `guest`. The role is recorded on every inbound message (the `sender_role` column in `_telegram_messages`) so your agent can authorize actions. Membership gates access; the role is what you check for permissions. `get-telegram-status` lists each allow-listed user with their role.

Entries are numeric user/chat ids or @usernames (a user finds their numeric id via **@userinfobot**). Manage with `set-telegram-role({ schema, user, role })`, `remove-telegram-user({ schema, user })`, and `set-telegram-access({ schema, visibility })`. `get-telegram-status` lists `users` with their roles.

## Storage

Every inbound and outbound message is written to `_telegram_messages` (`direction, update_id, tg_message_id, tg_chat_id, chat_type, tg_from_id, tg_username, sender_role, msg_type, text, media, status, error, raw, created_at`). Query it with `query`, join it (within the same app), or build a cowork dashboard on it. Dropping the app removes it.
