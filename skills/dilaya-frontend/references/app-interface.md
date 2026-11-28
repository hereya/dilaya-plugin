## Internal interface (Dilaya console) — not a website

**What it is.** The screens the person's TEAM uses every day, inside the Dilaya console (dilaya.eu/console, the app's « Interface » tab, on a computer or a phone): a list with search, a form to add or edit, a dashboard of key figures. It is **not** a website: no address of its own, no visitor, no `enable-frontend`, no `deploy-backend`. Members sign in to the console; the interface only reads and writes the app's own data through operations you declare.

**When.** By default, whenever the person says « interface », « écran », « tableau de bord pour l'équipe », « dans la console », « interne », or wants their team to use the app without talking to an AI. A **website** (the frontend recipe: `enable-frontend`) is for « site », « page publique », « pour mes clients », « nom de domaine » — people outside the organization. When it is truly unclear, ask ONE question: « pour votre équipe, dans la console, ou un site public ? ». Every app should have an internal interface once its tables exist.

### 1. Read before you write

`read-app-interface({ schema })` returns the current files and ops (`which: "draft"` = the unpublished draft). **An interface exists → start from its files, change only what was asked; never redeploy one from scratch.** Read the tables first too (`get-skill` / `describe-schema`).

A big file comes back cut: ask for it alone with `path` (the whole file, text or base64). **With the sandbox tools** (an agent that has `sandbox_pull_interface`): always `sandbox_pull_interface` → edit in place (`sandbox_files` edit / insert / search / read from..to) → `sandbox_render_check` → `sandbox_deploy_interface`; files never pass through the conversation.

### Kit Dilaya — the base of every NEW interface

Start every new interface from this (with the sandbox tools, `sandbox_pull_interface` lays it). **Never write, edit or send `kit/`**: index.html links `kit/1.0.0/dilaya.css` + `.js` and `deploy-app-interface` adds them itself. Only the accent changes (`--dy-accent` in app.css).
```html
<!doctype html><html lang="fr"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<title>Stock</title><link rel="stylesheet" href="kit/1.0.0/dilaya.css"><link rel="stylesheet" href="app.css"><script type="module" src="app.js"></script></head>
<body><div class="dy-app"><header class="dy-top"><h1 data-dy-title>Stock</h1><div class="dy-top-actions" data-dy-actions></div></header>
<nav class="dy-tabs" aria-label="Navigation"><a href="#stock" data-dy-tab><span class="dy-ico" aria-hidden="true">📦</span><span>Stock</span></a></nav>
<main class="dy-main" data-dy-main></main></div></body></html>
```
- `import { router, query, call, h, stat, list, table, field, form, sheet, confirm, toast, pref } from "./kit/1.0.0/dilaya.js"`.
- `router({ stock: { title: "Stock", render: async (main, args) => node } })`: one entry per tab (`#stock/12` → `["12"]`); title, active tab, loading and errors handled.
- `await query("products", { q })` → rows as objects; `call("add_move", {…})` writes; errors carry `message`.
- Blocks: `stat`, `list(rows, r => ({ title, sub, end, onclick }))`, `table(cols, rows)` (cards on a phone), `field`/`form`, `sheet`, `confirm` (never `window.confirm`); classes `dy-card dy-grid dy-stack dy-btn`. Use `h()`, never `innerHTML`.
- On a kit page `sandbox_render_check` (`interface/index.html#<tab>`) also checks the tabs, the title and 44 px targets. An interface without the kit or on an older one (answer `kit_latest`) stays as it is unless asked.

### 2. Deploy

```
deploy-app-interface({ schema: "stock",
  files: [ { path: "index.html", content: "<!doctype html>…" },
           { path: "app.js", content: "…" }, { path: "app.css", content: "…" } ],
  manifest: { ops: {
    products:  { kind: "query", sql: "SELECT id, name, qty FROM products WHERE name LIKE :q ORDER BY name", params: ["q"] },
    add_move:  { kind: "write", sql: "INSERT INTO moves (product_id, qty, note) VALUES (:product_id, :qty, :note)", params: ["product_id", "qty", "note"] }
  } } })
```

- **Files**: `index.html` at the root, at most 200 files and 5 MB, relative paths (letters, digits, `. _ - /`), extensions html js mjs css svg png jpg jpeg gif webp ico woff woff2 json (a JSON file ≤ 64 KB: settings, never data). Scripts are FILES (`<script type="module" src="app.js">`) — **no inline script, no CDN, no fetch**: the frame has no network at all. Bundle every library and font you use.
- **Ops** (≤ 50, names `^[a-z][a-z0-9_-]{0,39}$`, not `me` / `openLink`): `query` = ONE read-only SELECT; `write` = ONE INSERT / UPDATE / DELETE starting with its verb, an UPDATE or DELETE needs a WHERE. Parameters are named (`:name`), each one declared in `params` and used; never a platform table (`_…`, `sqlite_…`). SQL ≤ 10 000 characters, ≤ 20 params. The SQL lives in the manifest, never in the browser.
- The answer gives `version`, `status` (`live` or `draft`), `ops`, `write_ops` and a `publish_hint`.
- Always give `summary`: ONE line (≤ 140) the history shows instead of a version code — « Ajout de l'écran Bons de commande ». A draft's answer carries `draft_link`: give it to the person as a link « Voir le brouillon » (it opens the Interface tab on the draft).
- **Nothing published is dropped by accident**: a bundle that would remove files or ops of the published version is refused (`UI_WOULD_REMOVE`, with the list). Start again from the published files; give `replace: true` only when the person asked to remove them, or you renamed those files. `which: "draft"` with no draft answers the PUBLISHED version with a `note`.

### 3. The bridge (the only way the page talks to Dilaya)

With the kit: `query` / `call` / `pref` above. Without it: the protocol is in `deploy-app-interface`'s description (`dilaya-ui-init` → `e.ports[0]` → `{id, op, params}`, with the answers).

- Built-ins: `me` → `{ email, role }`; `openLink` `{ url }` (https only, new tab); `setPref` `{ key, value }` / `getPref` `{ key }` to remember a choice (there is **no** localStorage or cookie); `download` `{ name, content, type? }` for a CSV / JSON / text file (not `<a download>`).
- An error is `{ code, message }`: show `message` to the person. A query returns at most 1 000 rows; 120 calls and 20 writes per minute; at most 10 calls in flight.
- **One page, never navigate**: switch views inside the page (show / hide, or `#hash`). Loading another page cuts the bridge.

### 4. Phone first — the interface must fit a 360 px screen

The kit does all of this. Without it: the viewport meta, `*{box-sizing:border-box}`, `html,body{max-width:100%;overflow-x:hidden}`, no fixed widths, `overflow-wrap:anywhere`, one column under 640 px, buttons ≥ 44 px, a wide table → cards, light and dark (`prefers-color-scheme`). **No sideways scrolling, ever.**

### 5. Publication — draft, preview, « Publier pour l'équipe »

- A version with **only queries** goes live at once (it can only read).
- A version with a **write op** is a **draft**: the current version stays live. An owner, an admin or a Développeur of the organization opens the app's « Interface » tab in the console, sees « Ce que cette version peut modifier » (each write op and its SQL), checks « Aperçu », then « Publier pour l'équipe », confirmed with their « clé d'accès » (passkey). Tell the person exactly that, in one sentence — never claim it is published.
- One draft at a time: a new deploy replaces it. The 20 newest versions are kept: « Versions » → « Revenir à cette version » rolls back (same passkey).
- Safety you can promise: the page can only run the declared operations; nothing else of the app is reachable, and every write is shown to the person who publishes.
