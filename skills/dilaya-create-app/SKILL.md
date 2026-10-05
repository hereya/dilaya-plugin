---
name: dilaya-create-app
description: "Use when creating a new Dilaya app: its database tables, file folder and first skill."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# How to Create an App on Dilaya

An app on Dilaya is its own SQLite database + an S3 folder + one or more skills (instructions for using the app).

## Steps

### 1. Pick a name
Choose a lowercase alphanumeric name, starting with a letter — **no hyphens, no underscores**. Max 63 characters. Examples: `recipes`, `billing`, `contactmanager`, `orders2`.

This constraint exists because the app name is used verbatim in the frontend URL path segment (`/o/{orgId}/{app}/site/`) and as a DNS label in the per-app Postmark sender subdomain (`<app>-<orgslug>.<contentDomain>`). DNS labels don't allow underscores, and keeping the rule uniform avoids surprises at `enable-auth` time. (A handful of legacy apps pre-date this rule and keep working, but new apps must follow it.)

### 2. Create the schema
```
create-schema({ name: "recipes" })
```
This creates the app's SQLite database and an S3 folder for file storage.

### 3. Design your tables
Use `execute` to create tables. Table names are unqualified — one SQLite database per app, no schema prefix:
```
execute({
  sql: "CREATE TABLE recipes (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, servings INTEGER, prep_time INTEGER, cook_time INTEGER, category TEXT, created_at TEXT DEFAULT CURRENT_TIMESTAMP)"
})

execute({
  sql: "CREATE TABLE ingredients (id INTEGER PRIMARY KEY AUTOINCREMENT, recipe_id INTEGER REFERENCES recipes(id) ON DELETE CASCADE, name TEXT NOT NULL, quantity REAL, unit TEXT)"
})
```
SQLite types: use `INTEGER PRIMARY KEY AUTOINCREMENT` for ids, `TEXT` for strings, `INTEGER` (0/1) for booleans, `REAL` for decimals, `TEXT DEFAULT CURRENT_TIMESTAMP` for timestamps, and `TEXT` for JSON (query it with `json_extract`). FTS5 (full-text) and sqlite-vec `vec0` virtual tables (semantic/vector search — see the `dilaya-use-app` skill) are both available.

### 4. Insert seed data (optional)
Use `execute` for small inserts or `bulk-insert` for many rows:
```
bulk-insert({
  schema: "recipes",
  table: "ingredients",
  columns: ["recipe_id", "name", "quantity", "unit"],
  rows: [[1, "Flour", 500, "g"], [1, "Sugar", 200, "g"], [1, "Butter", 150, "g"]]
})
```
Inline `rows` caps at 10,000. For LARGE volumes, upload a CSV to org storage and import it by reference instead — the data never enters the conversation (max 20 MB / 200,000 rows per call):
```
get-upload-url({ path: "imports/ingredients.csv" })   // then HTTP PUT the CSV there
bulk-insert({ schema: "recipes", table: "ingredients", file: "imports/ingredients.csv" })
```
CSV rules: RFC 4180, UTF-8; the first row is a header naming the columns (or pass `columns` + `header: false`); `delimiter` "," (default) or ";"; values are coerced to the table's column types (INTEGER/REAL); empty fields insert NULL (`empty_as_null: false` for empty strings). The import is transactional — all rows or none; errors report the CSV line.

### 5. Write a skill
Save instructions that tell future agents how to use this app:
```
save-skill({
  schema: "recipes",
  name: "main",
  description: "Manage recipes with ingredients and portions",
  content: "... (see the `dilaya-write-skill` skill for guidance)"
})
```

### 6. Set up file storage (if needed)
```
create-folder({ path: "recipes/photos" })
```

The app is now ready. Any agent can discover it via `list-skills()` and load it via `get-skill()`.

### 7. Optional — expose a web frontend
If end users need to access the app from a browser (not just through Claude), follow the frontend workflow: the `dilaya-frontend` skill. **First reflex: call `list-app-templates`** — a curated starter (e.g. `template-astro-static`) can scaffold the whole site via `create-app-from-template` in one step, saving hours over writing a `handler.js` and pages by hand; only hand-build when no template fits. Then `enable-frontend` — a site is **private by default**: the first enable provisions passwordless e-mail login and the platform closes the site (a showcase site, a landing page or anything meant for everyone needs `public: true` explicitly); allow people in with `add-user`, and deploy a handler via `deploy-backend`. Apps without a frontend stay as pure data/skill surfaces — skip this step if the app is agent-only.

> **Choosing the architecture is YOUR job, not the user's.** Ask only business questions (what kind of site, what content, who visits) and pick the optimal setup yourself — a showcase site / landing / fixed-content site defaults to static-first serving on the app's host (see the `frontend` topic's "Choosing the architecture" section). Never surface technical trade-offs ("static or dynamic?", "cache", "vanity host") to a non-technical user; translate them into benefits ("faster", "ranks better on Google").
