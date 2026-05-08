# Gunnar's Vacation Bis — Picture 2

- **Category:** OSINT
- **Challenge ID:** 12
- **URL:** `http://osint-gunnar-s-vacations.ctf.thcon.party/geozint2.html`
- **Status:** **UNSOLVED**

## TL;DR

- Equirectangular panorama (`8192×4096`, Intel IPP encoder); no GPS in EXIF.
- Visual scene is a French/European-style village square: church facade, trees, a parked white delivery van. Multiple weak OCR fragments (`pour moi`, `Labbé`, `Gruau`) suggest local signage but never aligned to a confirmed place.
- All researched candidates (church squares around the noisy OCR fragments) returned `Essaye encore !`. Not solved.

## 1. Recon

```bash
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/src/views/2.jpg
$ exiftool 2.jpg | grep -iE "GPS|Make|Encoder"
Image Size       : 8192x4096
Encoder Comment  : Intel IPP
# (no GPS)

$ tesseract 2.jpg stdout -l fra | head
... pour moi ...
... Labbé ...
... Gruau ...
```

Multiple perspective re-projections at the horizon; OCR fragments above. Vision-based reverse search (Bing/Yandex/Google Lens) was either blocked or returned no exact match.

## 2. Visual cues that *did* land

- Urban / village square, not coast.
- White delivery truck parked in scene.
- Church facade + plane trees lining a square.
- Likely France or French-speaking region (signage language matches).

## 3. Hypotheses that didn't validate

Every attempt to map `Labbé` / `Gruau` / `pour moi` to a French communal square failed: tested coordinate sweeps over candidate towns through `/geozint/2?lat=...&lon=...` returned `Essaye encore !`.

## 4. Methodology

- **Don't lock in a hypothesis from one OCR fragment.** Tesseract on warped panorama text returns false friends — confirm at least two independent fragments before searching.
- **Match the *colorway* of signage and street furniture, not just the text.** A green-on-white French rural sign is different from a blue-on-white Belgian one. Rule out countries before chasing town names.
- **Reverse image search rarely works on equirectangular.** Crop a regular gnomonic perspective first; reverse-image engines compare on visual similarity, and panorama warping wrecks similarity.
- **The challenge endpoint is generous on retries.** When you have geographic candidates, batch-test them programmatically rather than one at a time.

## 5. Status

Not solved during this session.
