# Min Max

- **Category:** Cryptography
- **Service:** `nc 51.103.57.72 4243`

```
THC{fl0yd_w4rsh4ll_m33ts_crypt0gr4phy_1n_th3_tr0p1cs}
```

## TL;DR

- The server "encrypts" each plaintext block `p` of length `N = 8` with a tropical (min-plus) matrix product: `c[i] = min_j (K[i][j] + p[j])`.
- This map has a closed-form inverse — **tropical residuation**: `p[j] = max_i (c[i] − K[i][j])`. No search, no oracle, no key recovery.
- Pull `K` and `ct` from the menu, residuate every block, submit the flat integer list, read the flag.

## 1. Tropical encryption in three lines

The semiring just relabels:

| Standard | Tropical |
|---|---|
| `a + b` | `min(a, b)` |
| `a × b` | `a + b` |

A "matrix-vector product" inherits the same shape with `min` outside and `+` inside:

```
c[i] = min_j ( K[i][j] + p[j] )
```

That's the entire encryption. Apply it block by block to the plaintext (length-`N` blocks; the server pads with zeros if needed) to produce the ciphertext blocks `ct[0], ct[1], ...`.

## 2. Why this is invertible

For each fixed `j`, the term `K[i][j] + p[j]` is one of the candidates the `min` runs over, so:

```
c[i]  ≤  K[i][j] + p[j]      for every i
⟹  p[j]  ≥  c[i] − K[i][j]   for every i
```

Take the tightest lower bound across all `i`:

```
p[j]  ≥  max_i ( c[i] − K[i][j] )
```

The classical residuation result says equality is exactly the right value: setting `p[j]` to that maximum reproduces `c` under the tropical product. If the original ciphertext came from any plaintext at all, this formula recovers it.

### Sanity check

```
N = 3
K = [[10, 20,  5],   p = [2, 7, 0]
     [ 3,  8, 15],
     [12,  1,  9]]

c[0] = min(10+2, 20+7,  5+0)  = 5
c[1] = min( 3+2,  8+7, 15+0)  = 5
c[2] = min(12+2,  1+7,  9+0)  = 8

Residuation:
p[0] = max(5−10, 5−3, 8−12)  = max(−5,  2, −4) = 2  ✓
p[1] = max(5−20, 5−8, 8−1)   = max(−15,−3,  7) = 7  ✓
p[2] = max(5−5,  5−15,8−9)   = max( 0,−10, −1) = 0  ✓
```

## 3. Service

```
1. Print K and ciphertext
2. Submit plaintext
>
```

`[1]` dumps `K: [[…]]` and `ct: [[…], [...]]` as JSON arrays. `[2]` prompts `plaintext key>` and accepts the **flat** concatenation of all decrypted blocks as a JSON list.

## 4. Solver

```python
#!/usr/bin/env python3
import json, re, socket

HOST, PORT, N = "51.103.57.72", 4243, 8

def recv_until(s, m):
    buf = b""
    while m not in buf:
        c = s.recv(4096)
        if not c:
            break
        buf += c
    return buf

def solve_block(K, c):
    return [max(c[i] - K[i][j] for i in range(N)) for j in range(N)]

def main():
    with socket.create_connection((HOST, PORT)) as s:
        recv_until(s, b"> ")
        s.sendall(b"1\n")
        info = recv_until(s, b"> ").decode()
        K = json.loads(re.search(r"K: (\[.*\])", info).group(1))
        ct = json.loads(re.search(r"ct: (\[.*\])", info).group(1))

        plaintext = []
        for block in ct:
            plaintext.extend(solve_block(K, block))

        # The decoded bytes already contain the flag — but the server is
        # the source of truth, so submit and print whatever it returns.
        s.sendall(b"2\n")
        recv_until(s, b"key> ")
        s.sendall((json.dumps(plaintext) + "\n").encode())
        print(s.recv(4096).decode(errors="replace").strip())

if __name__ == "__main__":
    main()
```

```
$ python3 solve.py
THC{fl0yd_w4rsh4ll_m33ts_crypt0gr4phy_1n_th3_tr0p1cs}
```

## 5. Methodology

- **Tropical multiplication is not a one-way function.** It's the operation behind Floyd-Warshall shortest paths — fully invertible by residuation. Treating it as crypto without adding noise or ambiguity means the inverse is closed-form.
- **Flag-name as hint:** *"floyd_warshall_meets_cryptography_in_the_tropics"* — the author is openly pointing at the algorithm whose recurrence is identical to the encryption.
- **Whenever a CTF "encryption" is `c = K op p` for some semiring op, write down `op`'s left-inverse before doing anything else.** If the inverse is closed-form, you have nothing left to attack — just decrypt and submit.
