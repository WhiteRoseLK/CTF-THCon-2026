# Exponope

- **Category:** Cryptography
- **Challenge ID:** 49
- **Points:** 50

```
THC{u_n3eD_@_bett3r_eXp0neNT}
```

## TL;DR

- A `.crypt` file gives you `N` (2047-bit modulus) and `cyphertext`. The public exponent `e` is **deliberately omitted**.
- The plaintext is a flag (~30 ASCII bytes ≈ 240 bits). With any reasonable small `e`, the integer `m^e` never exceeds `N`, so `c = m^e mod N` reduces to `c = m^e` over the integers.
- Take an exact integer e-th root of `c` for `e ∈ {2, 3, 5, 7, ...}`. For `e = 5`, the root is exact and decodes to printable ASCII — that's the flag.

## 1. Reading the artifact

The single file is just two labelled hex integers:

```
$ cat vewy-much-mysterious-file-such-encryptationnly-encrypted.crypt
N = 0x6361956d13a35165...
cyphertext = 0xfd7d893b965ca58d...
```

| Field | Meaning |
|---|---|
| `N` | RSA public modulus, 2047 bits |
| `cyphertext` | `c = m^e mod N` |

What's missing is the smoking gun: there is no `e`. The whole challenge points at the exponent.

## 2. Why a missing `e` is fatal here

Standard RSA encryption is `c = m^e mod N`. The `mod N` step is what makes it one-way: `m` is hidden because the result wraps around an enormous space.

That wrap **only happens when `m^e ≥ N`**. If the message is small and the exponent is small, the integer `m^e` is genuinely smaller than `N`, the reduction never fires, and the ciphertext is just an ordinary integer power:

```
m^e mod N  =  m^e          when m^e < N
```

At that point RSA is degenerate — the attacker just takes the e-th integer root of `c` and reads the flag. No private key, no factoring, no oracle. This is the classic **small-`e` / short-message** failure (a flavor of Håstad).

For this challenge, the flag is 30 ASCII characters → 240 bits. With `e = 5`, `m^5` is at most `5 × 240 = 1200` bits, comfortably below the 2047-bit modulus. The message size never makes it across the wrap line.

## 3. The wrap-around safety net

If we got slightly unlucky and the message *did* wrap once or twice, the relation becomes:

```
m^e = c + k·N    for some small non-negative integer k
```

So before taking the root, we try `k = 0, 1, 2, 3` and re-check exactness. In this challenge `k = 0` is enough, but the search is free.

## 4. Big-integer e-th root

Python `int` is unbounded, but `float` is only ~53 bits — useless on a 2047-bit number. We need an **exact integer root** computed entirely in arbitrary precision. Binary search does it cleanly:

```python
def integer_nth_root(n: int, k: int) -> int:
    """Floor of n^(1/k), via binary search in big-int arithmetic."""
    lo, hi = 0, 1
    while hi ** k <= n:        # double until hi^k > n
        hi <<= 1
    while lo + 1 < hi:
        mid = (lo + hi) // 2
        if mid ** k <= n:
            lo = mid
        else:
            hi = mid
    return lo                  # lo^k <= n < (lo+1)^k
```

After the search, verify `lo ** k == n` to confirm exactness.

## 5. Solver

```python
#!/usr/bin/env python3
from pathlib import Path

CRYPT = Path('vewy-much-mysterious-file-such-encryptationnly-encrypted.crypt')

def integer_nth_root(n: int, k: int) -> int:
    lo, hi = 0, 1
    while hi ** k <= n:
        hi <<= 1
    while lo + 1 < hi:
        mid = (lo + hi) // 2
        if mid ** k <= n:
            lo = mid
        else:
            hi = mid
    return lo

def parse(path: Path):
    vals = {}
    for line in path.read_text().splitlines():
        if '=' not in line:
            continue
        k, v = [p.strip() for p in line.split('=', 1)]
        vals[k.lower()] = int(v, 16)
    return vals['n'], vals['cyphertext']

def main():
    n, c = parse(CRYPT)
    print(f'[*] N = {n.bit_length()} bits')
    for e in (2, 3, 5, 7, 11):
        for k in range(4):
            candidate = c + k * n
            m = integer_nth_root(candidate, e)
            if pow(m, e) == candidate:
                data = m.to_bytes((m.bit_length() + 7) // 8, 'big')
                if all(32 <= b < 127 for b in data):
                    print(f'[+] e={e}, k={k} → {data.decode()}')
                    return
    print('[-] no exact-root candidate found')

if __name__ == '__main__':
    main()
```

Output:

```
[*] N = 2047 bits
[+] e=5, k=0 → THC{u_n3eD_@_bett3r_eXp0neNT}
```

The flag itself is the spoiler — `u_n3eD_@_bett3r_eXp0neNT`.

## 6. Methodology

- **When a public RSA value is "missing" from a CTF artifact, the missing piece is usually the bug.** The challenge author gives you everything except the parameter that makes the system unsafe.
- **Estimate `m^e` vs. `N` before doing anything else.** A 30-byte flag is 240 bits; even `e = 11` keeps you under a 2047-bit modulus. If it fits, the modular reduction never fires.
- **Always combine exponent-sweep with a small `k` sweep.** The `m^e = c + k·N` form costs nothing to try and rescues the few cases where the message just barely wraps.
- **Use big-int binary search for roots.** `math.isqrt` works for `e = 2`, but for general `e` you need to roll your own — `**` plus binary search is enough.
