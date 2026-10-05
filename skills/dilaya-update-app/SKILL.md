---
name: dilaya-update-app
description: "Use when changing an existing Dilaya app's structure: new tables or columns, migrations, skill updates."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# How to Update an Existing App on Dilaya

## Steps

### 1. Load the current state
```
get-skill({ schema: "recipes", name: "main" })
```
This gives you the current skill instructions AND schema structure.

### 2. Make schema changes
Use `execute` for ALTER TABLE, CREATE TABLE, etc. — SQLite dialect, unqualified table names:
```
execute({ sql: "ALTER TABLE recipes ADD COLUMN difficulty TEXT" })
execute({ sql: "CREATE INDEX idx_recipes_category ON recipes (category)" })
execute({ sql: "CREATE TABLE tags (id INTEGER PRIMARY KEY AUTOINCREMENT, recipe_id INTEGER REFERENCES recipes(id), tag TEXT)" })
```
SQLite's `ALTER TABLE` supports `ADD COLUMN`, `RENAME`, and `DROP COLUMN` — but not changing an existing column's type. To change a type, add a new column and copy the data across (or rebuild the table).

### 3. Update the skill
After changing the schema, always update the skill to reflect the new structure:
```
save-skill({
  schema: "recipes",
  name: "main",
  description: "Manage recipes with ingredients, portions, and tags",
  content: "... (updated instructions reflecting new columns/tables)"
})
```

### 4. Verify
```
describe-schema({ schema: "recipes" })
```
Confirm the schema matches your intent.

## Destructive changes
- `DROP TABLE`: Use `execute({ sql: "DROP TABLE tags" })`
- `DROP SCHEMA`: Use the dedicated `drop-schema` tool (requires `confirm: true`). This also deletes all files and skills for the app. (`DROP SCHEMA` / `DROP DATABASE` via `execute` is rejected.)
- Column removal: `execute({ sql: "ALTER TABLE recipes DROP COLUMN difficulty" })`

Always update the skill after destructive changes.
