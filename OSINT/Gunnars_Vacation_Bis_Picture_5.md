# Gunnar's Vacation Bis — Picture 5

- **Category:** OSINT
- **URL:** `http://osint-gunnar-s-vacations.ctf.thcon.party/geozint5.html`
- **Status:** **UNSOLVED**

## TL;DR

- Panorama at `src/views/5.jpg`, plus `painting.png` and `logo.png` referenced in the page. EXIF stripped.
- The 2025 (non-*Bis*) edition's flag was `THC{M3h_1_Gu355_u_f1nd_stuff_3v3ntu411y}`; submitting it against the *Bis* endpoint failed. The image is a different shoot for 2026.
- No published writeup, no exact reverse-image hit, no OCR-readable text. Not solved.

## 1. What was tried

```bash
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/src/views/5.jpg
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/painting.png
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/logo.png

$ exiftool 5.jpg | grep -i GPS    # nothing
$ tesseract 5.jpg stdout -l fra+eng | head   # nothing useful
```

Cross-checked the 2025 Gunnar collection on Panoramax: same site structure, *different* image. Coordinate sweeps near the old 2025 location and across the old collection were rejected by the backend.

## 2. Likely path (untested)

- The auxiliary `painting.png` / `logo.png` suggest the location is identified via a **mural / shopfront / institutional logo** in the panorama. Reverse-search those two assets on TinEye / Yandex specifically (not the panorama).
- Once the mural / logo is identified, OSM Overpass on `man_made=mural` or business name + region narrows the location.

## 3. Methodology

- **Auxiliary assets next to a `geozintN.html` page are not random.** A `painting.png` shipped alongside the panorama is the intended landmark — the puzzle is "find this painting in the panorama, then find where the painting actually is."
- **Reverse-search the auxiliary asset first.** Static logo/painting images survive reverse-image search; the panorama doesn't.
- **Don't reuse last year's flag.** Even when the page structure is reused, the underlying photo is usually re-shot for the new edition.

## 4. Status

Not solved.
