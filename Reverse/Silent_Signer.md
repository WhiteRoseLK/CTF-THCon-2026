# Silent Signer

- **Category:** Reverse Engineering
- **Binary:** `sst-fwsign` (stripped x86-64 ELF, dynamically linked)

```
THC{int3_s3nt_u_h3r3_3bpf_t00k_1t_fr0m_th3r3!!!}
```

## TL;DR

- The host binary loads a 48-byte token, transforms it (XOR with a hardcoded table, then rotate-left by 7), and pushes the result through six BPF maps into an **eBPF** verifier program. The verifier lives inside the binary's `.data` section, encrypted with a single XOR-stream key derived from three immediates in the constructor.
- A second eBPF program is hooked on `tp/syscalls/sys_enter_ptrace` and silently corrupts the KDF map if anything calls `ptrace`. **Don't debug — analyse statically.**
- The verifier's per-block check is `rol64(kdf[i] * (acc ^ digest_i), 13) == target[i]`. All `kdf[i]` are odd, so the multiply is invertible mod 2⁶⁴. Each block is solved independently with `modinv`.
- Recover the 6 input qwords, pack little-endian, decode as ASCII → flag.

## 1. Triage

```
$ file sst-fwsign
ELF 64-bit LSB executable, x86-64, ..., stripped

$ readelf -S sst-fwsign | grep -E '\.(text|rodata|data)\b'
[15] .text     PROGBITS  0x401000  ...   12340  AX
[16] .rodata   PROGBITS  0x414000  ...    1240   A
[17] .data     PROGBITS  0x4553c0  ...   1f400  WA
```

`.data` is ~127 KB and full of high-entropy bytes — way too big for a normal `.data` of a small CLI. That's the encrypted eBPF blob.

```
$ strings sst-fwsign | grep -E 'fw_verify|integrity|ptrace|VALID'
fw_verify
integrity_watch
uprobe/fw_commit
tp/syscalls/sys_enter_ptrace
VALID
INVALID
```

Two BPF programs and their attach points are in cleartext. `fw_verify` does the validation; `integrity_watch` is the anti-debug.

## 2. Constructor — the key derivation

Disassembly of the `.init_array` constructor (Ghidra/r2 with stripped binary):

```
movabs rax, 0xcafebabedeadbeef        ; A
movabs rbx, 0xf69049cc7be24bd5        ; B
movabs rcx, 0x9bf0e8c145a8d663        ; C
xor    rax, rbx                       ; B ^ A
xor    rbx, rcx                       ; (used as A^C below)
sub    rax, rbx                       ; (B^A) - (A^C)  mod 2^64
xor    rax, 0x4141414141414141        ; ^ "AAAAAAAA"
; rax == decryption key
lea    rsi, [0x4553c0]                ; .data start
mov    rdx, blob_size
call   decrypt                        ; XORs every 8 bytes with the key
```

```python
A = 0xcafebabedeadbeef
B = 0xf69049cc7be24bd5
C = 0x9bf0e8c145a8d663
KEY = (((B ^ A) - (A ^ C)) & ((1<<64)-1)) ^ 0x4141414141414141
# KEY = 0xaa21e1b24b0bcdef
```

The decrypt routine is a textbook XOR stream:

```
xor QWORD [rsi], rax
add rsi, 8
dec rdx
jnz ...
```

## 3. Decrypting the blob

```python
import struct

with open("sst-fwsign", "rb") as f:
    binary = bytearray(f.read())

OFFSET, SIZE = 0x553c0, 0x1f400
key_bytes = struct.pack("<Q", 0xaa21e1b24b0bcdef)

for i in range(0, SIZE, 8):
    for j in range(8):
        binary[OFFSET + i + j] ^= key_bytes[j]

with open("bpf_object.elf", "wb") as f:
    f.write(binary[OFFSET:OFFSET + SIZE])
```

```
$ file bpf_object.elf
ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), not stripped
$ readelf -S bpf_object.elf | grep -E 'fw_verify|integrity|maps'
[ 1] uprobe/fw_commit               PROGBITS   ...
[ 2] tp/syscalls/sys_enter_ptrace   PROGBITS   ...
[ 3] .maps                          PROGBITS   ...
```

## 4. The two BPF programs

### 4.1 `integrity_watch` (anti-debug)

Pinned to `tp/syscalls/sys_enter_ptrace`. Body:

```c
for (i = 0; i < 6; i++)
    bpf_map_update_elem(kdf_map, &i, &random_junk, 0);
return 0;
```

Any `ptrace` syscall — **including from `gdb`, `strace`, `ltrace`, even `PTRACE_TRACEME` from a child** — clobbers the KDF map. After that, the verifier's arithmetic is computed against garbage and `INVALID` is returned forever (or worse: a different token is "valid").

The only safe path is full static analysis.

### 4.2 `fw_verify` (validator)

```c
acc = 0;
for (i = 0; i < 6; i++) {
    digest_i = *bpf_map_lookup_elem(input_map,  &i);   // host wrote here
    kdf_i    = *bpf_map_lookup_elem(kdf_map,    &i);
    target_i = *bpf_map_lookup_elem(target_map, &i);

    tmp = kdf_i * (acc ^ digest_i);                    // 64-bit wrapping
    if (rol64(tmp, 13) != target_i) return INVALID;
    acc ^= target_i;
}
return (acc == 0xaaf62074aad3ee0e) ? VALID : INVALID;
```

The host-side transform written into `input_map[i]` is:

```c
digest_i = rol64(xor_tbl[i] ^ input_qword[i], 7);
```

with `xor_tbl` and `kdf_map` constants extracted from `.rodata` and `.maps` respectively (FNV constants and SHA-256 initial hash values, used as obfuscation).

## 5. The constants

```
xor_tbl[6]  = a54ff53a3c6ef372  9b05688c510e527f
              1f83d9ab5be0cd19  bb67ae856a09e667
              e9b5dba558b1091b  71374491428a2f98

kdf[6]      = 6c62272e07bb0143  293d9ac069f8477d
              be5466cf34e90c6d  082efa98ec4e6c89
              452821e638d01377  9216d5d98979fb1b

target[6]   = 66185fcb3af43c42  fb9181fc9d741ac9
              f6f76d94d5f19c7c  9623be0fa7985447
              c801d5b2ee724650  9faaf86a914846ee

FINAL_ACC   = aaf62074aad3ee0e
```

Crucially, every `kdf[i]` is **odd** — so each is invertible mod 2⁶⁴.

## 6. Inversion

For each block in order:

| Step | Formula |
|---|---|
| 1. undo rotate | `tmp = ror64(target[i], 13)` |
| 2. undo multiply | `acc ^ digest = tmp * modinv(kdf[i], 2^64)` |
| 3. recover digest | `digest = acc ^ (acc ^ digest)` |
| 4. undo host rotate | `pre = ror64(digest, 7)` |
| 5. recover input | `input_q[i] = pre ^ xor_tbl[i]` |
| 6. advance | `acc ^= target[i]` |

After 6 blocks, `acc` should equal `FINAL_ACC` — that's the one-shot integrity check the BPF program runs at the end.

## 7. Solver

```python
#!/usr/bin/env python3
import struct

MASK = (1 << 64) - 1
MOD  = 1 << 64

def rol(v, n): return ((v << n) | (v >> (64 - n))) & MASK
def ror(v, n): return ((v >> n) | (v << (64 - n))) & MASK

KDF = [0x6c62272e07bb0143, 0x293d9ac069f8477d, 0xbe5466cf34e90c6d,
       0x082efa98ec4e6c89, 0x452821e638d01377, 0x9216d5d98979fb1b]

TARGETS = [0x66185fcb3af43c42, 0xfb9181fc9d741ac9, 0xf6f76d94d5f19c7c,
           0x9623be0fa7985447, 0xc801d5b2ee724650, 0x9faaf86a914846ee]

XOR_TBL = [0xa54ff53a3c6ef372, 0x9b05688c510e527f, 0x1f83d9ab5be0cd19,
           0xbb67ae856a09e667, 0xe9b5dba558b1091b, 0x71374491428a2f98]

FINAL_ACC = 0xaaf62074aad3ee0e

acc, qwords = 0, []
for i in range(6):
    tmp     = ror(TARGETS[i], 13)
    digest  = acc ^ (tmp * pow(KDF[i], -1, MOD) % MOD)
    qwords.append(XOR_TBL[i] ^ ror(digest, 7))
    acc    ^= TARGETS[i]

assert acc == FINAL_ACC
print(struct.pack("<6Q", *qwords).decode())
```

```
$ python3 solve.py
THC{int3_s3nt_u_h3r3_3bpf_t00k_1t_fr0m_th3r3!!!}
```

The flag itself is the spoiler: `int3` (debugger interrupt) sent you to the binary, then **eBPF took it from there**.

## 8. Methodology

- **Oversized `.data` in a stripped CLI = encrypted blob.** A 127-KB `.data` for a tool that prints `VALID/INVALID` is a screaming flag for an embedded payload. `xxd` it for entropy before assuming "it's just data".
- **Anti-debug via `tp/sys_enter_ptrace`.** Any kernel-side BPF tracepoint on ptrace can corrupt program state on debug. The defense is asymmetric: the user has to debug, the program just has to wait. Never `gdb` such a binary; never `strace` it; static-only.
- **Odd multipliers mod 2⁶⁴ are free wins.** A "non-linear mixing" step that multiplies by an odd 64-bit constant is not non-linear over `Z/2⁶⁴` — `pow(k, -1, 2**64)` undoes it instantly.
- **The accumulator-equality check at the end is a built-in integrity oracle.** Use it as a sanity check after inversion: if the computed `acc` doesn't match the constant, one of your block constants is wrong.
- **Authors love symbolic constants.** `xor_tbl` is the SHA-256 IV; `kdf` is FNV. Identifying these reduces "where did this number come from" anxiety to "purely cosmetic obfuscation, ignore the storyline."
