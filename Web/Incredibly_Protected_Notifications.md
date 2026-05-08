# Incredibly Protected Notifications

- **Category:** Web (easy)
- **Points:** 280 (dynamic, started at 500)
- **Author:** Adrien Barbanson (dridri) — Lyra Network
- **Target:** `http://incredibly-protected-notifications.ctf.thcon.party:8080`
- **Status:** **UNSOLVED** — investigation log

## TL;DR

- The merchant accepts checkout form data, signs a JWT, and redirects the user to a PSP. The PSP later "fires" an **IPN** (Instant Payment Notification) callback to an internal-only `processpayment.php?command=update`.
- The **`address`** form field is concatenated into the IPN URL without escaping, allowing parameter pollution. The flag is read by `processpayment.php?command=read`, which only succeeds in-cluster — i.e. classic SSRF via the PSP-fired IPN.
- The blocker found during the contest: the anti-fraud rejected every test card before the IPN ever fired, and `command=update` always reappeared after the injected `command=read`, with PHP's last-wins semantics for duplicate `$_GET` keys neutralizing the injection.
- Below is the live investigation, the dead ends, and where the chain remained incomplete.

## 1. Application surface

Three boxes in the same Kubernetes namespace:

| Component | URL | Role |
|---|---|---|
| Merchant | `http://incredibly-protected-notifications.ctf.thcon.party:8080` | checkout form, JWT issuer |
| PSP | `:8081/psp.php` | card form, fires the IPN |
| Internal merchant | `merchant.<ns>.svc.cluster.local:8080/processpayment.php` | only reachable from the PSP pod |

User flow: form → JWT → PSP → card validation → PSP calls IPN URL → user redirected back.

## 2. JWT and the IPN URL

The JWT (HS256, properly verified) carries the entire payment context, including the IPN URL:

```json
{
  "expiration": 1778190176,
  "amount": "100",
  "contractNumber": 31000,
  "ipn": "http://merchant.<ns>.svc.cluster.local:8080/processpayment.php?address=test&bill=2026001&amount=100&command=update",
  "bill": "2026001",
  "address": "test",
  "redirect": "http://incredibly-protected-notifications.ctf.thcon.party:8080/confirmation.php"
}
```

The IPN URL is built by **string concatenation**: form `address` → `?address=...&bill=...&amount=...&command=update`. So `address=test&command=read` injects a fresh `command` parameter:

```
?address=test&command=read&bill=2026001&amount=100&command=update
```

But there are now two `command` keys; PHP's `$_GET` keeps the **last** one (`update`).

## 3. Endpoint behavior

External access:

```bash
$ curl '...:8080/processpayment.php?command=read'                          # → 200, empty body
$ curl '...:8080/processpayment.php?address=test&bill=2026001&amount=100&command=read'   # → 500
```

The endpoint exists but is gated — likely an in-cluster source IP / hostname check. Only requests originating from the PSP container reach the production-key branch.

## 4. What was tried and failed

### 4.1 JWT attacks
- `alg: none` confusion → rejected.
- Common secret dictionary against HS256 → no hit.
- Header injection (`kid`, `jku`, `jwk`) → no parser took the bait.

### 4.2 Anti-fraud bypass
Every test card was rejected with HTTP 400 *before* the IPN was scheduled:

```
4242424242424242, 5555555555554444, 378282246310005, 4111111111111111, ...
```

The PSP appears to fire the IPN only after the auth succeeds — no auth, no IPN.

### 4.3 Parameter pollution variants

| Variant | Result |
|---|---|
| `&command=read` | duplicate; PHP picks `update` |
| `&command[]=read` | PHP array notation; not parsed as scalar |
| `#command=read` | fragment, never sent server-side |
| `%00&command=read` | null byte ignored |
| CRLF injection | filtered |

### 4.4 Reconnaissance dead-ends
- `robots.txt`, `.git/config`, `/admin`, debug endpoints on the PSP — all 404.
- Timing differences across `command` values — none significant.

## 5. The likely intended chain (best guess)

The pun in the title (*"Incredibly Protected Notifications"* / IPN) plus the `address` injection strongly suggests the answer is one of:

1. **A way to make `command=read` win the parameter race.** A specific PHP version's handling of `command[]=read&...&command=update`, or a CRLF/null trick the in-pod parser handles differently than the external one.
2. **A second injection further up the URL** that turns the rest of the IPN query string into the path/fragment — e.g. a `#` before `command=update`, depending on how PHP parses the URL it constructs.
3. **Bypassing anti-fraud** to reach the post-auth IPN trigger. The card-validation logic might trust a header (`X-Forwarded-For`, `X-Test`) when set in the JWT or the request.
4. **Exfiltration via the redirect URL.** If the IPN response body leaks through into the `confirmation.php` redirect (e.g. status, headers, cookies), the attacker gets the production key without the response ever returning to them directly.

None of the four were productive in the time available.

## 6. Methodology — what was learned

- **Webhook/IPN flows are SSRF magnets** when any part of the callback URL is user-controlled. Always test for parameter pollution in concatenated callback URLs.
- **Confirm the parameter parser before crafting payloads.** PHP's "last value wins" for duplicated `$_GET` keys is the opposite of, e.g., Node/Express's "first" or "array" semantics. The trick that worked on the same kind of bug last year may not work here.
- **Anti-fraud and rate-limit gates can hide the true bug.** When auth-required code paths can't be triggered, the real exploit may still live behind them — but the bypass becomes its own subproblem.
- **Internal-only endpoints (`*.svc.cluster.local`) are a tell for SSRF being intended.** The challenge architect put the secret behind in-cluster ACLs deliberately; the exploit pivot is somewhere upstream.
