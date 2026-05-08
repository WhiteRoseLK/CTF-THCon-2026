# Gunnar's Vacation Bis — Picture 6

- **Category:** OSINT
- **Challenge ID:** 16
- **URL:** `http://osint-gunnar-s-vacations.ctf.thcon.party/geozint6.html`
- **Status:** **UNSOLVED**

## TL;DR

- Equirectangular GoPro Max panorama, EXIF stripped, no OCR-readable signage.
- Solving requires a human to open the in-page A-Frame viewer and read landmarks/signs directly. Not solved during this run.

## 1. What was tried

```bash
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/src/views/6.jpg
$ exiftool 6.jpg | grep -iE "GPS|Make|Model"
Make             : GoPro
Model            : GoPro Max
# (no GPS)
$ tesseract 6.jpg stdout -l fra+eng         # empty
```

## 2. Submission endpoint

```
GET /geozint/6?lat=<lat>&lon=<lon>     →  flag on success
```

## 3. Methodology — recommended next pass

- Open the in-page A-Frame viewer at `geozint6.html` and rotate to read any visible signage / shopfront / vehicle plate.
- Run perspective reprojections (`py360convert`) at multiple yaw/pitch angles before OCR; raw equirectangular OCR loses too much.
- The *Bis* series is mostly France-centric (Pictures 3 and 4 are Drôme and Isère). Bias hypotheses toward Auvergne-Rhône-Alpes first.

## 4. Status

Not solved.
