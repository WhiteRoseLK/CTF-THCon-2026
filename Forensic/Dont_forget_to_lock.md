# Don't Forget to Lock

- **Category:** Forensics
- **Challenge ID:** 31

```
THCON{v1tl0ck3r_1n_MEm}
```

## TL;DR

- The drop is two files: a **BitLocker AES-XTS-128** disk image (`disk.raw`) and a Windows 11 RAM dump (`dump.elf`, ELF core wrapping physical memory).
- BitLocker stores the FVEK (16 B) and the tweak key (16 B) in unprotected kernel pools while the volume is mounted. A memory dump of a running/sleeping machine leaks them.
- Run a Volatility 3 plugin that scans pool allocations for valid AES-128 expanded key schedules. One candidate validates against the volume's first sector by producing the `NTFS` magic.
- Decrypt with `dislocker -V disk.raw --fvek <file>`, walk the NTFS with Sleuth Kit, `icat` `flag1.txt`.

## 1. The artifacts

```bash
$ file files/disk.raw
DOS/MBR boot record; partition 1: ID=0x17 (Hidden NTFS WinNT)

$ file files/dump.elf
ELF 64-bit LSB core file x86-64

$ bdeinfo files/disk.raw
Volume information:
  Encryption method: AES-XTS 128-bit
  Encryption state:  Fully Encrypted
  Unlock method:     Password
```

AES-XTS-128 means **two** 16-byte keys: the FVEK (data) and a tweak key (used to encrypt the per-sector tweak). Both must be recovered.

## 2. Why memory leaks the keys

When BitLocker mounts a volume, the AES-NI driver expands the FVEK and tweak key into key schedules and parks them in kernel pool allocations tagged with markers like `FVEc`, `dFVE`, `Cngb`. The expanded schedule has **structural redundancy** that scanning can exploit: every 4-byte word past the first round is derivable from earlier words via the Rijndael S-box and Rcon constants. A correct schedule is therefore self-consistent — any random 176-byte slice almost never is.

So the attack is: enumerate kernel pool allocations, slide a 176-byte window over each, and check whether the bytes form a valid AES-128 expanded key.

## 3. Volatility 3 — pool scan for FVEK

```bash
$ vol -f files/dump.elf windows.info | head
Kernel Base    0xf80002000000
NtBuildLab     22621.1.amd64fre.ni_release.220506-1250
...

$ vol --plugin-dirs ./plugins -f files/dump.elf windows.bitlocker_fvek_scan
Pool Address          FVEK (hex)                                                          Notes
0xa30bf013d000        d21eecfa0ae20e8036f504bec32d36a29682a727a1037aa6be99671378e1668e   candidate
0xa30bf04dacc0        d21eecfa0ae20e8036f504bec32d36a29682a727a1037aa6be99671378e1668e   candidate
0xa30bf0651aa0        d21eecfa0ae20e8036f504bec32d36a29682a727a1037aa6be99671378e1668e   candidate
0xa30bf0760510        93c6450bf0226b49a531fada4f1f5e45fffddbda17fd86e8feff378a21c86142   ← VALID
```

The plugin emits a `*-Dislocker.fvek` file per candidate (Dislocker's binary FVEK format: `[u16 type][16 B FVEK][16 B tweak][...]`).

## 4. Validating a candidate

Decrypt sector 0 of the volume and look for the NTFS BPB:

```python
from Crypto.Cipher import AES

raw = open("0xa30bf0760510-Dislocker.fvek", "rb").read()
fvek_key  = raw[2:18]    # 16 B
tweak_key = raw[18:34]   # 16 B

with open("files/disk.raw", "rb") as f:
    sector0 = f.read(512)

c = AES.new(fvek_key + tweak_key, AES.MODE_XTS)
plain = c.decrypt(sector0)
print(plain[:8].hex())   # eb 52 90 4e 54 46 53 20  → 'NTFS '  ✓
```

`eb 52 90 NTFS` is the Windows NTFS boot sector signature — the right candidate is the one that produces this header. The other three candidates are stale schedules from earlier mounts, scrubbed allocations, or unrelated AES contexts.

## 5. Decrypting the volume

```bash
$ mkdir -p mnt_dl
$ dislocker -V files/disk.raw --fvek 0xa30bf0760510-Dislocker.fvek -- mnt_dl/
$ ls mnt_dl
dislocker-file
```

`mnt_dl/dislocker-file` is the decrypted volume exposed as a virtual block device. If FUSE isn't available (unprivileged container), `dislocker-file` writes the same image to disk:

```bash
$ dislocker-file -V files/disk.raw --fvek 0xa30bf0760510-Dislocker.fvek -v > files/decrypted.raw
```

## 6. Pulling the flag from NTFS

No mount needed; Sleuth Kit walks NTFS directly:

```bash
$ fsstat files/decrypted.raw | head -1
File System Type: NTFS

$ fls -r -f ntfs files/decrypted.raw | grep -iE 'flag|topsecret|events'
r/r 48-128-3:  TOPSECRET.pdf
r/r 51-128-1:  events.log
r/r 59-128-1:  flag1.txt
r/r 62-128-1:  vitlocker.pdf

$ icat -f ntfs files/decrypted.raw 59-128-1
THCON{v1tl0ck3r_1n_MEm}
```

`events.log` has the bonus story: the user opened the secrets, walked away without locking, and the dump was taken while the volume was still mounted.

## 7. Methodology

- **AES-XTS = two keys, scan for both.** Plugins that only output the FVEK and miss the tweak key produce a "valid-looking" key that decrypts to garbage. Always pull both halves of the schedule.
- **AES key schedules are self-validating.** The Rijndael recurrence (`W[i] = W[i-Nk] XOR f(W[i-1])`) means random 176-byte windows almost never satisfy it. That's the whole reason FVEK pool scanning works at all.
- **Validate candidates against sector 0 with the filesystem magic, not by trial-decrypting the whole volume.** A 512-byte test plus magic check is instant; full-volume decrypt for a wrong candidate wastes minutes per try.
- **Memory at-rest defeats BitLocker for a forensic adversary.** This is intentional behavior, not a bug — but it's why "lock the screen" / "use TPM-backed keys with secure suspend" is the actual mitigation.
