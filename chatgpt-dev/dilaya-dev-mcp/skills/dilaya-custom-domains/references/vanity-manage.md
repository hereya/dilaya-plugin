## Manage

```
list-app-hosts({ schema? })                    # your org's vanity hosts (host + url + app)
disable-app-host({ schema, confirm: true })    # remove an app's vanity host(s); the path URL stays live
check-app-hosts()                              # repair: rebuild the edge routing from the registry, report your hosts
```

`check-app-hosts` is the self-heal: it rebuilds the edge host map from the registry (the source of truth) and re-publishes it — run it if a host ever stops resolving.
