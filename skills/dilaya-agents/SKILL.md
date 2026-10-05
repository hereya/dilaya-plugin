---
name: dilaya-agents
description: "Use when automating a Dilaya app with an agent: a code agent on a local poller, or a scheduled task."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# Agents — automate an app with JUDGMENT (code poller + scheduled tasks)

An app can have **one code agent** (the singleton, name `main`) and **any number of scheduled agents** (named). Each is a prompt the connector stores with a wake signal; the connector never runs AI itself.

> **⚠️ Agents are for work that needs INTELLIGENCE — not for anything an algorithm can do.**
> An agent run costs LLM tokens and minutes of latency. Before creating one, check the
> deterministic alternatives (see the "Choosing the right pattern" section of the usage skill):
> - fixed-schedule deterministic work (reminders, reports, purges) → **app crons**
>   (the `dilaya-crons` skill) — the backend handler runs it, instantly, for free;
> - user-triggered multi-step workflows (confirm + email + calendar + Zoom…) → **inline handler**
>   with `mcp.call`/`mail.send` (`topic: "mcp"`, `"frontend"`) — no agent in the loop.
> Keep the agent for: interpreting free-form language (Telegram concierge), decisions with
> judgment (triage, exceptions, apologize-or-retry), building/evolving the app, and RESCUE when a
> deterministic path fails (reuse partial artifacts — never recreate).

## Two kinds

- **code** — a cheap, AI-free **local poller** (the `dilaya` CLI, `npm i -g dilaya-cli`) that wakes an interactive Claude session in tmux only when there's work, runs it to idle, then reaps it. Installed in **Claude Code at the project's repo root**. One per app (named `main`). Use it for an autonomous dev/ops agent that reacts to Telegram + task changes and can run long sessions.
- **scheduled** — a **scheduled task** (cloud) that runs the prompt on a cron, inside a **runtime** that owns the schedule: `chatgpt` (a ChatGPT scheduled task using the Dilaya app) or `claude-cowork` (a Claude Cowork scheduled task, the default). No poller, no token, no public route. Installed by pasting a prompt into the runtime. Many per app. Use it for recurring jobs that genuinely need judgment each run (editorial digests, triage sweeps) — NOT for deterministic schedules (those are app crons). On the wire a scheduled agent is `type: "scheduled"` (set-agent still accepts the historical `"cowork"` as an alias) — `runtime` says which product runs it; agents created before runtimes existed are `claude-cowork` and keep working unchanged.

## Create / update an agent

1. `set-agent({ schema, type: "code" | "scheduled", runtime?, prompt, name?, cron?, tz? })` (passing `runtime` implies scheduled; `"cowork"` is an accepted alias) — create or update (idempotent). On CREATE it returns a ready **copy-paste install prompt** and **where to paste it** (Claude Code for code; the runtime — ChatGPT or Claude Cowork — for scheduled).
2. the `dilaya-agent-setup` skill — step-by-step setup **by type** (the type is the install guard: a code agent can't be installed as a scheduled task or vice-versa), and for a scheduled agent **by runtime** (`runtime` shows another runtime's steps, e.g. to move it).
3. code only: `get-agent-setup-token({ schema })` — a single-use 15-min token the CLI exchanges for a long-lived poll token (the poll token never enters the conversation).

> **⚠️ Updating an existing agent is a TWO-step change — the stored spec is only a copy.** What actually
> runs lives elsewhere: a **scheduled** agent runs as a **scheduled task in its runtime** (which holds its own
> copy of the prompt + schedule), a **code** agent runs from the **command file on the poller's machine**
> (`.claude/commands/dilaya-agent.md`). After `set-agent` on an existing agent, the response returns
> `updated: true` plus a `replicate` field with the exact follow-through: scheduled → update the scheduled
> task's instructions/schedule in its runtime (a paste-ready prompt is included); code → rewrite the command file
> and re-run `dilaya agent install`. **Relay that to the user immediately** — the change is not live until
> it's replicated. Same on `delete-agent`: a scheduled agent's task must also be deleted in its runtime. Changing `runtime` on an existing agent returns `movedFrom`: install it in the new runtime AND delete the old task, or it runs twice.

Inspect with `list-agents({ schema })` / `get-agent({ schema, name? })` (enabled, runtime, wake mode, run lifecycle).

## Scheduled-agent run contract (every runtime)

Each tick: `begin-agent-run({ schema, org, name })` FIRST → the work in the prompt, with `org` on every app call → the agent's own inbox (`list-notifications` / `ack-notifications` with `name`) → `end-agent-run({ schema, org, name, runId })` LAST (→ idle) → end the run. No `set-agent-mode`, no `await-work` loop: the runtime fires the next tick. The next tick then says `resumed_from_idle`; `recovered_from_crash` means the previous tick stopped before `end-agent-run` (an agent installed before it existed says it on every tick — re-run `set-agent` and replicate the new run body). Runtime-specific steps (how the product creates a scheduled task, what an unattended run there cannot do) come from the `dilaya-agent-setup` skill.

## Validate a ChatGPT scheduled agent (read-only)

1. `set-agent({ schema, org, name: "check-read", runtime: "chatgpt", cron: "0 8 * * *", tz: "Europe/Paris", prompt: "READ ONLY. Count the rows of one table with query and report the number. Never call execute, bulk-insert, send-mail, send-telegram or any other tool that writes." })`.
2. Paste the returned `installPrompt` into a ChatGPT chat with the Dilaya app enabled; it creates the scheduled task, then run it once by hand.
3. Check: `get-agent({ schema, org, name: "check-read" })` shows a `run_started_at`, the task's answer carries the row count, and the app's data is unchanged (same count, no new rows).
4. Clean up: `delete-agent({ schema, org, name: "check-read" })`, then delete the task in ChatGPT.

## Code-agent runtime contract (what the prompt must do each wake)

**Connector readiness comes before `begin-agent-run`:** on a fresh session this app's tools are deferred MCP tools loaded via ToolSearch, and the tool index can lag the connection by a few seconds. An empty first ToolSearch is a transient indexing race, **not** a disconnected connector — retry ToolSearch ~5x over ~20s; never declare the connector down or end the turn without retrying. Then: `begin-agent-run` FIRST (returns why it woke: first_run | resumed_from_idle | recovered_from_crash) → do all available work → optionally `await-work` for a cheap back-and-forth (~20 s server-side block, no tokens) → before idle `ack-notifications` what it handled and `set-agent-mode({ mode: "continue" | "new" })` to pick the next wake → end the turn. Never sleep/busy-wait; the poller wakes it on work.

## Scheduled proactive work (code agent)

`set-agent-schedule({ schema, cron, tz, prompt })` / `list-agent-schedules` / `remove-agent-schedule` — a cron tick injects a notification carrying the prompt, draining through the same notifications path as reactive work. No extra infra.

## Lifecycle bookends

`begin-agent-run` (→ running) and `set-agent-mode` (→ idle) are symmetric bookends — for a scheduled agent the closing one is `end-agent-run` (→ idle, no wake mode; with `runId` it never closes a newer run). A run killed before its closing call stays `running`, so the next `begin-agent-run` reports `recovered_from_crash` → reconcile from the DB before taking new work. `delete-agent({ schema, name? })` removes the spec + poll token + public route (also run `dilaya agent uninstall --schema <schema>` on the poller machine).
