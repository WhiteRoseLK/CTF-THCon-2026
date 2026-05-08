# Break the Chain

- **Category:** Cryptography
- **Challenge ID:** 4

```
THC{4lL_Dr0Nz-R-g0N3}
```

## TL;DR

- A C&C server emits encrypted **drone action packets** in a length-prefixed binary frame; a reference MiTM (`client.py`) just echoes them back unmodified.
- The crypto is a CBC-style chained scheme. The **action_type** byte sits at a known offset (byte 4 of each 16-byte block) and is the only field we need to mutate.
- Classic CBC bit-flip: XOR-ing the ciphertext at position *k* XORs the decrypted plaintext at position *k* — same offset, same value.
- XOR-ing every `action_type` byte with `0x03` turns the legitimate command into the **Autodelete** opcode. The server decrypts, sees self-destruct, and prints the flag.

## 1. Recon — what the wire actually looks like

Two artifacts are dropped with the challenge:

| File | Role |
|---|---|
| `client.py` | A reference MiTM skeleton: receives, prints, re-sends the payload **unchanged** |
| `SST-documentation.pdf` | The SNAFU Swarm Targeting protocol spec — packet format and crypto |

A bare `nc` confirms what the spec says about framing:

```bash
$ nc 4.178.152.74 9000 | xxd | head
```

The server pushes a single length-prefixed blob and waits for the client to echo it back. The framing is:

```
[ 4-byte big-endian length ][ payload of that many bytes ]
```

Both `send_prefixed` / `recv_prefixed` in `client.py` do exactly this — it's the same convention top-to-bottom.

## 2. The action packet, one block at a time

The PDF describes the **action payload** as a count followed by a sequence of 16-byte action blocks:

```
[ action_count : 4 bytes, big-endian uint32 ]
[ action_0     : 16 bytes ]
[ action_1     : 16 bytes ]
...
[ action_{N-1} : 16 bytes ]
```

Each 16-byte block is packed with `struct.pack(">HHBq3s", ...)`:

```
Offset  Size  Field
  0      2    id           (uint16, BE)
  2      2    robot_id     (uint16, BE)
  4      1    action_type  ← this is our target byte
  5      8    timestamp    (int64, BE)
 13      3    padding (zeros)
```

Only **one byte per block** matters: `action_type` at offset `4`.

## 3. The vulnerability — chained encryption with a known target offset

The PDF describes the cipher as a **CBC-like chaining**: each plaintext block is XORed with the previous ciphertext block (or an IV) before encryption. That gives us the textbook bit-flip primitive:

> Flipping bit *k* in ciphertext block *N* destroys the decrypted plaintext of block *N* (it becomes garbage), but flips bit *k* in the decrypted plaintext of block *N+1* exactly.

The challenge cooperates with us: the cipher is applied so that the **same-offset property** holds — XOR-ing ciphertext byte at offset *k* of a block flips plaintext byte at offset *k* of that block on decryption. The server decrypts, then *only* checks `action_type`. It never validates structure beyond that opcode, so corrupting other fields is free.

### What XOR mask?

We need the post-decryption byte to land on the **Autodelete** opcode. Either probe the service one byte at a time, or read the doc carefully — the answer is `0x03`. Why `0x03` works depends on what the server originally encrypted, but the relation is:

```
decrypted_action_type XOR 0x03 == AUTODELETE_OPCODE
```

So a single XOR mask, applied uniformly to every block's offset-4 byte, converts the entire payload into a kill-all order.

## 4. Reference client (do-nothing MiTM)

The provided `client.py` is the skeleton we hijack:

```python
def main():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((HOST, PORT))
        value_to_return = recv_prefixed(sock)

        # Print received blocks for inspection
        for i in range(4, len(value_to_return), 16):
            chunk = value_to_return[i:i+16]
            print(f"  {i:04x}  {chunk.hex(' ').upper()}")

        send_prefixed(sock, value_to_return)   # ← echoes payload UNCHANGED
        ...
```

Its own header says **"Status: Not yet working"** — yes, because it never tampers. We add exactly the tamper step it's missing.

## 5. Exploit

```python
#!/usr/bin/env python3
# break_the_chain.py — CBC bit-flip on the SNAFU drone control protocol
import socket, struct

HOST, PORT = "4.178.152.74", 9000

def recvn(s, n):
    buf = b""
    while len(buf) < n:
        chunk = s.recv(n - len(buf))
        if not chunk:
            raise ConnectionError("closed")
        buf += chunk
    return buf

def recv_prefixed(s):
    length = struct.unpack(">I", recvn(s, 4))[0]
    return recvn(s, length)

def send_prefixed(s, data):
    s.sendall(struct.pack(">I", len(data)) + data)

def main():
    with socket.socket() as s:
        s.connect((HOST, PORT))

        # 1. Receive the encrypted blob
        payload = bytearray(recv_prefixed(s))

        # 2. Parse the leading action count
        action_count = struct.unpack_from(">I", payload, 0)[0]

        # 3. Flip the action_type byte in every block
        #    Layout: [count:4][block0:16][block1:16]...
        #    action_type sits at offset 4 inside each block
        for i in range(action_count):
            payload[4 + 16 * i + 4] ^= 0x03

        # 4. Echo the tampered ciphertext back
        send_prefixed(s, bytes(payload))

        # 5. Read the verdict
        out = b""
        while True:
            chunk = s.recv(65536)
            if not chunk:
                break
            out += chunk
        print(out.decode(errors="replace"))

if __name__ == "__main__":
    main()
```

Running it:

```
$ python3 break_the_chain.py
All drones have been successfully deleted!
THC{4lL_Dr0Nz-R-g0N3}
```

## 6. Methodology

- **When a challenge ships with both a reference client *and* a protocol PDF, read the PDF first.** The PDF tells you exactly which fields are integrity-checked vs. ignored, and where they sit in the block.
- **CBC chaining + a known target offset = bit-flip.** Don't let the surrounding sci-fi narrative distract from the primitive — the moment you see "previous ciphertext XOR plaintext", the attack is fixed.
- **Probe the XOR mask if the doc is ambiguous.** With only 256 possible single-byte values, a brute-force probe over a fresh connection per attempt converges in seconds.
- **Tamper uniformly when the same field is repeated.** Every block has identical structure, so the same XOR works for all — no per-block bookkeeping needed.
