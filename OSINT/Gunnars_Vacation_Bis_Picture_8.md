# Gunnar's Vacation Bis — Picture 8

- **Category:** OSINT
- **Challenge ID:** 18
- **URL:** `http://osint-gunnar-s-vacations.ctf.thcon.party/geozint8.html`
- **Status:** **UNSOLVED**

## TL;DR

- GoPro Max panorama, EXIF stripped, OCR returns nothing usable.
- Manual visual inspection required. Not solved.

## 1. What was tried

```bash
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/src/views/8.jpg
$ exiftool 8.jpg | grep -iE "GPS|Make|Model"
Make             : GoPro
Model            : GoPro Max
# (no GPS)
$ tesseract 8.jpg stdout -l fra+eng         # empty
```

## 2. Submission endpoint

```
GET /geozint/8?lat=<lat>&lon=<lon>     →  flag on success
```

## 3. Methodology

Same as Pictures 6/7. Open the A-Frame viewer, look for distinctive built features (church, statue, signpost, shopfront), then OSM Overpass on the resulting type.

## 4. Status

Not solved.
