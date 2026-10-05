## Actions — freeze an exact treatment in the backend, and CALL it

A calculation the user relies on (a shopping list for N guests, an invoice total, a stock count)
must give the SAME answer every time it is asked — in two conversations a week apart, and on the
screen when the app has one. Recomputing it in SQL each time drifts: a rounding, a forgotten
filter, a unit. So once a treatment matters, it lives ONCE, as code in the app's handler, and
Claude (and the pages, if any) call that code. « L'IA fabrique, le logiciel exécute. »

**An app with NO web page needs this just as much** — there the conversation is the only way in,
so an unpinned calculation drifts with nothing on screen to catch it. See « No screen? » below.

### The rule

- When `get-skill` shows `backend_actions` (or `list-app-actions` lists one) that covers the
  request, **call it**: `run-app-action` for a read, `run-app-action-write` when `write` is true.
  Do not recompute its answer in SQL, and do not "check" it with a query of your own.
- SQL (`query`) stays the tool for EXPLORING the data — questions no action answers.
- **Before a write action, or one that reaches people** (an email, a message, a booking), say in
  one sentence what it will do and wait for the user's yes. Pass an `idempotencyKey` so a retry
  after a timeout is recognised instead of run twice.

### 1. Write the treatment as ONE function the route and the action share

```javascript
const { parseRequest, query } = require("hereya");

// The treatment — pure logic over the data, no HTTP.
async function listeCourses({ invites }) {
  const { rows } = await query("SELECT ingredient, qte_par_invite, unite FROM recette");
  return rows.map((r) => ({ ...r, total: Math.ceil(r.qte_par_invite * invites) }));
}

exports.handler = async (event) => {
  const req = parseRequest(event);
  if (req.action) {
    if (req.action.probe) return { statusCode: 204 };            // deploy check: routed, do nothing
    if (req.action.name === "liste-courses") {
      const invites = Number(req.action.input.invites);
      if (!Number.isInteger(invites) || invites < 1) return json(400, { error: "invites: entier ≥ 1" });
      return json(200, await listeCourses({ invites }));
    }
    return json(404, { error: "action inconnue" });
  }
  if (req.path === "/api/liste-courses") {                         // the page calls the SAME function
    return json(200, await listeCourses({ invites: Number(req.query.invites) }));
  }
  // ... other routes ...
};
const json = (statusCode, body) => ({ statusCode, headers: { "content-type": "application/json" }, body: JSON.stringify(body) });
```

- `req.action` = `{ name, input, write, idempotencyKey?, probe? }`; `req.auth` = the org member
  calling (`email`, `role`, `via: "claude"`). A write action refuses `write: false` (403), and may
  refuse a role (`req.auth.role` not owner/admin → 403).
- An unknown name answers **404**, a probe answers without doing anything.

### No screen? A handler that serves only actions

Same handler, without the page routes: everything that is not `req.action` answers 404.

```javascript
exports.handler = async (event) => {
  const req = parseRequest(event);
  if (!req.action) return json(404, { error: "pas de page" });   // no web page on purpose
  if (req.action.probe) return { statusCode: 204 };
  if (req.action.name === "liste-courses") return json(200, await listeCourses(req.action.input));
  return json(404, { error: "action inconnue" });
};
```

Call `enable-frontend({ schema })` ONCE before the first `deploy-backend`: a deploy alone turns
the site on OPEN, while `enable-frontend` closes it by default. No page is served either way; this
only keeps the app's address shut. Then `deploy-backend` as usual (no `site/` folder, no
`static_prefixes`). The proof in step 3 compares the action with today's SQL answer, and the
same question asked in two conversations must give the same number.

### 2. Declare it — `actions.json` at the zip root, next to handler.js

```json
[{ "name": "liste-courses", "description": "La liste de courses pour N invités (quantités arrondies au-dessus).",
   "write": false,
   "input": { "type": "object", "properties": { "invites": { "type": "integer" } }, "required": ["invites"] } }]
```

`deploy-backend` reads it, stores it, and probes each READ action (`actions.warnings` in its answer
names one the handler does not route). The description is what tells Claude WHEN to call it.

### 3. Prove it against the old answer BEFORE switching

1. Pick 3 to 5 real cases (e.g. 10, 30, 80 guests).
2. For each, run the current SQL recipe AND the action (`env: "staging"` first, after a
   `deploy-backend` with `env: "staging"`). Show the user the two answers side by side.
3. Any difference is a decision for the user — which one is right — never a silent pick. Fix the
   function until they agree (or until the user confirms the new one is the right one).
4. `promote-deployment`, then run one case on production.

### 4. Switch the skill

Replace the SQL recipe in the app's skill with « pour X, appelle l'action `liste-courses` » (name,
inputs, what it returns). Keep SQL in the skill only for exploration.
