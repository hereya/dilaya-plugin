## How sources travel between sessions — zip or git (the app remembers)

When the frontend outgrows a hand-written `handler.js` + a few files (a real framework, npm dependencies, a build step), its SOURCES must survive from one session to the next. There are **two ways, and the app records which one it uses** — start every session with:

```
get-app-sources({ schema: "myapp" })   // says the mode AND hands back what that mode needs
```

Its `source_mode` is `"zip"` or `"git"`. **Never try git on a zip-mode app**, and never assume an app has no sources because it has no repo.

**Zip mode (the default).** The connector fetches the sources server-side and returns a **presigned download URL**; you `curl` + `unzip`, work, rebuild, and ship through the ordinary flow (`get-upload-url` → `deploy-backend`). No git and no GitHub credential ever crosses your network — so it works in sandboxes whose egress proxy replaces a git URL's credentials (a clone fails there, valid token or not). Between sessions, `get-app-sources` returns the **last deployed zip** (every `deploy-backend` archives one); before the first deploy, the starter archive the app was created from. `get-app-template-archive({ schema, template })` gets any template as a zip for an app that already exists.

**Git mode.** A private repo the platform manages, with a short-lived authenticated remote:

```
check-git-access({})                                  // 1. get the probe (a read-only remote + one command)
git ls-remote <that remote_url> HEAD                  // 2. run it HERE
check-git-access({ schema: "myapp", result: "ok" })   // 3. record the verdict ('failed' → stay in zip mode)
create-app-repo({ schema: "myapp" })                  // 4. only if the probe passed
get-app-repo({ schema: "myapp" })                     // any later session — fresh URL for the same repo
```

**Run the probe BEFORE creating a repo.** One command, nothing created; skipping it leaves an app with a repo its own agent cannot clone. The `remote_url` embeds a token scoped to THIS app's repo only, valid ~1 hour: `git clone`, work, commit, push — then **build locally and deploy the OUTPUT through the normal flow**, the same as zip mode. The repo stores sources; it is never a serving path. Treat the whole URL as a secret (never commit it); when it expires, call `get-app-repo` again. Dropping the app archives the repo (read-only) rather than deleting it. Unavailable deployments return `GITHUB_REPOS_NOT_CONFIGURED`.

**Switching.** `set-app-source-mode({ schema, mode })` records the choice at any time. It never deletes anything (an app moved to zip keeps its repo).

**Apps delivered from Git** (`deploy-backend` answers `GIT_MANAGED_APP`): no zip, see the `dilaya-update-app` skill.

> **Templates.** The **App templates** section is the starting point when a frontend needs a real framework/build — prefer `create-app-from-template` over wiring a stack by hand.
