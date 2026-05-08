# Forged Goods

- **Category:** Cryptography
- **Challenge ID:** 58
- **Service:** `nc 40.66.60.171 4244`

```
THC{tr0p1c4l_f4ct0r1z4t10n_1s_NP_h4rd_but_wh0_c4r3s}
```

## TL;DR

- The server publishes an 8×8 **tropical** (min-plus) matrix `T = X ⊗ Y` where `X` is 8×7, `Y` is 7×8, and entries are bytes in `[0, 255]`. Submit any valid factorization to get the flag.
- Tropical matrix factorization is NP-hard in general — but here the inner rank `K = 7` equals `N − 1`, so any valid `X` can use **7 of the 8 columns of `T` as its column basis**. Only `C(8, 7) = 8` bases to try.
- For each candidate `X`, recover `Y` by **tropical residuation** (`y_{k,j} = max_i(t_j[i] − X[i,k])`), check the product, then apply shift equivalence to push every entry into `[0, 255]`.
- The flag tells on the parameter choice: "*tropical factorization is NP-hard but who cares*."

## 1. Tropical algebra in one paragraph

The tropical (min-plus) semiring replaces ordinary addition and multiplication:

| Standard | Tropical |
|---|---|
| `a + b` | `min(a, b)` |
| `a × b` | `a + b` |
| identity for `+`: `0` | identity for `min`: `+∞` |
| identity for `×`: `1` | identity for `+`: `0` |

Matrix product follows the same shape, just with `min` outside and `+` inside:

```
(X ⊗ Y)_{i,j} = min_k ( X_{i,k} + Y_{k,j} )
```

Associative, non-commutative — good enough to build a one-way function on top of.

## 2. The service

```
[1] Get public key
[2] List orders
[3] Submit factorization
```

`[1]` gives `T` (8×8). `[3]` accepts a JSON `{"X": ..., "Y": ...}` and verifies `X ⊗ Y == T` with shape `(8×7) ⊗ (7×8)` and entries in `[0, 255]`.

## 3. Why the parameter choice is fatal

The "security" relies on tropical factorization being NP-hard. But every column `t_j` of `T` is a tropical combination of the 7 columns of `X`:

```
t_j[i] = min_k ( X[i,k] + Y[k,j] )
```

So all 8 columns of `T` lie in the tropical column span of just 7 vectors (the columns of `X`). Generically, **any 7 of `T`'s 8 columns will themselves form such a basis** — they're already linear combinations of the same 7 underlying vectors. We only need to find which 7 work.

That's `C(8, 7) = 8` candidates. Brute force is the entire attack.

## 4. The attack

### 4.1 Pick a candidate `X`

Choose 7 columns of `T`; treat that 8×7 submatrix as `X_candidate`.

### 4.2 Recover `Y` by tropical residuation

For each column `t_j` of `T`, find the coefficient column `y_{:,j}` (length 7) such that:

```
min_k ( X[i,k] + y_{k,j} ) == t_j[i]   for every i
```

The standard solution is **tropical residuation** — the largest `y_{k,j}` that doesn't make the min undershoot:

```
y_{k,j} = max_i ( t_j[i] − X[i,k] )
```

Pick this value per `(k, j)` and reconstruct the column. If any column doesn't match exactly, this basis fails — try the next.

### 4.3 Push entries into `[0, 255]` with shift equivalence

The recovered values may be negative or > 255. Tropical factorizations have a free gauge: for any constants `a_k`,

```
X'[i,k] = X[i,k] + a_k
Y'[k,j] = Y[k,j] - a_k
```

leaves `X' ⊗ Y' = X ⊗ Y` because the `a_k` cancels under `min`. For each `k`, the feasible interval is:

```
lo_k = max( -min(X[:,k]) ,  max(Y[k,:]) - 255 )
hi_k = min( 255 - max(X[:,k]) ,  min(Y[k,:]) )
```

Any `a_k ∈ [lo_k, hi_k]` works; if the interval is empty, skip this basis.

### 4.4 Fallback by transposition

If no column basis works, transpose: `T^T = Y^T ⊗ X^T` is the same problem with rows and columns swapped. Apply the same search to `T^T` and transpose back.

## 5. Solver

```python
#!/usr/bin/env python3
import itertools, json, re, socket

HOST, PORT = "40.66.60.171", 4244
N, K = 8, 7

def recv_until(sock, marker=b"> "):
    data = b""
    while marker not in data:
        chunk = sock.recv(65536)
        if not chunk:
            break
        data += chunk
    return data.decode()

def trop_mul(A, B):
    r1, c1 = len(A), len(A[0])
    c2 = len(B[0])
    return [[min(A[i][k] + B[k][j] for k in range(c1)) for j in range(c2)]
            for i in range(r1)]

def transpose(M):
    return [list(row) for row in zip(*M)]

def factor_columns(T):
    cols = [[T[i][j] for i in range(N)] for j in range(N)]
    for basis in itertools.combinations(range(N), K):
        X = [[cols[j][i] for j in basis] for i in range(N)]
        Y_cols, ok = [], True
        for target in cols:
            coeffs = [max(target[i] - X[i][k] for i in range(N)) for k in range(K)]
            if [min(X[i][k] + coeffs[k] for k in range(K)) for i in range(N)] != target:
                ok = False
                break
            Y_cols.append(coeffs)
        if not ok:
            continue
        Y = transpose(Y_cols)
        shifts = []
        feasible = True
        for k in range(K):
            xcol = [X[i][k] for i in range(N)]
            yrow = Y[k]
            lo = max(-min(xcol), max(yrow) - 255)
            hi = min(255 - max(xcol), min(yrow))
            if lo > hi:
                feasible = False
                break
            shifts.append(0 if lo <= 0 <= hi else lo)
        if not feasible:
            continue
        X2 = [[X[i][k] + shifts[k] for k in range(K)] for i in range(N)]
        Y2 = [[Y[k][j] - shifts[k] for j in range(N)] for k in range(K)]
        if trop_mul(X2, Y2) == T:
            return X2, Y2
    return None

def factor_tropical(T):
    direct = factor_columns(T)
    if direct:
        return direct
    swap = factor_columns(transpose(T))
    if swap:
        Xt, Yt = swap
        return transpose(Yt), transpose(Xt)
    raise RuntimeError("no basis")

def main():
    s = socket.create_connection((HOST, PORT))
    recv_until(s)
    s.sendall(b"1\n")
    out = recv_until(s)
    T = json.loads(re.search(r"Public key T \(8x8\):\n(\[.*\])\n", out, re.S).group(1))
    X, Y = factor_tropical(T)
    assert trop_mul(X, Y) == T

    s.sendall(b"3\n")
    recv_until(s)
    s.sendall((json.dumps({"X": X, "Y": Y}) + "\n").encode())

    blob = b""
    while b"THC{" not in blob:
        chunk = s.recv(65536)
        if not chunk:
            break
        blob += chunk
    print(re.search(r"THC\{[^}]+\}", blob.decode(errors="replace")).group(0))

if __name__ == "__main__":
    main()
```

```
$ python3 solve.py
THC{tr0p1c4l_f4ct0r1z4t10n_1s_NP_h4rd_but_wh0_c4r3s}
```

## 6. Methodology

- **Read the dimensions before reading the math.** A tropical scheme with inner rank `K = N − 1` is structurally broken: the column space of `T` is already rank `K`, so any `K` columns of `T` are a candidate basis.
- **Residuation is the universal "given a basis, find the coefficients" tool.** Whenever a tropical product hits a known target, the unique tightest coefficient is `max_i (target[i] − basis[i,k])`.
- **Don't forget the gauge.** A factorization that lives outside `[0, 255]` is still valid — applying a column/row shift fixes it without re-searching. Skipping this step looks like a "bug" in the search.
- **Tropical schemes generally fail to dimension/rank attacks rather than to deep tropical algebra.** When you see a tropical CTF, look at `(N, K)` first.
