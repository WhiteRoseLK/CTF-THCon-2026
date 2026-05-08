# PNG is a Lie

- **Category:** Steganography
- **Challenge ID:** 28
- **Points:** 50

```
THC{PNG3D}
```

## TL;DR

- `weird_file.thc` is a UTF-8 text file mixing ASCII letters with millions of `👍` / `👎` emojis. The ASCII is decoy; the emojis are the payload.
- Map `👍 → 1`, `👎 → 0`. Concatenate, pack into bytes. The first 8 bytes are `89 50 4E 47 0D 0A 1A 0A` — the PNG signature.
- Reconstruct the PNG (1000×1000), OCR the visible text → `THC{PNG3D}`.

## 1. Inspecting the carrier

```bash
$ file weird_file.thc
weird_file.thc: Unicode text, UTF-8 text

$ head -c 200 weird_file.thc
asdfghjk👎👍👎👎👎👎👎👍👎👍👎👍👎👎👎👎👎👍👎👎👎👎👎👎👎👍👎👎👍👎👎👎...
```

Two distinct emojis, plus filler ASCII letters scattered through. A `python -c` count of each emoji shows hundreds of thousands per glyph — way too high to be incidental.

## 2. The mapping

A natural binary encoding gives two candidates for the bit assignment. The signature test settles it instantly:

```python
from pathlib import Path

text = Path("weird_file.thc").read_text("utf-8")
emojis = [c for c in text if c in "👍👎"]
bits = "".join("1" if c == "👍" else "0" for c in emojis)
data = bytes(int(bits[i:i+8], 2) for i in range(0, len(bits) - len(bits) % 8, 8))

print(data[:8].hex())   # 89 50 4E 47 0D 0A 1A 0A   →  PNG
```

`👍 = 1`, `👎 = 0`, and the file is a PNG. The other mapping yields garbage — verifying the bit choice by magic-number alignment is how you confirm it.

## 3. Recovering the image and reading the flag

```python
Path("extracted.png").write_bytes(data)
```

Result: a 1000×1000 PNG with the flag rendered as visible text on the canvas. Either visually or via OCR:

```python
from PIL import Image
import pytesseract, io

print(pytesseract.image_to_string(Image.open(io.BytesIO(data))).strip())
# THC{PNG3D}
```

## 4. Methodology

- **When a "weird" text file has a strict 2-symbol alphabet, treat it as binary.** Two emojis, dots vs. dashes, capitals vs. lowercase — the alphabet size is the giveaway.
- **Confirm the bit-direction with a magic number.** Don't guess `👍 = 1` vs `0`; pack 8 bytes both ways and check whether either looks like a known file header (PNG, ZIP, ELF, JPEG, PDF…). One choice will produce signal; the other entropy.
- **Strip the noise before bit-packing.** ASCII filler is decoy. Filter the input down to the emoji alphabet before grouping, otherwise the byte boundaries shift and the magic number disappears.
- **The flag may live inside the recovered file, not as text.** A reconstructed PNG / ZIP / PDF still needs to be opened or parsed — OCR is just one way; sometimes there's a second layer (binwalk, hidden chunks, EXIF) underneath.
