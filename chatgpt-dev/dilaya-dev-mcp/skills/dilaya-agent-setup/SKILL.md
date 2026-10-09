---
name: dilaya-agent-setup
description: "Use when installing, moving or reinstalling an agent of a Dilaya app (after set-agent, or when the user asks how to set one up)."
---

# Set up a Dilaya agent

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

1. Read the agent: `get-agent({ schema, name })` (`name` defaults to `main`, the code agent). It returns its `type` (`code` or `scheduled`), `prompt`, `runtime`, `cron` and `tz`. An agent that does not exist yet is created with `set-agent` first.
2. A **code** agent (a local poller that wakes Claude Code on a developer's machine) is not installed from this plugin: it is a developer setup, done in Claude Code connected to `https://app.dilaya.eu/mcp`. Tell the user so.
3. A **scheduled** agent → the file of its `runtime`: `claude-cowork` → [references/scheduled-claude-cowork.md](references/scheduled-claude-cowork.md), `chatgpt` → [references/scheduled-chatgpt.md](references/scheduled-chatgpt.md). To move it to another runtime, use that runtime's file.

Replace every `<…>` placeholder with the value named inside it, and keep the rest verbatim.
