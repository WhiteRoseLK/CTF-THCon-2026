# THCity: Authentication Collapse — 2

- **Category:** Web (Insane)
- **Points:** 442 (19 solves)
- **Target:** `http://web-thcity-authentication-collapse.ctf.thcon.party:8888`
- **Status:** **UNSOLVED** — investigation log

```
(flag not extracted)
```

## TL;DR

- The challenge ships its full stack: Apache + custom `mod_auth_thcity.so` (Basic-auth provider) → HTTP `GET` to an Express SSO at `express_sso:3000/sso/text?username=…&password=…`. The Express side strict-equals against `crypto.randomBytes(32).toString("hex")`, so brute force is out.
- Apache parses the SSO response as text, line by line: `status ≤ 201`, then a line containing `AuthOK` *before* any line with `AuthNOK`, then a line with `"username"`. If all three pass, auth is granted.
- Multiple bypass classes were tried (CRLF/HTTP-smuggling, Apache traversal CVEs, exotic verbs, headers, PHP LFI → RCE, K8s SA reach, timing). All failed against Express 4.21.2 + Node 20 + Envoy in front.
- The intended bypass almost certainly involves the line-scan response parser — getting an Express response body to contain `AuthOK` before `AuthNOK`, and a `"username"` line, despite Express's hard-coded JSON. The exact crafting was not found in time.

## 1. Stack handed to us

`attachment.zip` (carried over from part 1) is the entire deployment:

```
docker/
├── flag_server/
│   ├── thcon.conf                 ← Alias /secret + AuthBasicProvider thcity-sso
│   ├── image.php                  ← LFI primitive from part 1
│   ├── secret/index.php           ← reads flag from Redis on success
│   └── mod_auth_thcity.c          ← custom Apache module
└── sso_server/
    ├── index.js                   ← 50-LoC Express SSO
    └── package.json               ← express ^4.21.2
```

```
Client ──► Envoy (istio) ──► Apache + mod_auth_thcity ──► Express SSO
              :8888              /secret/                 :3000/sso/text
                                       │
                                       └──► (PHP secret/index.php) ──► Redis
```

## 2. The two relevant code blocks

### Apache module (the parser)

```c
snprintf(request, BUFFER_SIZE,
    "GET %s?username=%s&password=%s HTTP/1.1\r\n"
    "Host: %s\r\nUser-Agent: mod_auth_thcity/1.0\r\n"
    "Connection: close\r\n\r\n",
    SSO_PATH, username, password, SSO_HOST);
// send to express_sso:3000, recv ≤ 2048 B

if (get_http_status_code(response) > 201) return AUTH_USER_NOT_FOUND;
char *body = strstr(response, "\r\n\r\n");
// scan body line by line (ap_getword \n):
//   if line contains "AuthNOK" -> fail
//   if line contains "AuthOK"  -> ok flag = true
// then scan again to find "username" and extract its value
```

Three observations:

1. **`username` and `password` from Basic auth are pasted unencoded into the GET line.** That's CRLF-injectable into the upstream socket — the smuggling vector.
2. **Status acceptance is `≤ 201`.** A single 100-Continue line at the top of the buffer satisfies this even if the real response is a 403 later.
3. **The line scan stops at the first `AuthNOK` it sees.** Whichever string appears first in the response body decides auth.

### Express SSO (the oracle)

```js
const PASSWORD = crypto.randomBytes(32).toString("hex");   // 256 bits, 64 hex chars

app.get("/sso/text", (req, res) => {
    const { username, password } = req.query;
    if (username === "admin" && password === PASSWORD)
        return res.status(200).json({status: "AuthOK", username});
    return res.status(403).json({status: "AuthNOK",
        error: `Invalid credentials for user: ${username || "unknown"}`});
});
```

Failure JSON includes the **un-encoded `username`** in the `error` field — a reflection oracle for whatever we put in the username.

## 3. Vectors tried (and why they failed)

### 3.1 Brute force / dictionary
`admin/admin`, `root`, `sst`, `P4t4t0rz`, `Dynamics314!`… none match a 256-bit secret. Dead.

### 3.2 CRLF smuggling
Putting `\r\n` into the username injects a second pipelined HTTP request to Express:

```
username = "admin HTTP/1.1\r\nHost: x\r\nExpect: 100-continue\r\nFoo:"
```

Express then emits `100 Continue` followed by `403 ... AuthNOK ...`. The status check is satisfied (`100 ≤ 201`), but the line scan still hits `AuthNOK` from the 403 body before any synthetic `AuthOK`. The reflection through the `error` field of the *first* response also includes `AuthNOK` literal in the JSON, so injecting `AuthOK` into the username doesn't beat it.

### 3.3 Parameter pollution via `#`
`username=admin&password=KNOWN#` cuts off the rest of the upstream URL at the fragment, making `password === "KNOWN"` server-side. Useless — we don't know `KNOWN`.

### 3.4 Forced JSON SyntaxError reflecting `AuthOK`
Idea: smuggle a second request whose body parsing fails in a way that echoes `{"username":"AuthOK"}` in the SyntaxError message (matching both `strstr`s). Blocked because the *first* response (the legitimate GET) is always present in the recv buffer first and contains `AuthNOK`.

### 3.5 Apache traversal CVEs (CVE-2021-41773 / 42013)
`/secret/.%2e/etc/passwd` and variants — `404`/`400` on this Apache 2.4.57. Patched.

### 3.6 Exotic verbs / HTTP versions
OPTIONS, HEAD, PROPFIND, MERGE, PATCH, TRACE, DEBUG — all `401` or `405`. HTTP/1.0 returns `426 Upgrade Required`; HTTP/2 stays `401`.

### 3.7 Bypass headers
`X-Original-URL`, `X-Rewrite-URL`, `X-Real-IP`, `X-Envoy-Original-Path`, doubled `Authorization` — none move the auth boundary.

### 3.8 LFI → RCE
`image.php` uses `readfile()`, not `include()` — no PHP wrappers/pearcmd. `register_argc_argv = Off`. Logs go to stdout (Docker). No path to RCE this way.

### 3.9 Timing attack on `password === PASSWORD`
Strict equality on 64-char strings; network noise ~50–100 ms per request, no exploitable timing signal across 10× runs.

### 3.10 Kubernetes SA token
`/var/run/secrets/.../token` is readable via the LFI, but `image.php`'s `readfile()` is local-only — no SSRF primitive to reach `kubernetes.default.svc`. Stuck behind the read-only file primitive.

## 4. Likely intended path (best guess)

The two parser quirks point at the bypass:

- **Status check is `> 201` for failure.** Anything from 100 to 201 passes.
- **Body line scan accepts the first `AuthOK` if no earlier `AuthNOK` is seen.**

So the goal is to make the recv buffer look like:

```
HTTP/1.1 100 Continue\r\n\r\n        ← passes status check
HTTP/1.1 200 OK\r\n
...
\r\n
{"status":"AuthOK","username":"admin"}\r\n
```

before any `AuthNOK` JSON appears. CRLF in username only got us as far as adding *another* request after the legitimate one — never *before*. Possibilities not fully exhausted:

1. **h2c / HTTP/2 smuggling between Envoy and Apache.** Envoy istio is famous for desync gadgets; would need to test admin endpoints on `:15000`/`:15090`.
2. **`express.static` collision on `/sso/text`.** If a deployed `public/sso/text` file exists (not present in the repo), it would be served by static middleware before the route handler.
3. **`qs` query parser quirk** in Express 4.21.2 — array notation, prototype pollution, or duplicate-key handling that makes `username === "admin"` while `password` evaluates to a non-string equal to `PASSWORD`. Unlikely with `===`, but worth a methodical fuzz.
4. **Sidecar Envoy admin access.** Some istio deployments leak the admin port; if reachable, dynamic config tweaks become possible.

## 5. Methodology — what was learned

- **Read the parser, not the server.** When Apache parses an upstream HTTP response by hand, the bypass is in *its* string-scan rules, not in Express's logic. Map the scan order before crafting the payload.
- **Status ranges are oracles.** A `≤ 201` check accepts `100 Continue`, which is also one of the few responses that lets you smuggle a follow-up cleanly.
- **Reflection of unencoded user input is a string-injection vector even into JSON.** The Express error message reflects `username` raw — the issue is just that `AuthNOK` always appears *before* it.
- **Document the dead ends.** Listing them publicly saves the next person a day. Even an unsolved writeup is useful if the trail of failed attempts is precise.
