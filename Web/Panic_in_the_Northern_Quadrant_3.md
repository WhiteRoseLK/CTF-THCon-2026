# Panic in the Northern Quadrant — 3

- **Category:** Crypto / Hash cracking
- **Challenge ID:** 50
- **Points:** 448 (18 solves)

```
THC{Dynamics314!}
```

## TL;DR

- The `credentials` table from part 2's SQLite dump has the admin row hashed with **unsalted SHA-256**.
- A separate page (`register.php`) leaks the **password policy**: upper + lower + ≥3 digits + ≥1 special. That rules out plain rockyou hits and explains why a vanilla dictionary attack misses.
- `hashcat -m 1400 admin.hash rockyou.txt -r dive.rule` cracks it in minutes: `Dynamics314!` — the company name + `314!` matches the policy.
- The flag is the cracked plaintext wrapped: `THC{Dynamics314!}`.

## 1. The hash

```sql
$ sqlite3 db.bak "SELECT * FROM credentials;"
1|admin    |6e97320f1cd2e9d07347aa5c985fa4353d9aeab4530500831a9b3a8975b7768e|superadmin
2|operator |81cb3a0b84f737444e02e69bf6ac8e1a85f46507412b796defa25dbfb312d06a|user
```

64 hex characters, no salt prefix, no PHC string. SHA-256 unsalted (hashcat mode `1400`).

```bash
$ echo 6e97320f1cd2e9d07347aa5c985fa4353d9aeab4530500831a9b3a8975b7768e > admin.hash
```

## 2. The password policy hint

`register.php` is a wired-up form with a hidden help div for "bad password" errors:

```html
<!-- TODO display on bad password error -->
<div hidden>
  You input a password that does not follow the password policy
  (upper and lower case, at least 3 numbers and one special character)
</div>
```

So the password matches roughly:

```
^(?=.*[a-z])(?=.*[A-Z])(?=(?:.*\d){3,})(?=.*[^A-Za-z0-9]).+$
```

Plain rockyou candidates almost never satisfy this. Need rule-based mutation.

## 3. Cracking

`dive.rule` is the heavy ruleset shipped with hashcat — ~100k mutations covering capitalization tweaks, digit suffixes, special-character append/prepend. Tailored exactly for "company name plus suffix" passwords against a strict policy.

```bash
$ RULES=$(find /opt/homebrew /usr/share -name 'dive.rule' | head -1)
$ hashcat -m 1400 admin.hash rockyou.txt -r "$RULES" --quiet
6e97320f1cd2e9d07347aa5c985fa4353d9aeab4530500831a9b3a8975b7768e:Dynamics314!
```

Verification:

```bash
$ python3 -c "import hashlib; print(hashlib.sha256(b'Dynamics314!').hexdigest())"
6e97320f1cd2e9d07347aa5c985fa4353d9aeab4530500831a9b3a8975b7768e
```

`Dynamics` (the company name in the page footer: "SST Dynamics") + `314!` — capital, lowercase, three digits, one special. The mutation rule that produced it is roughly *append "314!" + capitalize first*.

## 4. Flag wrapping

> The flag is `THC{Dynamics314!}`. Submitting the bare `Dynamics314!` is rejected. Always wrap.

## 5. Methodology

- **Identify hash type from layout, not from the column name.** A 64-hex string is SHA-256/Keccak-256/Blake-2s; `hashcat --identify` confirms in seconds. Don't waste a run on `-m 100` (SHA-1) — the length doesn't fit.
- **Recon the password policy *before* cracking.** Dictionary attacks work in proportion to how well candidate distribution matches policy distribution. A leaked policy turns "guess and pray" into "rule-mutate to fit" — a 100× speedup.
- **`dive.rule` is the universal "I don't know what mutation is needed" hashcat rule.** When you don't have a per-target ruleset, dive is the right default.
- **Company / project names + numeric suffixes dominate the policy-compliant password space.** The "SST Dynamics" branding visible on the homepage *is* the wordlist.
- **Unsalted SHA-256 is fail-deadly.** A salted bcrypt/argon2 with the same plaintext would have made this challenge several orders of magnitude harder. The author chose unsalted on purpose.
