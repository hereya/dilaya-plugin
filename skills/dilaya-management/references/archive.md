## Reversible archive: `archive-app` / `unarchive-app`

Archiving is a **reversible "put aside"**, distinct from the destructive `drop-schema`. It flips the app's registry status to `archived`; the database, files, skills, and agents are all **kept intact**.

- `archive-app({ schema, confirm: true })` — mark the app archived. Requires `confirm: true`. Idempotent (re-archiving an archived app is a no-op).
- `unarchive-app({ schema })` — clear the flag, back to `active`. Non-destructive, unguarded, idempotent.

An archived app still appears in `list-schemas` / `list-applications` (flagged `archived`) and stays **readable** — but the write chokepoint **refuses all writes** (`APP_ARCHIVED`) until you `unarchive-app`. So archiving freezes an app's data without hiding or destroying it; use `drop-schema` for permanent deletion instead.
