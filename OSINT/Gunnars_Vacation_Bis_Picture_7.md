# Gunnar's Vacation Bis — Picture 7

- **Category:** OSINT
- **Challenge ID:** 17
- **URL:** `http://osint-gunnar-s-vacations.ctf.thcon.party/geozint7.html`
- **Status:** **UNSOLVED**

## TL;DR

- GoPro Max panorama, EXIF stripped, OCR finds no readable text.
- Same recipe as Picture 6: requires interactive visual inspection. Not solved.

## 1. What was tried

```bash
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/src/views/7.jpg
$ exiftool 7.jpg | grep -iE "GPS|Make|Model"
Make             : GoPro
Model            : GoPro Max
# (no GPS)
$ tesseract 7.jpg stdout -l fra+eng         # empty
```

## 2. Submission endpoint

```
GET /geozint/7?lat=<lat>&lon=<lon>     →  flag on success
```

## 3. Methodology

See [Picture 6](Gunnars_Vacation_Bis_Picture_6.md). Same playbook: open the A-Frame viewer, perspective reproject, bias toward France / Auvergne-Rhône-Alpes.

## 4. Status

Not solved.
