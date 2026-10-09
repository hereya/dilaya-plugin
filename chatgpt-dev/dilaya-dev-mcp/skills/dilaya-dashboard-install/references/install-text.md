# Set up the cowork dashboard "<name>" — paste this whole message into Claude Cowork

Paste this entire message into a Claude Cowork chat. It is addressed to the Cowork
agent and tells it everything; you (the human) add nothing else.

**About:** <description — from get-cowork-dashboard; drop this line if it has none>

---

**Claude Cowork agent: do BOTH of the following, end to end, without waiting for further instructions.**

## 1 — Generate the live artifact
Build the dashboard artifact from this prompt:

> <prompt — from get-cowork-dashboard, verbatim>

## 2 — Attach the source code back to the connector (source of truth)
Cowork artifacts are **local to this machine**, so once it's generated, upload the
artifact's source code to the Dilaya connector and attach it to this definition — this
is what lets the dashboard be reproduced on another machine. The source is usually a
single `index.html`, but may be several files (a folder) — **handle both**. Do this
yourself, immediately after generating, using these connector (MCP) tool calls:

1. Collect the relative paths of every source file (e.g. `index.html`, or
   `index.html` + `assets/app.js` + `assets/style.css`). Keep folder structure in
   the relative paths.
2. Get presigned upload URLs:
   `get-cowork-dashboard-source-upload-urls({ schema: "<schema>", name: "<name>", files: [ ...relative paths... ] })`
3. For each returned entry, HTTP **PUT** the file's bytes to its `upload_url`.
4. Record the manifest (the source of truth), naming the entry file:
   `attach-cowork-dashboard-source({ schema: "<schema>", name: "<name>", files: [ ...same paths... ], entry_file: "index.html" })`

Then confirm to the user that the dashboard is generated and its source is attached.

---

That's it — the dashboard's definition (prompt) + attached source now live in the
connector. On another machine, `get-cowork-dashboard({ schema: "<schema>", name: "<name>" })`
returns the prompt plus download URLs for every source file to reproduce it.

**Updating:** paste this message again — the Cowork agent re-generates from the prompt
and re-attaches the new file set, overwriting the previous source (simple overwrite, no
version history).
