---
name: dilaya-use-app
description: "Use before reading or changing the data of an existing Dilaya app (queries, inserts, files, skills)."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# How to Use an Existing App on Dilaya

**IMPORTANT:** Before performing any operation on an app, ALWAYS start by checking available skills with `list-skills()`. Skills contain critical context — table structures, query patterns, business rules, stored scripts, and templates — that prevent you from making mistakes or duplicating work. Never skip this step, even if you think you already know the app's structure.

## Steps

### 1. Discover available apps (always do this first)
```
list-skills()
```
Returns all skills across all apps, with schema name, skill name, and description. An app may have multiple skills (e.g., "main", "cost-analysis", "reporting") — load all relevant ones before proceeding.

### 2. Load the skill(s)
```
get-skill({ schema: "recipes", name: "main" })
```
Returns the skill content (instructions) AND the full schema structure (tables, columns, types, constraints) in one call. A long skill comes in pages: when `has_more` is true, read the next page with `offset: next_offset` before acting on any of it.

### 3. Follow the skill instructions
The skill tells you:
- What the app does
- What tables exist and what each column means
- Example queries for common operations
- Business rules and validation logic
- File storage conventions

### 4. Call the app's actions first
When `get-skill` shows `backend_actions`, a treatment exists as code in the app — the same code its pages use, when it has any. If one covers the request, call it (`run-app-action`, or `run-app-action-write` for `write: true`, after the user's yes) instead of recomputing it in SQL. Details: the `actions` section of the `dilaya-frontend` skill.

### 5. Query and write data
Use `query` for reads (SELECT) and `execute` for writes (INSERT, UPDATE, DELETE). Table names are unqualified — one SQLite database per app, no schema prefix:
```
query({
  sql: "SELECT r.name, r.servings FROM recipes r WHERE r.category = :cat",
  params: { cat: "desserts" }
})
```

### 6. Semantic / vector search (sqlite-vec)
Every app database has the **sqlite-vec (vec0)** extension preloaded — use it for semantic search / RAG over embeddings. Store vectors in a vec0 virtual table and run KNN:
```
execute({ sql: "CREATE VIRTUAL TABLE recipe_vectors USING vec0(recipe_id INTEGER, embedding float[1024])" })
execute({
  sql: "INSERT INTO recipe_vectors (rowid, recipe_id, embedding) VALUES (:id, :rid, :v)",
  params: { id: 1, rid: 42, v: "[0.12, -0.03, ...]" }   // vector = JSON array as TEXT
})
query({
  sql: "SELECT recipe_id, distance FROM recipe_vectors WHERE embedding MATCH :q ORDER BY distance LIMIT 10",
  params: { q: "[0.11, -0.02, ...]" }
})
```
Rules: the dimension (`float[N]`) is FIXED per table — pick it from your embedding model (e.g. 1024, 1536) and keep every vector that size. Add plain metadata columns (e.g. `recipe_id`) to map hits back to source rows. Brute-force KNN is fast well past 100k vectors; no index tuning needed. `SELECT vec_version()` confirms availability.

**Prefer the NATIVE embedding tools when they're enabled for your org**: `embed` / `vector-search` / `vector-index-list` / `vector-index-drop` generate the vectors server-side on Dilaya's platform key — no backend handler, no provider secret. See the `dilaya-vectors` skill. The raw vec0 SQL above remains right when you bring your own embeddings (custom model, backend-generated vectors).

### 7. Use files if needed
```
get-upload-url({ path: "recipes/photos/tarte-tatin.jpg", content_type: "image/jpeg" })
list-files({ path: "recipes/photos" })
get-download-url({ path: "recipes/photos/tarte-tatin.jpg" })
```
**Your environment blocks the transfer?** Both link tools also return `fallback_url`. Give it to the person: they open it in their own browser and upload (or download) the file themselves — the bytes go straight to storage. After an upload, check it really arrived with `get-upload-status({ path, transfer_id })` (`received` / `waiting` / `other_content`) — never rely on the person's word alone. After a download, they can attach the file to the conversation. Nothing guarantees their browser is not blocked too.

## Tips
- If no skill exists for a schema, use `describe-schema` to see the raw structure
- You can create additional skills for an app (e.g., a "cost-analysis" skill alongside the "main" skill)
- Each app is its own SQLite database — you CANNOT JOIN across apps; keep every query within one app

## Showing data visually

There is ONE way to give an app a visual surface: a **frontend**
(`enable-frontend` / `deploy-backend`) — a standalone web app people open in a
browser at its vanity host `https://<app>-<orgslug>.<contentDomain>/` (or a custom domain), with no AI in the
loop. They visit the URL, optionally log in with email OTP, and use the app
directly. See the `dilaya-frontend` skill.

In the conversation itself, show data the ordinary way: run the query and
answer with a table, a list or a short summary. The in-chat HTML views
(`save-view`/`get-view`) that used to live here were decommissioned on
2026-09-04 — do not look for them, and do not tell a user they exist.

## Accelerating Recurrent Tasks with Stored Scripts and Templates

When you notice a task pattern that recurs (e.g., generating a report, transforming data, producing a document), store reusable scripts and templates in S3 file storage so you can reuse them next time instead of rebuilding from scratch.

### When to store
- You've built a multi-step workflow the user is likely to repeat (weekly report, invoice generation, data import pipeline, etc.)
- You've crafted a complex SQL query, prompt template, CSV template, or HTML template that took significant effort
- The user explicitly asks you to "remember how to do this" or "make this faster next time"

### How to store
1. Upload the script or template to a well-known path under the app's S3 folder:
```
get-upload-url({ path: "{schema}/scripts/{script-name}.sql", content_type: "text/plain" })
get-upload-url({ path: "{schema}/templates/{template-name}.html", content_type: "text/html" })
get-upload-url({ path: "{schema}/templates/{template-name}.csv", content_type: "text/csv" })
```
2. Upload the content via the presigned URL.
3. Update the app's skill to document what's stored and when to use it:
```
save-skill({
  schema: "myapp",
  name: "main",
  content: "... existing skill content ...\n\n## Stored Scripts & Templates\n- `myapp/scripts/weekly-report.sql` — query for the weekly KPI report\n- `myapp/templates/invoice.html` — HTML invoice template with {{placeholders}}\n\nWhen the user asks for the weekly report, download and execute weekly-report.sql instead of rebuilding the query."
})
```

### Conventions
- Scripts: `{schema}/scripts/{name}.{ext}` (e.g., `.sql`, `.py`, `.sh`)
- Templates: `{schema}/templates/{name}.{ext}` (e.g., `.html`, `.csv`, `.md`)
- Always document stored files in the skill so future agents know they exist
- When reusing a stored script/template, download it via `get-download-url`, adapt if needed, then execute
