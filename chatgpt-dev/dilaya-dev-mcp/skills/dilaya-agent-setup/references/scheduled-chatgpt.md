# Set up the "<name>" scheduled agent for "<schema>" — runtime: ChatGPT

⚠️ This is a **scheduled** agent: ChatGPT owns its schedule and runs it. There is NO local poller,
NO `dilaya` CLI, NO tmux, NO poll token. Each run calls this app's Dilaya tools directly —
the tools of the Dilaya app connected to this ChatGPT account (`begin-agent-run`, `query`, `list-notifications`…).

## Spec
- app: `<schema>` · org: `<org — the orgId the agent belongs to>` · agent: `<name>`
- cron: `<cron — from get-agent>` · timezone: `<tz — from get-agent>`
- Confirm it first: `get-agent({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>" })`.

## Install in ChatGPT
1. Check the Dilaya app is connected and usable in this ChatGPT account: call
   `get-agent({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>" })` from this chat.
   A scheduled run can only use tools the account already has — if the call fails, fix the connection first.
2. Create a **scheduled task in ChatGPT** whose instructions are the **Run body** below, verbatim, recurring
   on cron `<cron — from get-agent>` in timezone `<tz — from get-agent>`. If ChatGPT's scheduler cannot
   express that cron exactly, pick the closest recurrence it offers and tell the user which one you chose.
3. Nobody is there to confirm anything during a scheduled run. Run the task once now WITH the user, and tell
   them which steps asked for a confirmation: an unattended run stops at each of those, so they decide whether
   to approve that tool for their scheduled tasks or to change the agent's prompt.
4. Check that run reached Dilaya: `get-agent` (step 1) now shows a `run_started_at`.

## Run body (the scheduled task's instructions, verbatim)
You are the "<name>" scheduled agent of the Dilaya app "<schema>". Each run:
1. FIRST call `begin-agent-run({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>" })` and keep its `runId`. Its `reason` is `first_run` on the
   very first run — do any one-time setup then — and `resumed_from_idle` after a run that ended normally.
   `recovered_from_crash` means the previous run stopped before step 4: check the app's data for work
   it left half done before starting this run's work.
2. Do the work described in the agent prompt below, with this app's Dilaya tools. Give
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
- Change prompt or cadence: `set-agent({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>", runtime: "chatgpt", prompt, cron, tz })`,
  then update the scheduled task in ChatGPT to match.
- Remove: `delete-agent({ schema: "<schema>", org: "<org — the orgId the agent belongs to>", name: "<name>" })`, then delete the scheduled task in ChatGPT.
- Another runtime: the `dilaya-agent-setup` skill shows its steps; record the move with
  `set-agent` (`runtime`) and delete the task here, so the agent never runs twice.
