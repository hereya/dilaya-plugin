## 3. Package + upload

Zip `handler.js` at the **root** of the zip (not nested in a folder) and upload it to `<app>/backend/deployment.zip`:

```
get-upload-url({ path: "myapp/backend/deployment.zip", content_type: "application/zip" })
# then PUT the zip bytes to the returned URL
```

For a multi-file handler, bundle it down to a single `handler.js` first (e.g. esbuild with `hereya` and `@aws-sdk/*` marked external) and zip the bundle's contents.

### Edge-served static assets (`assets/` in the zip) + cache opt-in

Put static files (CSS, JS, images, fonts) in an `assets/` folder inside the SAME zip: deploy-backend extracts them and they are served at `/static/<path>` on the app's vanity/custom-domain host STRAIGHT from the edge (CDN + S3, no Lambda — fast, great for SEO). Reference them from your HTML as `/static/app.css`. Rules:
- **Content-hash the filenames** (`app.3f9a1c.css`): assets are cached ~1 year immutable at the edge — a same-name overwrite can serve stale. New content → new filename.
- The HTML page itself still comes from your handler. To ALSO cache a PUBLIC page at the edge, add `...cacheHeaders(seconds)` (from `hereya`) to its response headers — e.g. `headers: { "content-type": "text/html", ...cacheHeaders(300) }`. Normally only for pages identical for every visitor; personalized responses stay uncached by default (no header, no cache).
- **A POLLED endpoint is the exception, and it is the cheapest protection you have.** If your page calls something on a timer (a session check, a notifications count, a live counter), put a SHORT `cacheHeaders(5)` on it even though the response is per-user. The edge cache key includes the viewer's session cookie, so one visitor's response can never reach another — and 5 seconds of caching turns a page polling 10 times a second into one origin call every 5 seconds. This is what absorbed a real incident: a tenant page looping on its own `/api/auth/me` produced 17 386 requests in under two hours, ~70 % of the platform's traffic that day, every one a healthy 200. It also protects you when a page is simply POPULAR, which a rate limit answers by cutting visitors off.
- **Two rules if you cache a per-user response.** (1) Keep the TTL to a few seconds: the header says `public`, which is safe here because the cookie is in the cache key, but it is the wrong word for personal content and a short TTL means a mistake expires by itself. (2) **Request headers are NOT part of the cache key** — only the path, the query string and the session cookie are. A response that varies on a custom header or an `Authorization` header will be served the wrong cached entry, silently. If your endpoint varies on anything else, do not cache it.
- deploy-backend reports `static_assets: { uploaded, deleted }` when the feature is active; assets sync on every deploy (removed files are deleted).
