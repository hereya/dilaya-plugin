## 4. Deploy + test

```
deploy-backend({ schema: "myapp" })
# first call creates the per-app Lambda + public routes and turns the site on;
# later calls just push new code.
test-backend({ schema: "myapp", path: "/", method: "GET" })
```

`test-backend({ schema, path?, method?, query? })` invokes the Lambda with a synthetic READ request (GET/HEAD/OPTIONS) so you can verify it before sharing the URL; `test-backend-write({ schema, path?, method?, body?, query? })` does the same for POST/PUT/PATCH/DELETE. Roll everything back with `disable-frontend({ schema, confirm: true })` (tears down the Lambda + routes).
