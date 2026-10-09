---
name: dilaya-catalog-install
description: "Use when installing an app from the Dilaya app catalog (the user names a catalog app, or pastes an « Installe l'app … du catalogue Dilaya » prompt)."
---

# Install an app from the Dilaya catalog

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

1. Read the listing: `catalog-get({ name, version? })` (`version` defaults to the latest). It returns `title`, `summary`, `author_org`, `version`, `requires` (ready-to-read lines), `dilaya_md` (the install document) and `files`.
2. Follow [references/install-prompt.md](references/install-prompt.md) — its rules are NOT negotiable (pre-flight and explicit consent first, strict scope: only the newly created app). Replace every `<…>` placeholder with the value named inside it, and keep the rest verbatim. If `files` is empty, the package only contains the install document.
