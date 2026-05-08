# THCon 2026 — Writeups

A collection of writeups for the [THCon 2026](https://ctf.thcon.party) CTF. Each writeup follows a uniform structure: TL;DR up top, numbered sections, peer-to-peer technical tone, and a Methodology paragraph at the end.

## Index

### Cryptography

| Challenge | Flag (or status) |
|---|---|
| [Break the Chain](Cryptography/Break_The_Chain.md) | `THC{4lL_Dr0Nz-R-g0N3}` |
| [Exponope](Cryptography/Exponope.md) | `THC{u_n3eD_@_bett3r_eXp0neNT}` |
| [Forged Goods](Cryptography/Forged_Goods.md) | `THC{tr0p1c4l_f4ct0r1z4t10n_1s_NP_h4rd_but_wh0_c4r3s}` |
| [Min Max](Cryptography/Min_Max.md) | `THC{fl0yd_w4rsh4ll_m33ts_crypt0gr4phy_1n_th3_tr0p1cs}` |
| [P4t4t0rz at the Library](Cryptography/P4t4t0rz_at_the_library.md) | `Knowledge is relative` |
| [Rhaaah SH-T (again)](Cryptography/Rhaaah_SH-T.md) | `THC{17858771354678072}` (math correct, judge rejected) |

### Forensics

| Challenge | Flag |
|---|---|
| [Breach at SST — 1](Forensic/Breach_at_SST_1.md) | `THCON{imsi-901701337133713}` |
| [Breach at SST — 2](Forensic/Breach_at_SST_2.md) | `THCON{h0p3_y0u_gr4bb3d_c0ff33_f0r_th3_n3xt_st3p}` |
| [Breach at SST — 3](Forensic/Breach_at_SST_3.md) | `THCON{sp3ctr4l_p34ks_d0nt_l13}` |
| [Don't Forget to Lock](Forensic/Dont_forget_to_lock.md) | `THCON{v1tl0ck3r_1n_MEm}` |

### Misc

| Challenge | Flag |
|---|---|
| [Welcome to the SoC](Misc/Welcome_to_the_SoC.md) | `THC{DMA-1s_n0t_5tr0ng_en0ugh?}` |

### OSINT

| Challenge | Flag (or status) |
|---|---|
| [Gunnar's Vacation Bis — Picture 1](OSINT/Gunnars_Vacation_Bis_Picture_1.md) | unsolved |
| [Gunnar's Vacation Bis — Picture 2](OSINT/Gunnars_Vacation_Bis_Picture_2.md) | unsolved |
| [Gunnar's Vacation Bis — Picture 3](OSINT/Gunnars_Vacation_Bis_Picture_3.md) | `THC{h16hw4y5_4r3_50_0h10}` |
| [Gunnar's Vacation Bis — Picture 4](OSINT/Gunnars_Vacation_Bis_Picture_4.md) | `THC{60774_61v3_cr3d17_70_7h3_516n}` |
| [Gunnar's Vacation Bis — Picture 5](OSINT/Gunnars_Vacation_Bis_Picture_5.md) | unsolved |
| [Gunnar's Vacation Bis — Picture 6](OSINT/Gunnars_Vacation_Bis_Picture_6.md) | unsolved |
| [Gunnar's Vacation Bis — Picture 7](OSINT/Gunnars_Vacation_Bis_Picture_7.md) | unsolved |
| [Gunnar's Vacation Bis — Picture 8](OSINT/Gunnars_Vacation_Bis_Picture_8.md) | unsolved |
| [No Cap Just Root — 2](OSINT/No_Cap_Just_Root_2.md) | `THC{king_p4t4t0rz_1337@sst.thcon}` |

### Pwn / Web (chained)

| Challenge | Flag |
|---|---|
| [No Cap Just Root — 1](Pwn/No_Cap_Just_Root_1.md) | `THC{sqli_and_awk_sudo_is_pure_brainrot}` |

### Reverse Engineering

| Challenge | Flag |
|---|---|
| [Silent Signer](Reverse/Silent_Signer.md) | `THC{int3_s3nt_u_h3r3_3bpf_t00k_1t_fr0m_th3r3!!!}` |

### Steganography

| Challenge | Flag |
|---|---|
| [Getting to the Bottom of Things](Steganography/Getting_to_the_Bottom_of_Things.md) | `THCon{TMTC_B1nwalk_D3t3ct3d}` |
| [M4terM4xima HINT (part 2)](Steganography/M4terM4xima_HINT_part2.md) | `THC{lui zero, ox123}` |
| [PNG is a Lie](Steganography/PNG_is_a_lie.md) | `THC{PNG3D}` |

### Web

| Challenge | Flag (or status) |
|---|---|
| [Incredibly Protected Notifications](Web/Incredibly_Protected_Notifications.md) | unsolved |
| [Panic in the Northern Quadrant — 1](Web/Panic_in_the_Northern_Quadrant_1.md) | `THC{s3cur3p455}` |
| [Panic in the Northern Quadrant — 2](Web/Panic_in_the_Northern_Quadrant_2.md) | `THC{r4c3d_2_t0p}` |
| [Panic in the Northern Quadrant — 3](Web/Panic_in_the_Northern_Quadrant_3.md) | `THC{Dynamics314!}` |
| [THCity: Authentication Collapse — 1](Web/THCity_Authentication_Collapse_1.md) | `THC{L34k_Ap4ch3_m0dul3_fR0m_F1l3_r3@d}` |
| [THCity: Authentication Collapse — 2](Web/THCity_Authentication_Collapse_2.md) | unsolved |
| [XSS iN tHe Web — 1](Web/XSS_iN_tHe_Web_1.md) | `THC{W1tH_eYe5_Wid3_0p3ns_WesTANd}` |
| [XSS iN tHe Web — 2](Web/XSS_iN_tHe_Web_2.md) | `THC{Th3_R1ght3ous_S1d3_0f_JinJa}` |

## Style

Each writeup follows roughly the same skeleton:

1. **Title + flag block** — name, category, and the flag (or `unsolved`).
2. **TL;DR** — bullet-point executive summary of the path.
3. **Numbered sections** — recon, vulnerability, primitive, exploit, validation. Sub-sections use decimal numbering.
4. **Methodology** — what generalizes from this challenge to others. Not a rehash of the steps; a "what would I do next time" paragraph.

Code uses fenced blocks with language tags. Tables map structured data (packet layouts, register offsets, key constants). ASCII art for byte/field diagrams when useful.
