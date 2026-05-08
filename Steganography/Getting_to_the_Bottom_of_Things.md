# Getting to the Bottom of Things

- **Category:** Steganography
- **Challenge ID:** 44

```
THCon{TMTC_B1nwalk_D3t3ct3d}
```

## TL;DR

- The challenge reuses the BitLocker volume from "Don't forget to lock" (Ch. 31). After unlocking that volume, `TOPSECRET.pdf` carries the next stage.
- The PDF is a valid PDF — *and* has a ZIP archive **appended after `%%EOF`**. `binwalk` flags it instantly.
- Carve from the first `PK\x03\x04` to the end of the EOCD record, trim post-EOCD garbage, unzip, read `flag.txt`.

## 1. Locating the carrier

Pull the file out of the decrypted NTFS image (carry-over from Ch. 31):

```bash
$ fls -r -f ntfs files/decrypted.raw | grep -i pdf
r/r 48-128-3:  TOPSECRET.pdf
r/r 62-128-1:  vitlocker.pdf

$ icat -f ntfs files/decrypted.raw 48-128-3 > TOPSECRET.pdf
$ ls -lh TOPSECRET.pdf
-rw-r--r--  1 user  user  ~1.4M  TOPSECRET.pdf
```

## 2. Smell test

A quick `binwalk` is the dead giveaway:

```bash
$ binwalk TOPSECRET.pdf
DECIMAL    HEXADECIMAL    DESCRIPTION
0          0x0            PDF document, version: "1.7"
112395     0x1B70B        End of PDF (%%EOF)
112401     0x1B711        Zip archive data, at least v2.0 to extract
                          file: flag.txt
                          file: bookmarks.csv
                          file: maintenance_log_2125.log
                          file: TODOLIST.txt
                          file: coffee_debt.csv
                          file: firmware_backup.bin
                          file: file1_declassified.pdf
                          file: file2_declassified.pdf
                          file: file3.pdf
```

A ZIP starts 6 bytes past the PDF's `%%EOF`. PDF readers ignore everything after `%%EOF`, ZIP readers find their archive by scanning for `PK\x05\x06` from the end, so both formats coexist in one file.

## 3. Carving the ZIP cleanly

`unzip` will *usually* tolerate the prefix bytes, but trailing garbage after the EOCD record can break some tools. Carve precisely:

```python
from pathlib import Path

data = Path("TOPSECRET.pdf").read_bytes()
zip_start = data.find(b"PK\x03\x04")
zip_data  = data[zip_start:]

# End of Central Directory: PK\x05\x06 + 18 bytes of fields + 2-byte comment length + comment
eocd        = zip_data.rfind(b"PK\x05\x06")
comment_len = int.from_bytes(zip_data[eocd + 20: eocd + 22], "little")
trimmed     = zip_data[: eocd + 22 + comment_len]

Path("topsecret.zip").write_bytes(trimmed)
```

Then:

```bash
$ unzip -p topsecret.zip flag.txt
THCon{TMTC_B1nwalk_D3t3ct3d}
```

## 4. Methodology

- **`binwalk` first, `file` second.** A PDF that's silent on `file -k` will still light up `binwalk`, which scans for embedded magic numbers anywhere in the stream — exactly the case for "valid format A + appended format B".
- **Carve to the EOCD, not just to the file end.** ZIP readers scan backwards for `PK\x05\x06`, but they want the right comment length to follow it. `data[eocd + 22 + comment_len:]` is junk you should drop.
- **PDF + ZIP = polyglot.** PDF readers stop at `%%EOF`; ZIP readers start from `PK\x05\x06` near the end. The two parsers don't overlap, so a single file can satisfy both.
- **Always check chained challenges for shared artifacts.** This PDF only became readable because Ch. 31's BitLocker unlock landed `decrypted.raw`. Re-mount, re-list, re-carve before assuming the new challenge ships its own bundle.
