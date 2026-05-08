# Breach at SST — 2

- **Category:** Forensics
- **Challenge ID:** 7
- **Points:** 487

```
THCON{h0p3_y0u_gr4bb3d_c0ff33_f0r_th3_n3xt_st3p}
```

## TL;DR

- The same `drive.img` from part 1 has a third partition flagged by Sleuth Kit as **LUKS2**. The flag lives inside.
- The unlock passphrase isn't on disk — `auth.log` and an IRC log point at *"I'll send it over the 5G channel, not here."*
- Inside the same NGAP/GTP-U capture from part 1, an HTTP `POST /api/v1/memory/store` carries `{"key":"vault_key", "value":"d1m1tr1_0w3s_m3_c0ff33"}`. That's the LUKS passphrase.
- Carve partition 3, unlock with `cryptsetup`, mount the inner ext4, read `flag.txt`.

## 1. Filesystem layout

```bash
$ mmls drive.img
...
002:  000:000  0000002048   0000206847   0000204800   Linux (0x83)
003:  000:001  0000206848   0000262143   0000055296   Encryption detected (LUKS)
```

Partition 3 (offset 206848, length 55296 sectors of 512 B) is the LUKS volume.

## 2. Why the password isn't on disk

Partition 2 (regular Linux root) has the operational story:

```bash
$ icat -f ext4 -o 2048 drive.img <inode of auth.log> | grep cryptsetup
... cryptsetup luksOpen /dev/sda3 vault
```

Repeated unlocks of `/dev/sda3` confirm the LUKS volume is the daily-driver vault. The shell history shows the same.

The smoking gun is in `home/crypt/.local/share/weechat/logs/irc.xss-mesh.#ops.weechatlog`:

```
<dimitri> can you re-send me the vault pw, I lost it
<viktor>  I'll send it over the 5G channel, not here
<viktor>  using the memory API on your handler
```

So the passphrase travels in-band over the GTP-U user-plane traffic, written into the same `/api/v1/memory` endpoint that Viktor's robot was already abusing in part 1.

## 3. Recovering the passphrase from the PCAP

The capture is the same `sst_north_sector.pcap` we extracted in part 1. Filter HTTP bodies for `vault_key`:

```bash
$ tshark -r sst_north_sector.pcap \
    -Y 'http.request.uri contains "/api/v1/memory" || http.file_data contains "vault_key"' \
    -T fields -e frame.number -e http.request.method \
              -e http.request.uri -e http.file_data
... POST  /api/v1/memory/store  {"key":"vault_key","value":"d1m1tr1_0w3s_m3_c0ff33"}
... GET   /api/v1/memory/get?key=vault_key
... <response body: same value>
```

Passphrase:

```
d1m1tr1_0w3s_m3_c0ff33
```

(The Russian-flavored joke is also a tell — the IRC log mentions Dimitri owing Viktor coffee.)

## 4. Carving and unlocking partition 3

Carve the partition out of the image (cryptsetup wants its own file/loop):

```bash
$ dd if=drive.img of=p3.img bs=512 skip=206848 count=55296 status=progress
$ cryptsetup luksDump p3.img | head
LUKS2 header...
$ printf '%s' 'd1m1tr1_0w3s_m3_c0ff33' \
    | cryptsetup open --test-passphrase p3.img -
# (no output → passphrase is correct)
```

In a privileged environment, just open and mount:

```bash
$ printf '%s' 'd1m1tr1_0w3s_m3_c0ff33' | cryptsetup luksOpen p3.img vault --key-file=-
$ mount -o ro /dev/mapper/vault /mnt/vault
$ cat /mnt/vault/flag.txt
THCON{h0p3_y0u_gr4bb3d_c0ff33_f0r_th3_n3xt_st3p}
```

If the container has no device-mapper, decrypt offline by piping the unlocked block device (or use a userspace LUKS reader like `dissect.fve` to get a decrypted ext4 image, then `fls`/`icat`).

## 5. Inside the vault

```bash
$ ls /mnt/vault/
flag.txt        intercept.wav   sigdb
README_DIMITRI.txt  vault_note.txt
```

Other files (`intercept.wav`, `sigdb`, `vault_note.txt`) belong to part 3 — see the next writeup.

## 6. Methodology

- **`mmls` flags LUKS partitions explicitly.** The "Encryption detected (LUKS)" hint is loud — go look for the password before fighting `cryptsetup`.
- **When the password isn't on disk, the disk usually points at where it is.** `auth.log` + IRC + bash history is a triangle that almost always names the next channel.
- **Re-read the same PCAP through a different lens for the next part.** Part 1 used the capture to identify a robot; part 2 reuses it to extract data the robot exfiltrated. Don't redownload — refilter.
- **Carve before unlock.** A separate `dd`-extracted partition file is easier to operate on than a full image with offsets, and doesn't risk touching the wrong partition.
- **`cryptsetup --test-passphrase` is a safe oracle.** It validates without needing device-mapper — handy in unprivileged containers when you only need to confirm a guess.
