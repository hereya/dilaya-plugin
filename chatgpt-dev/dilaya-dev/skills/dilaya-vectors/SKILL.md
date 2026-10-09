---
name: dilaya-vectors
description: "Use when a Dilaya app needs semantic search: embeddings and vector indexes."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# Vector search & embeddings (native tools)

Turn text into embedding vectors and run semantic (KNN) search INSIDE an app's SQLite db — no backend handler, no provider API key to manage. Embeddings run on **Dilaya's platform OpenAI key**; access is **enabled per organization** by a Dilaya admin (dilaya.eu > Organizations > « Recherche vectorielle »).

Two error codes gate everything, fail-closed:
- `VECTOR_TOOLS_NOT_ENABLED` — the org's flag is off. Ask the user to get it enabled at dilaya.eu. (A fresh enable works immediately — the connector re-checks on refusal.)
- `VECTOR_TOOLS_NOT_CONFIGURED` — the deployment has no platform key (infra-level; report it, don't retry).

## Model & invariant

Default model `text-embedding-3-small` (1536 dims; Matryoshka: any 1..1536 works, 512 ≈ ⅓ the storage for a small quality loss). `text-embedding-3-large` (≤3072) also available. **A query MUST be embedded with the SAME model + dimensions as the index it searches** — the per-app `_vec_indexes` registry row records them and the tools enforce it (`VEC_INDEX_MISMATCH` / `VEC_DIMENSION_MISMATCH`). Changing model = build a NEW index; never mix in one table.

## Tools

- `embed({ schema, texts[≤100], index?, model?, dimensions?, as? })` — texts → vectors. Give `index` to inherit an existing index's model/dims/prefixes (the safe way to embed a query); returns vectors + `usage.tokens` + `cost_estimate` (small ≈ 0.02$/M tokens).
- `vector-search({ schema, index, query_text?|query_vector?, k?, filter_sql?, return_columns? })` — pure KNN over one index. `query_text` is embedded internally with the index's own settings. `filter_sql` (aliases: `src` = companion row, `t` = source table) applies AFTER the KNN with automatic over-fetch; `return_columns` joins source-table columns. Returns `{ source_pk, text, distance, … }` — never raw vectors.
- `vector-index-list({ schema })` / `vector-index-drop({ schema, index, confirm: true })`.
- `embed-bulk-csv({ schema, csv_path, index, id_column, text_template, model?, dimensions?, mode? })` — index a whole CSV from org storage (`get-upload-url` first; header row required; caps 20 MB / 200k rows). Creates the index on first run. **Idempotent by hash**: a row re-embeds only when its rendered text changed. Works ~18 s per call and returns `status:"partial"` + counters until finished — **re-call with the same arguments to continue** (a cron/agent can loop until `"done"`). Resume is **constant-cost**: the job persists a cursor, so continuing a 100k-row file costs the same per call as starting it — no manual file splitting, whatever the size (within the caps). `mode:"replace"` wipes and rebuilds. Enrich `text_template` with synonyms/codes to catch abbreviations.
- `embed-status({ schema, job_id })` — one job's counters (status running|partial|done|failed, tokens, cost, last_error). Failures are LOUD: the job row records the error and an error notification lands in the app's inbox.

## Hybrid lexical + vector ranking — a CALLER-side recipe, not a primitive

FTS5 (exact codes, references, acronyms) and KNN (synonyms, reformulations) are two orthogonal engines; how to MIX them is an app-level ranking policy. The standard recipe is Reciprocal Rank Fusion (~10 lines, in your handler/agent):

1. `A = vector-search({ index, query_text, k: 50 })`
2. `B = query({ sql: "SELECT id FROM my_fts WHERE my_fts MATCH :q ORDER BY rank LIMIT 50" })`  (an FTS5 table YOUR app maintains)
3. RRF: `score(d) = Σ 1/(60 + rank_d)` over both lists → sort → top k.

Tip for catalogs: exact-reference match (FTS5) first, then vector re-ranking; enrich the indexed text with synonyms/codes to catch abbreviations.

## Storage notes

An index = a vec0 virtual table `<index>` + companion `<index>_src` (source_pk, text, hash) + one `_vec_indexes` row. 1536 dims ≈ 6 KB/vector. Raw vec0 SQL stays available for bring-your-own-embeddings (see the use-app topic §5).
