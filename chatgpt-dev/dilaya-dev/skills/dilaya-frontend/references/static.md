## 6. Static sections — pre-built pages served from the edge

For pages that are the same for every visitor (a landing, docs, a public SPA), skip per-request rendering on those paths:

```
enable-frontend({ schema: "myapp", static_prefixes: ["/"] })          # whole site static (pure landing/docs)
enable-frontend({ schema: "myapp", static_prefixes: ["/landing"] })   # hybrid: /landing static, the rest dynamic (auth OK)
set-app-host({ schema: "myapp" })     # REQUIRED — static sections serve ONLY on the vanity/custom-domain host
```

Build the zip with the static pages under `site/`, **mirroring the URL paths**, alongside the usual `handler.js` for the dynamic pages:

```
site/index.html               # entry point of the "/" section (whole-static case)
site/landing/index.html       # entry point of a "/landing" section (hybrid case)
site/landing/app.3f9a1c.js    # content-hash every non-HTML filename (cached ~1 year immutable)
handler.js                    # dynamic pages + /api/* — OPTIONAL only when static_prefixes is ["/"]
assets/…                      # still works too (served at /static/*)
```

Upload to `<app>/backend/deployment.zip` and run `deploy-backend` as usual — it extracts `site/` to the edge bucket (reports `site_files`) and deploys the Lambda whenever `handler.js` is present. Serving rules on the host:

- A path under a static prefix: WITH a file extension (`/landing/app.3f9a1c.js`) → that file from S3; WITHOUT → **file-first resolution**: `site/<path>/index.html` when the deployed bundle has it (`/a-propos` → `site/a-propos/index.html`, trailing slash normalized — multi-page bundles like Astro/Hugo/Next export serve every page under ONE `/` section), else the SECTION's `index.html` (SPA fallback per section, longest prefix wins — existing SPAs behave exactly as before). deploy-backend records the bundle's page routes (`page_routes` in its response, capped at 500 with a warning).
- Every other path — always including `/api/*` and `/auth/*` — → your `handler.js` Lambda (same runtime layer; auth flows untouched). A landing form can `POST` to `/api/subscribe` and the handler writes to SQLite / sends mail.
- HTML files are cached ~60 s at the edge (a redeploy shows within a minute); every other site file ~1 year immutable — **content-hash their filenames** (new content → new filename).

Auth: on a PRIVATE site (the default) the platform closes static sections too — a visitor without a session is sent to login at the edge before any file is served — so `static_prefixes: ["/"]` works with login, backend or not. Only the older handler-guarded contract (`enable-auth({ enforce: false })`) leaves static sections public and is refused for `["/"]` without a backend Lambda (`STATIC_MODE_AUTH_CONFLICT` — nothing would be auth-checked). Other notes: the path URL (`…/site/`) does not serve the static bundle, only the host does; `test-backend` invokes the Lambda only. Go back fully dynamic anytime with `enable-frontend({ schema, static_prefixes: [] })`.
