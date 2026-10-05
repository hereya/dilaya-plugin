## 1. Turn the frontend on

```
enable-frontend({ schema: "myapp" })
# → { url: "https://myapp-<orgslug>.<contentDomain>/", frontend_enabled: true }  (the link that serves — the /o/{orgId}/{app}/site/ path is origin-locked and answers 403)
```

Flips the flag and returns the public URL. The site does **not** serve yet — it starts serving once a handler is deployed.
