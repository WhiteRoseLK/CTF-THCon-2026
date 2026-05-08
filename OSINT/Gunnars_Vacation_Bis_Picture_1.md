# Gunnar's Vacation Bis — Picture 1

- **Category:** OSINT
- **Challenge ID:** 11
- **URL:** `http://osint-gunnar-s-vacations.ctf.thcon.party/geozint1.html`
- **Status:** **UNSOLVED**

## TL;DR

- Equirectangular GoPro Max panorama at `src/views/1.jpg`. EXIF is stripped (no GPS).
- Tesseract OCR (`fra+eng`) over both raw and perspective-projected crops yields no readable signs.
- Solving requires actual visual inspection — language of signage, vegetation, road furniture, sky/sun direction. Not solved during the run; logged here for completeness and future passes.

## 1. What was checked

```bash
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/src/views/1.jpg
$ exiftool 1.jpg | grep -iE "GPS|Make|Model|Projection"
Make             : GoPro
Model            : GoPro Max
Projection Type  : equirectangular
# (no GPS fields)

$ tesseract 1.jpg stdout -l fra+eng     # → empty
```

Multiple perspective re-projections through `py360convert` / `equirectangular_crop`, then OCR — still empty.

## 2. Submission endpoint

```
GET /geozint/1?lat=<lat>&lon=<lon>
```

Returns the flag if within ~20 m. Free to retry; no rate limit observed.

## 3. Methodology — when EXIF is empty

- **Reproject the spherical image to gnomonic crops at the horizon.** Signs and license plates only become readable after un-warping.
- **Use the sun's azimuth + a known shooting time** (sometimes recoverable from CDN headers / `Last-Modified`) to bound latitude bands.
- **Scrub every sub-feature**: road paint patterns, signage colorway, traffic-light geometry, fire-hydrant style, license-plate format. France-specific cues (yellow plate eu-strip, blue motorway signage) collapse the search to a single country fast.
- **Cross-reference Panoramax / Mapillary** for the original GoPro Max trip if the photographer's account is known. The 2025 Gunnar collection was on Panoramax; the *Bis* set may share metadata.

## 4. Status

Not solved during this session. Recommended next steps: open the panorama in the on-page A-Frame viewer, transcribe any readable text manually, then OSM Overpass for matching landmark names.
