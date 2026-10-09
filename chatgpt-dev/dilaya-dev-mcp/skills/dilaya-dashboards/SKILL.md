---
name: dilaya-dashboards
description: "Use when creating or reproducing a live dashboard of a Dilaya app."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# Cowork dashboards (live artifact + S3 source of truth)

A **cowork dashboard** is a **live artifact in Claude Cowork**. Its stored definition is just `name` + optional `description` + `prompt`; pasting the prompt into Cowork **generates** the artifact. The catch: Cowork artifacts are **local to the machine** that generated them. So the dashboard's **source code is uploaded to S3 and attached to the definition** as the **source of truth**, letting you reproduce the dashboard on another machine from the prompt + attached source.

## Storage model

- Definitions live in a dedicated per-schema table `_cowork_dashboards`.
- Source files are stored as **individual S3 keys + a manifest** (no zip) under a **stable prefix** `cowork-dashboards/<name>/`. **Simple overwrite, no version history** — re-attaching replaces the source set.
- Source is **one HTML file or several files (a folder)** — both are supported; keep folder structure in the relative paths.

## Tools

1. `save-cowork-dashboard({ schema, name, prompt, description? })` — create/update the definition (idempotent).
2. the `dilaya-dashboard-install` skill — the self-contained text to paste into Claude Cowork. Call it right after `save-cowork-dashboard` and hand the text to the user **unprompted** — don't wait to be asked. The text is addressed to the Cowork agent and tells it to generate the artifact AND upload + attach its source on its own; the user pastes it once and adds nothing.
3. `get-cowork-dashboard-source-upload-urls({ schema, name, files:[relpath…] })` — presigned PUT URLs for each source file; PUT the bytes, then attach.
4. `attach-cowork-dashboard-source({ schema, name, files:[relpath…], entry_file? })` — record the manifest as the source of truth (verifies each file is uploaded; overwrites the prior set).
5. `get-cowork-dashboard({ schema, name })` — the reproduce payload: definition (prompt) + entry file + presigned download URLs for every source file.
6. `list-cowork-dashboards({ schema })` / `delete-cowork-dashboard({ schema, name })`.

## Typical flow

The flow is **self-driving** — the user never has to spell out the steps:

1. `save-cowork-dashboard` (create/update the definition), then **immediately** the `dilaya-dashboard-install` skill and hand that text to the user — unprompted.
2. The user pastes that one text into Claude Cowork and adds nothing. It is addressed to the Cowork agent, which then does both halves itself: generate the artifact, then `get-cowork-dashboard-source-upload-urls` → PUT files → `attach-cowork-dashboard-source`.

To reproduce elsewhere: `get-cowork-dashboard` → download each file (preserving relative paths) → open the entry file.
