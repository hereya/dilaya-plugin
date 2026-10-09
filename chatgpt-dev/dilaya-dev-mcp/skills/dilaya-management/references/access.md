## Who in the organization can use an app

Every app carries a `visibility`, and the two values name the AUDIENCE rather than saying "public" or "private" — because "public" never says public TO WHOM:

- **`organisation`** (the default at creation) — every member of the organization may use it.
- **`personal`** — the member who created it, alone.

- `set-app-visibility({ schema, visibility: "organisation" | "personal" })` — the app's **owner** only.
- `reassign-app({ schema, to_user_id? })` — the **organization's owner** only; `to_user_id` omitted takes it for themselves. This is how an app is recovered when the member who created it leaves, instead of being stranded. Visibility is left untouched.

What `personal` does, so you can explain it accurately:

- Another member cannot open the app — not its data, files, skills, views, dashboards, crons or secrets — and does not even see it exists: naming it answers exactly like a name that was never created. Do not present this as a listing filter; it is refused at the door.
- The **organization's owner** is told it exists (it appears in `list-applications` with `visibility: "personal"`) but cannot open it. Taking it over with `reassign-app` is the deliberate, logged way in.
- Org-wide totals stay honest: `get-usage-report` still counts every app in its totals (they explain a quota refusal) and simply does not itemize the ones you may not see (`apps.notItemized`).

⚠️ **This setting decides nothing about the web.** An app set to `personal` whose frontend is published stays reachable by anyone on the internet — that is what publishing a site is. The two are unrelated, and the value names are chosen so the confusion cannot start: say « visible par les membres de votre organisation » or « visible par vous seul », never « publique ».
