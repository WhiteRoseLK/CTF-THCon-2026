# Breach at SST — 1

- **Category:** Forensics

```
THCON{imsi-901701337133713}
```

## TL;DR

- A raw `drive.img` ext4 image holds a 5G NGAP/GTP-U packet capture, the Home Network's ECC private key, and a `notes.txt` describing the architecture.
- Inside the GTP-U traffic, exactly one robot (`10.0.3.17`) hits a memory-vault HTTP API that nobody else touches — that's the suspect.
- Walk the NGAP control plane: GTP TEID `a1b20007` → `RAN_UE_NGAP_ID = 8` → the 8th Registration Request (frame 438) → the SUCI.
- Decrypt the SUCI with **ECIES Profile A** (`secp256r1` ECDH → ANSI X9.63 KDF → AES-128-CTR + HMAC-SHA-256/8) using the dumped HN private key. Recovered MSIN: `1337133713`. Full IMSI: PLMN `90170` + MSIN.

## 1. Disk image triage

```bash
$ mmls drive.img
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000206847   0000204800   Linux (0x83)
003:  000:001   0000206848   0000262143   0000055296   Linux (0x83)
```

Slot 002 (offset 2048, ext4) is the data partition. No mount available in the container — Sleuth Kit handles the rest:

```bash
$ fls -f ext4 -o 2048 drive.img -r
...
r/r 13:    sst_north_sector.pcap
r/r 17:    sst_hn_privkey.pem
r/r 21:    notes.txt

$ for i in 13 17 21; do icat -f ext4 -o 2048 drive.img $i > out_$i; done
```

`notes.txt` confirms the setup: Open5GS standalone core, SUPI format `9017XXXXXXXXXX`, HN running **ECIES Profile A** for SUCI concealment.

## 2. Spotting the misbehaving robot

The PCAP carries **NGAP** on the N2 control plane and **GTP-U** tunnels carrying each UE's IP traffic. Robots are supposed to be doing routine telemetry — anything else stands out.

```bash
$ tshark -r sst_north_sector.pcap -Y "gtp && http" \
    -T fields -e ip.src -e http.request.uri | sort -u | head
10.0.3.17  /api/v1/memory/store
10.0.3.17  /api/v1/memory/store
10.0.3.17  /api/v1/memory/get?key=vault_key
10.0.3.5   /api/v1/telemetry
10.0.3.8   /api/v1/telemetry
...
```

Only **`10.0.3.17`** touches the memory API, and it's exfiltrating notes and pulling `vault_key`. Self-identifies as `SST-T7-003` in User-Agent. Suspect confirmed.

## 3. Mapping IP → tunnel → NGAP UE ID

GTP-U tunnels are keyed by **TEID**. Find the TEID that carries `10.0.3.17`:

```bash
$ tshark -r sst_north_sector.pcap -Y "gtp && ip.src == 10.0.3.17" \
    -T fields -e gtp.teid | sort -u
a1b20007
```

The N2 `PDUSessionResourceSetupRequest` ties a TEID to a `RAN_UE_NGAP_ID` and a UE IP at session establishment:

```bash
$ tshark -r sst_north_sector.pcap \
    -Y "ngap && ngap.PDUSessionResourceSetupRequest_element" \
    -T fields -e frame.number -e ngap.RAN_UE_NGAP_ID \
              -e ngap.gTPTEID -e ngap.iPv4Address
2969  8  a1b20007  10.0.3.17
```

So the suspect is **UE #8** on the gNB.

## 4. Pulling the right SUCI

The SUCI is sent in the initial NAS Registration Request (`nas_5gs.mm.message_type == 0x41`). Each registration gets its own `RAN_UE_NGAP_ID`:

```bash
$ tshark -r sst_north_sector.pcap \
    -Y "ngap && nas_5gs.mm.message_type == 0x41" \
    -T fields -e frame.number -e ngap.RAN_UE_NGAP_ID -e nas_5gs.mm.suci
...
438  8  <SUCI bytes>
...
```

The frame-438 SUCI breaks down as:

```
SUCI type:           0  (SUCI)
PLMN:                90170
Routing Indicator:   0000
Protection Scheme:   Profile A (1)
HN Public Key ID:    1
Scheme Output:
  Ephemeral pub key (65 B, uncompressed):  04 || X(32) || Y(32)
  Ciphertext (5 B):                        MSIN_ENC
  MAC tag (8 B):                           TAG
```

## 5. ECIES Profile A in one page

3GPP TS 33.501 Annex C, **Profile A**:

| Step | Primitive |
|---|---|
| ECDH | `secp256r1` |
| KDF | ANSI X9.63 (SHA-256, counter `0x00000001`, `shared_info` = full 65-B ephemeral public key) |
| Encryption | AES-128-CTR with IV = 16 zero bytes |
| MAC | HMAC-SHA-256, **truncated to 8 bytes** |
| Key split | first 16 B → AES key, last 16 B → MAC key |

```python
import hashlib, hmac, struct
from cryptography.hazmat.primitives.serialization import load_pem_private_key
from cryptography.hazmat.primitives.asymmetric.ec import (
    ECDH, EllipticCurvePublicNumbers, SECP256R1
)
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

eph_pub_hex   = "04" + "<X hex>" + "<Y hex>"
ciphertext    = bytes.fromhex("<MSIN_ENC>")
mac_tag       = bytes.fromhex("<TAG>")

with open("sst_hn_privkey.pem", "rb") as f:
    hn = load_pem_private_key(f.read(), password=None, backend=default_backend())

eph_raw = bytes.fromhex(eph_pub_hex)
x = int.from_bytes(eph_raw[1:33], "big")
y = int.from_bytes(eph_raw[33:65], "big")
eph = EllipticCurvePublicNumbers(x, y, SECP256R1()).public_key(default_backend())

z         = hn.exchange(ECDH(), eph)              # 32 B shared secret
ks        = hashlib.sha256(z + struct.pack(">I", 1) + eph_raw).digest()
enc_k, mk = ks[:16], ks[16:]

assert hmac.new(mk, ciphertext, hashlib.sha256).digest()[:8] == mac_tag

iv     = b"\x00" * 16
msin   = Cipher(algorithms.AES(enc_k), modes.CTR(iv),
                backend=default_backend()).decryptor().update(ciphertext)

print("MSIN:", msin.decode("ascii"))
print("IMSI:", "90170" + msin.decode("ascii"))
```

```
MSIN: 1337133713
IMSI: 901701337133713
```

## 6. Flag

The format is `THCON{imsi-<IMSI>}` with the SUPI (PLMN || MSIN):

```
THCON{imsi-901701337133713}
```

## 7. Methodology

- **`mmls` + `fls -r` + `icat` is the no-mount triage trio.** When a CTF container ships without `losetup`, Sleuth Kit walks the inode tree and pulls files by inode — same as `dd` carving but content-aware.
- **Use the data plane to spot the suspect, the control plane to identify them.** GTP-U tells you what a UE *did*; NGAP tells you *who* the UE was. The SUCI lives only on N2.
- **TEID is the join key for IP ↔ NAS identity.** `PDUSessionResourceSetupRequest` is the canonical place where TEID, UE IP, and `RAN_UE_NGAP_ID` are all listed in the same message.
- **For ECIES Profile A, `shared_info` in the X9.63 KDF is the full uncompressed ephemeral public key (65 bytes), not just the X coordinate.** Mismatched `shared_info` is the #1 reason a hand-rolled SUCI decrypter fails MAC verification.
