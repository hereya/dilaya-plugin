## 5. Login — private by default, public on request (per-app auth)

**A site is PRIVATE unless it is explicitly public.** The first `enable-frontend` (without `public: true`) provisions passwordless e-mail login (a per-app Cognito pool + Postmark sender, passkeys offered) and the PLATFORM closes the site: a visitor without a session is sent to `/auth/login` by the edge and refused by the origin — nothing to code, and it also covers static sections. The person who created the site is already allowed in; allow others with `add-user`. Decide from what the site IS, never by asking a technical question: a client space, a back-office, an intranet, a tool for a team → private (the default); a showcase site, a landing page, a public menu, documentation → `enable-frontend({ schema, public: true })`.

```
enable-frontend({ schema: "myapp" })                         # private: login provisioned, the platform closes the site
enable-frontend({ schema: "myapp", public: true })           # public: open to everyone, no login
add-user({ schema: "myapp", email: "marie@example.com" })    # allowlist who may sign in (the creator already is)
enable-auth({ schema: "myapp", public_paths: ["/menu", "/api/hooks"] })   # keep some paths open on a private site
```

`enable-auth` is the same provisioning on its own (to close a site that was created public, or to change its settings). Idempotent. On a first enable it closes the site (`enforce: true`); `public_paths` keeps prefixes open (`'/menu'` opens `/menu` and `/menu/…`; `'/'` the root page only; `/auth/*` and `/static/*` are always open) — **a public webhook or API endpoint on a private site must be declared there**, or the platform refuses it. `enforce: false` hands closing back to your handler (the older contract: check `req.auth.authenticated` and return a 302 to `/auth/login` yourself — apps that enabled login before this existed still work that way until re-run with `enforce: true`). Either way your handler reads the visitor:

```js
// on a platform-closed site every page request is authenticated already
const who = req.auth.email;              // the Cognito-verified user
if (!req.auth.authenticated) {           // only reachable on a public_paths route, or with enforce:false
  return { statusCode: 302, headers: { location: "/auth/login" }, body: "" };
}
```

**Verify before saying "it is protected":** `enable-frontend`/`enable-auth` answer `enforce: true` (and `access: "private"`); the site's URL, opened without a session, must redirect to `/auth/login`. If it serves a page, it is public.

The `/auth/login`, `/auth/verify` and `/auth/logout` routes are provided for you, served on the app's vanity/custom domain (the ONLY public serving surface — the first-party path URL is origin-locked for the site AND the auth pages, so always use the host-relative `/auth/...` form above, never a `/o/<org>/<app>/...` path). Manage the allowlist with `add-user` / `remove-user-access` / `list-users`; `migrate-auth({ schema, copy_users: true })` re-syncs the allowlist into Cognito after a pool re-provision; `disable-auth({ schema, confirm: true })` reverts the site to public (deletes the pool); `enable-frontend({ schema, public: true })` reopens it while keeping the login for later.

**Passkeys.** The login page also offers **"Sign in with a passkey"** (Touch ID, Face ID, Windows Hello, security keys) — on by default with `enable-auth`: after a first e-mail code, the user is invited to register a passkey on their device; next time, e-mail + passkey, no code. The passkey is bound to the app's **principal host** (its verified custom domain, else the vanity host — `enable-auth` returns it as `passkeys.rp_id`); the staging host and any other host keep the e-mail code, which always stays available as the fallback. `enable-auth({ schema, passkeys: false })` turns it off; re-run `enable-auth` after the app's host changes so the binding follows (passkeys registered on the previous host stop working — users simply register again after their next code). Nothing to do in your handler: a passkey sign-in sets the same session as an OTP.

**Branding the login pages.** The login / code / passkey pages share one neutral look. Set the app's LANGUAGE and ONE accent color via `enable-auth` (idempotent — re-run it any time just for branding): without `lang` the pages follow the visitor's browser, so a French title can sit over English strings; `accent_color` is applied by the platform to every page — buttons, the secondary "resend code" button, focus rings, links — which hand-written CSS routinely missed on the code page:

```
enable-auth({
  schema: "myapp",
  lang: "fr",                                         # the APP's language ('fr' | 'en'; 'auto' = the browser)
  accent_color: "#b4532a",                            # ONE brand color, every page, every button
  login_title: "Espace client Cariaco",              # replaces the "Connexion"/"Sign in" heading + tab title
  logo_url: "https://myapp--org.dilaya-apps.eu/logo.png",  # https image shown above the heading
})
```

`custom_css` is the escape hatch for what `accent_color` cannot do (fonts, the `.card`, `body` background): it is injected AFTER the shared stylesheet on ALL pages, so cover `button.secondary` (the code page's "resend") as well as `button`. Pass an empty string `""` to reset any param to the default. Changes are live within ~a minute (no redeploy). Subtitles, field labels and error messages stay on the built-in FR/EN i18n.

> **Key user records by `email`**, never `cognito_sub` — each app has its own Cognito pool, so `cognito_sub` is pool-local and will not survive a pool migration.
