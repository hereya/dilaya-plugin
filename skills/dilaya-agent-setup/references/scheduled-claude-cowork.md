# Set up the "<name>" scheduled agent for "<schema>" — runtime: Claude Cowork

⚠️ This is a **scheduled** agent: Claude Cowork owns its schedule and runs it. There is NO local poller,
NO `dilaya` CLI, NO tmux, NO poll token. Each run calls this app's Dilaya tools directly —
your `mcp__<NAME>__*` tools (the exact prefix in your own tool names; confirm with `get-name`).

## Spec
- app: `<schema>` · org: `<org — the orgId the agent belongs to>` · agent: `<name>`
- cron: `<cron — from get-agent>` · timezone: `<tz — from get-agent>`
- Confirm it first: `get-agent({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>" })`.

## Install in Claude Cowork
1. Set this up **inside Claude Cowork**, NOT in Claude Code or on a local machine.
2. Create a **scheduled task in Claude Cowork** with cron `<cron — from get-agent>` in timezone
   `<tz — from get-agent>`. Use Cowork's scheduling (the schedule skill / scheduled-tasks).
3. The task's instruction is the **Run body** below, verbatim.

## Run body (the scheduled task's instructions, verbatim)
You are the "<name>" scheduled agent of the Dilaya app "<schema>". Each run:
1. FIRST call `begin-agent-run({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>" })` and keep its `runId`. Its `reason` is `first_run` on the
   very first run — do any one-time setup then — and `resumed_from_idle` after a run that ended normally.
   `recovered_from_crash` means the previous run stopped before step 4: check the app's data for work
   it left half done before starting this run's work.
2. Do the work described in the agent prompt below, with this app's Dilaya tools. Pass
   `schema: "<schema>"` and `org: "<org — the orgId the agent belongs to>"` on every app call.
3. Read this agent's own inbox: `list-notifications({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>", unreadOnly: true })`; after handling,
   `ack-notifications({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>", ids: [...] })` for ONLY what you handled.
4. LAST call `end-agent-run({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>", runId })` with the runId from step 1, then end the run.
   Do NOT call `set-agent-mode` and do NOT loop on `await-work` — the schedule fires the next run.

Agent prompt:
---
<prompt — the agent's `prompt` from get-agent, verbatim>
---

## Updating / removing
- Change prompt or cadence: `set-agent({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>", runtime: "claude-cowork", prompt, cron, tz })`,
  then update the scheduled task in Claude Cowork to match.
- Remove: `delete-agent({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>" })`, then delete the scheduled task in Claude Cowork.
- Another runtime: the `dilaya-agent-setup` skill shows its steps; record the move with
  `set-agent` (`runtime`) and delete the task here, so the agent never runs twice.
