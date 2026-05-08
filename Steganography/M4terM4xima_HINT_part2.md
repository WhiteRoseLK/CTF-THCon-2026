# M4terM4xima HINT (part 2 — steganography)

- **Category:** Steganography
- **Challenge ID:** 46

```
THC{lui zero, ox123}
```

## TL;DR

- The artifact is a 32-bit RISC-V ELF (`HINT.elf`). The challenge name nudges you at **RISC-V HINT instructions** — instructions that write to `x0`, which the architecture defines as having no effect, leaving the immediate operand free as a covert channel.
- 20 mystery bytes sit in `.rodata` between two `Are you sure ...` strings, plain hex with no printable structure.
- Decoding is a **chain XOR** with seed `0x55`: `decoded[i] = mystery[i] ^ (i == 0 ? seed : decoded[i-1])`.
- Output: `THC{lui zero, ox123}` — itself a RISC-V HINT instruction (`lui x0, 0x123`).

## 1. ELF triage

```
$ file HINT.elf
ELF 32-bit LSB executable, UCB RISC-V, RVC, soft-float ABI, statically linked, not stripped

$ readelf -s HINT.elf | grep -E 'HINT|maybe'
... 80000248  72  FUNC  HINT
... 80000470  958 FUNC  maybe_HINT
... HINT_PTR  (riscv_hint::HINT_PTR static)
```

`maybe_HINT` is the heavy function and `HINT_PTR` is suspicious. The binary size matches the ELF structure exactly — no appended payload.

## 2. The mystery bytes

Inside `.rodata`, between two helpfully-distinct strings:

```
"Are you sure that you are looking for HINT?\n"
01 1c 0b 38 17 19 1c 49  5a 1f 17 1d 43 0c 4f 17  49 03 01 4e
"Are you sure this is a HINT?\n"
```

20 bytes. None printable. Not the start of an embedded format. Tightly aligned between strings, suggesting deliberate placement rather than alignment padding.

## 3. Decoding — chain XOR

The flag must start with `THC{`. So `decoded[0..3]` are `T H C {`. Reading the first byte of the mystery, `0x01`:

```
0x01 ^ ? = 'T' = 0x54   →   ? = 0x55
```

That's the seed. Try a chain-XOR with each byte using the previous **decoded** value (CBC-style):

```
decoded[0] = 0x01 ^ 0x55 = 0x54 = 'T'
decoded[1] = 0x1c ^ 0x54 = 0x48 = 'H'
decoded[2] = 0x0b ^ 0x48 = 0x43 = 'C'
decoded[3] = 0x38 ^ 0x43 = 0x7b = '{'
decoded[4] = 0x17 ^ 0x7b = 0x6c = 'l'
decoded[5] = 0x19 ^ 0x6c = 0x75 = 'u'
decoded[6] = 0x1c ^ 0x75 = 0x69 = 'i'
decoded[7] = 0x49 ^ 0x69 = 0x20 = ' '
...
decoded[19] = 0x4e ^ 0x33 = 0x7d = '}'
```

The chain stays printable end-to-end → the seed and scheme are both right.

## 4. Solver

```python
mystery = bytes.fromhex("011c0b3817191c495a1f171d430c4f174903014e")
seed = 0x55

out, prev = [], seed
for b in mystery:
    d = b ^ prev
    out.append(d)
    prev = d

print(bytes(out).decode())
```

```
THC{lui zero, ox123}
```

## 5. Why the flag is itself the hint

`lui x0, 0x123` is a RISC-V `LUI` (Load Upper Immediate) targeting register `x0`. Per the spec, writes to `x0` are discarded — so the instruction is **architecturally defined as a HINT**, doing nothing visible while still encoding a 20-bit immediate. That immediate is the covert payload. The challenge name `M4terM4xima HINT`, the binary's `HINT` symbol, and the flag itself are all the same nudge.

## 6. Methodology

- **Plaintext-prefix recovery cracks any single-seed scheme.** When the flag format is known (`THC{`), every monoalphabetic / chain-XOR / additive scheme reveals its seed in 1–2 bytes.
- **Look between strings, not just inside them.** Compilers don't pad random bytes between consecutive `.rodata` strings — anything sitting there with non-printable content is intentional.
- **Read the challenge name as a primary clue.** `HINT` in a RISC-V binary is not just flavor; it's the ISA term for "instruction whose result is intentionally discarded". The author is naming the technique.
- **Chain XOR (`d[i] = c[i] ^ d[i-1]`) is the cheapest stream cipher with the same plaintext-recovery profile as XOR.** Useful pattern to recognize: encoded bytes have a different distribution from plaintext, but consecutive ciphertext-byte differences mirror the plaintext.
