# Breach at SST — 3

- **Category:** Forensics
- **Challenge ID:** 8

```
THCON{sp3ctr4l_p34ks_d0nt_l13}
```

## TL;DR

- The LUKS volume from part 2 holds two unfamiliar files: `intercept.wav` (mono, 16-bit PCM, 44.1 kHz, ~127 s) and `sigdb` (5.7 MB binary "signal database").
- `sigdb` is a flat table of 10-byte records `<u16 F0, F1, F2, F3, ascii>`. The character is uniquely keyed by `(F0, F1)` — `F2`, `F3` are dead.
- The audio is **60 short tone bursts in 30 pairs**. With a fixed 1024-point FFT (43.07 Hz/bin), each tone's peak bin is one of `F0` or `F1`. Look up `(F0, F1)` in `sigdb` per pair → one character.
- Decoded message: `THCON{sp3ctr4l_p34ks_d0nt_l13}` — *"spectral peaks don't lie."*

## 1. The two leftover files

```bash
$ ls /mnt/vault/
flag.txt   intercept.wav   sigdb   README_DIMITRI.txt   vault_note.txt
$ file intercept.wav sigdb
intercept.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, mono 44100 Hz
sigdb:         <misidentified — see below>
```

`file sigdb` reports a Targa image — false positive. The size (5,708,280 B) is a giveaway: `5_708_280 / 10 == 570_828` records of exactly 10 bytes.

## 2. Reverse-engineering `sigdb`

10 bytes per record matches `<HHHHH>` (5 little-endian `u16`s):

```c
struct entry {
    uint16_t F0;     // first FFT bin index
    uint16_t F1;     // second FFT bin index
    uint16_t F2;     // unused (in the lookup that matters)
    uint16_t F3;     // unused
    uint16_t F4;     // ASCII code of the encoded character
};
```

Quick exploration in Python:

```python
import struct
data = open("sigdb", "rb").read()
recs = [struct.unpack("<5H", data[i:i+10]) for i in range(0, len(data), 10)]
chars = {r[4] for r in recs if 32 <= r[4] <= 126}
# {'T','H','C','O','N','{','}',...}  – 58 unique printable ASCII
```

Field ranges, observed across the table:

| Field | Range | Role |
|---|---|---|
| `F0` | 10–209 | FFT bin index |
| `F1` | 10–209 | FFT bin index |
| `F2`, `F3` | varied | unused for lookup |
| `F4` | printable ASCII | output character |

The clinching observation: for every printable ASCII character `c` produced by the table, **`(F0, F1) → c`** is a function — different `(F2, F3)` rows with the same `(F0, F1)` always agree on `F4`. We only need a `(F0, F1) → char` dict:

```python
lookup = {(r[0], r[1]): chr(r[4]) for r in recs if 32 <= r[4] <= 126}
```

Bin pairs cluster in twos: `(20,21) → 'T'`, `(24,25) → 'H'`, `(28,29) → 'C'`, `(30,31) → 'O'`, `(44,45) → 'N'`, `(54,55) → '{'`, …, `(188,189) → '}'`. The encoding pre-pairs adjacent bins per character.

## 3. Reading the audio

Plot or RMS-window the waveform: 60 tone bursts of ~30–40 ms, organized in 30 pairs. Within a pair, the two tones are ~1 s apart; pairs are ~4 s apart. Frequencies climb monotonically from ~870 Hz to ~8130 Hz across the message.

That 200-bin range maps cleanly to the `F0/F1 ∈ [10, 209]` range in `sigdb` once we pick the FFT size. The challenge frequency resolution `~43 Hz` corresponds to `44100 / 1024`, so:

```
FFT_SIZE = 1024
bin = freq_Hz * FFT_SIZE / sample_rate
```

Each tone → one bin → matches one of `F0`/`F1` per pair.

## 4. Decoder

```python
#!/usr/bin/env python3
import wave, struct, numpy as np

# 1. Load sigdb -> (F0, F1) -> char
data = open("sigdb", "rb").read()
lookup = {}
for i in range(0, len(data), 10):
    f0, f1, _, _, ascii_code = struct.unpack("<5H", data[i:i+10])
    if 32 <= ascii_code <= 126:
        lookup[(f0, f1)] = chr(ascii_code)
        lookup[(f1, f0)] = chr(ascii_code)   # tolerate swapped order

# 2. Load audio
with wave.open("intercept.wav", "rb") as w:
    sr = w.getframerate()
    samples = np.frombuffer(w.readframes(w.getnframes()), dtype=np.int16).astype(float)

# 3. Segment by RMS energy
WIN, THR = 441, 500   # ~10 ms windows, empirical threshold
segments, in_seg, seg_start = [], False, 0
for i in range(0, len(samples) - WIN, WIN):
    rms = np.sqrt(np.mean(samples[i:i+WIN] ** 2))
    if rms > THR and not in_seg:
        in_seg, seg_start = True, i
    elif rms <= THR and in_seg:
        in_seg = False
        segments.append(seg_start)

# 4. FFT each segment, peak bin -> F0/F1, lookup pairwise
N = 1024
out = []
for i in range(0, len(segments) - 1, 2):
    a = samples[segments[i]   : segments[i]   + N]
    b = samples[segments[i+1] : segments[i+1] + N]
    f0 = int(np.argmax(np.abs(np.fft.rfft(a, n=N))))
    f1 = int(np.argmax(np.abs(np.fft.rfft(b, n=N))))
    out.append(lookup.get((f0, f1), "?"))

print("".join(out))
```

```
$ python3 decode.py
THCON{sp3ctr4l_p34ks_d0nt_l13}
```

The first six pairs land on `T H C O N {`, exactly as the bin clusters predicted.

## 5. Methodology

- **`file` is a starting point, not a verdict.** A "Targa image" of 5.7 MB that's exactly divisible by a small integer is almost always a flat record table. Compute `size mod k` for `k ∈ [4, 32]` to find the record size.
- **Strip a binary table to its smallest determining key.** Out of 5 fields, only 2 mattered. Verifying `(F0, F1) → F4` is functional saves time later — no inverse search, no ambiguity.
- **Match FFT bin width to the data, not to a default.** A "pretty" 4096-point FFT here would smear adjacent bins of the bin-pair encoding; 1024 (43 Hz/bin) is what the encoder actually used.
- **When the distfile is missing, look at sibling challenges.** `intercept.wav` and `sigdb` showed up only because part 2's LUKS unlock made them readable — chained challenges share artifacts.
