## Dynamic pages, static sections — mix freely

Without `static_prefixes` every page renders **dynamically**: the per-app Lambda runs your `handler.js` on EVERY request (SSR, auth, personalization) — the flow below. Additionally, an app can declare **static sections**: URI prefixes whose pages you pre-build and ship in the zip's `site/` folder, served STRAIGHT from the edge (CDN + S3) — no Lambda per view: faster, cacheable, ideal for public content/SEO. See §6.

- `static_prefixes: ["/"]` → the whole site is static (pure landing/docs; `handler.js` optional, serves only `/api/*`).
- `static_prefixes: ["/landing","/docs"]` → **hybrid**: those sections static, everything else dynamic — including pages behind `enable-auth`.
- Static bytes are identical for every visitor (no per-request rendering) — on a PRIVATE site the platform still closes them at the edge (a visitor without a session is sent to login before any file is served); anything personalized stays dynamic.
