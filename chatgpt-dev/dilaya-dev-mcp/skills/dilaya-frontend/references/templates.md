## App templates — check these FIRST (before writing any `handler.js` or HTML)

**Before writing the first line of a frontend — any `handler.js` or HTML page, and before you settle on an architecture below — call `list-app-templates`.** A template is a curated **starter repo** with the framework, styling, build step and Dilaya packaging already wired (the first one, `template-astro-static`, is Astro 5 static + Tailwind v4 + Lit islands, optional `handler.js` on `/api/*`). Starting from one instead of scaffolding by hand routinely saves **hours** of work and avoids reinventing a static-site generator. If a template covers the need, use `create-app-from-template`; write a frontend from scratch ONLY when `list-app-templates` shows none that fits.

```
list-app-templates()                                          // the available templates (+ what each provides)
create-app-from-template({ schema: "myapp", template: "template-astro-static" })
```

`create-app-from-template` creates the app (if it doesn't exist), **delivers the template's sources**, enables the frontend with the template's static sections, provisions the vanity host, and returns `next_steps`. The connector does NOT build — follow `next_steps`: get the sources, run the manifest's build/package commands (e.g. `npm install` then `npm run package` → `deployment.zip`), upload to `<app>/backend/deployment.zip` and `deploy-backend`. Errors: `TEMPLATE_NOT_FOUND`, `REPO_EXISTS` (git mode, app already has a repo), plus the create-app-repo set. (No template fits, or the frontend is small enough to hand-write? Read on — the architecture guidance and, for a hand-managed framework build, the "How sources travel" section below still apply.)

**It defaults to `mode: "zip"`** — the sources come as a presigned download, no git anywhere. Give `mode: "git"` for a private source repo instead, but only after `check-git-access` proved git works from your environment (next section). Either way the mode is recorded on the app, so the next session knows how to work without guessing.
