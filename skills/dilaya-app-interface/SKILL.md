---
name: dilaya-app-interface
description: "Use when the team needs screens inside the Dilaya console for an app (an internal interface: list, search, forms, dashboard) — whenever the person says interface, écran, tableau de bord, interne or dans la console. Not a public website."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# Internal interface (Dilaya console) — not a website

**What it is.** The screens the person's TEAM uses every day, inside the Dilaya console (dilaya.eu/console, the app's « Interface » tab, on a computer or a phone): a list with search, a form to add or edit, a dashboard of key figures. It is **not** a website: no address of its own, no visitor, no `enable-frontend`, no `deploy-backend`. Members sign in to the console; the interface only reads and writes the app's own data through operations you declare.

**When.** By default, whenever the person says « interface », « écran », « tableau de bord pour l'équipe », « dans la console », « interne », or wants their team to use the app without talking to an AI. A **website** (the frontend recipe: `enable-frontend`) is for « site », « page publique », « pour mes clients », « nom de domaine » — people outside the organization. When it is truly unclear, ask ONE question: « pour votre équipe, dans la console, ou un site public ? ». Every app should have an internal interface once its tables exist.

## 1. Read before you write

`read-app-interface({ schema })` returns the current files and ops (`which: "draft"` = the unpublished draft). **An interface exists → start from its files, change only what was asked; never redeploy one from scratch.** Read the tables first too (`get-skill` / `describe-schema`).

A big file comes back cut: ask for it alone with `path` (the whole file, text or base64). **With the sandbox tools** (an agent that has `sandbox_pull_interface`): always `sandbox_pull_interface` → edit in place (`sandbox_files` edit / insert / search / read from..to) → `sandbox_render_check` → `sandbox_deploy_interface`; files never pass through the conversation.

## 2. Deploy

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

## 3. The bridge (the only way the page talks to Dilaya)

```js
let port, next = 0; const waiting = new Map();
window.addEventListener("message", (e) => {
  if (e.data?.type !== "dilaya-ui-init" || port) return;
  port = e.ports[0];
  port.onmessage = ({ data }) => { const p = waiting.get(data.id); waiting.delete(data.id);
    data.ok ? p.resolve(data.result) : p.reject(data.error); };
  start();
});
const call = (op, params) => new Promise((resolve, reject) => {
  const id = ++next; waiting.set(id, { resolve, reject }); port.postMessage({ id, op, params }); });
// await call("products", { q: "%vis%" }) → { columns, rows, truncated }
// await call("add_move", { product_id: 3, qty: -2, note: "" }) → { changes, last_insert_rowid }
```

- Built-ins: `me` → `{ email, role }`; `openLink` `{ url }` (https only, new tab); `setPref` `{ key, value }` / `getPref` `{ key }` to remember a choice (there is **no** localStorage or cookie); `download` `{ name, content, type? }` for a CSV / JSON / text file (not `<a download>`).
- An error is `{ code, message }`: show `message` to the person. A query returns at most 1 000 rows; 120 calls and 20 writes per minute; at most 10 calls in flight.
- **One page, never navigate**: switch views inside the page (show / hide, or `#hash`). Loading another page cuts the bridge.

## 4. Phone first — the interface must fit a 360 px screen

`<meta name="viewport" content="width=device-width,initial-scale=1">`, `*{box-sizing:border-box}`, `html,body{max-width:100%;overflow-x:hidden}`, no fixed widths (`max-width` and %), long words wrapped (`overflow-wrap:anywhere`), grids that fall to one column under 640 px, buttons ≥ 44 px high. A wide table goes inside `<div style="overflow-x:auto">` — or, better on a phone, becomes a list of cards. Light and dark themes (`prefers-color-scheme`). **No sideways scrolling, ever.**

## 5. Publication — draft, preview, « Publier pour l'équipe »

- A version with **only queries** goes live at once (it can only read).
- A version with a **write op** is a **draft**: the current version stays live. An owner, an admin or a Développeur of the organization opens the app's « Interface » tab in the console, sees « Ce que cette version peut modifier » (each write op and its SQL), checks « Aperçu », then « Publier pour l'équipe », confirmed with their « clé d'accès » (passkey). Tell the person exactly that, in one sentence — never claim it is published.
- One draft at a time: a new deploy replaces it. The 20 newest versions are kept: « Versions » → « Revenir à cette version » rolls back (same passkey).
- Safety you can promise: the page can only run the declared operations; nothing else of the app is reachable, and every write is shown to the person who publishes.
