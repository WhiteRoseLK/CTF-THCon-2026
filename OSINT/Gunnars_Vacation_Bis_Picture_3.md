# Gunnar's Vacation Bis — Picture 3

- **Category:** OSINT
- **Challenge ID:** 13
- **URL:** `http://osint-gunnar-s-vacations.ctf.thcon.party/geozint3.html`

```
THC{h16hw4y5_4r3_50_0h10}
```

## TL;DR

- 360° GoPro Max panorama (`8192×4096`, equirectangular). EXIF stripped — no GPS.
- The center crop at the horizon, OCR'd with `tesseract -l fra`, reveals **"aire de Bellevue 300m"**: a French motorway rest-area signpost.
- OSM Overpass returns two French motorway "Aire de Bellevue" rest areas (A7, A10). Vegetation and signage colorway match the **A7** in the Drôme.
- Submit `lat=44.80447, lon=4.85074` (Étoile-sur-Rhône) → flag.

## 1. Triage

```bash
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/src/views/3.jpg
$ exiftool 3.jpg | grep -iE "Make|Model|Image Size|Projection|GPS"
Make             : GoPro
Model            : GoPro Max
Image Size       : 8192x4096
Projection Type  : equirectangular
# (no GPS)
```

## 2. The OCR-readable strip

In an equirectangular panorama, signs at the horizon land in the middle vertical band and warp least near the front-facing center. Crop the front 180° at the horizon:

```python
from PIL import Image
img = Image.open("3.jpg")
W, H = img.size
img.crop((W // 4, H // 3, 3 * W // 4, 2 * H // 3)).save("3_crop.jpg")
```

```
$ tesseract 3_crop.jpg stdout -l fra | grep -i bellevue
... aire de Bellevue 300m ...
```

`Aire de` + `300m` + the blue-on-white road-sign palette = French motorway service-area panel.

## 3. Candidate hunt with Overpass

```
[out:json];
( node["name"~"Bellevue"]["highway"="rest_area"](42,0,50,8);
  way ["name"~"Bellevue"]["highway"="rest_area"](42,0,50,8); );
out center;
```

Returns two strong candidates:

| Aire | Motorway | Region | Approx. coordinates |
|---|---|---|---|
| Bellevue Nord | **A7 Autoroute du Soleil** | Drôme (26), Étoile-sur-Rhône | 44.804°N, 4.851°E |
| Bellevue Sud  | A10 L'Aquitaine | Loiret (45), Chaingy | 47.860°N, 1.741°E |

The panorama vegetation is dry/garrigue with east-facing pre-Alpine hills — Drôme, not Loiret. A7 is the bet.

## 4. Submit

```bash
$ curl -s "http://osint-gunnar-s-vacations.ctf.thcon.party/geozint/3?lat=44.80447550972472&lon=4.850737056947453"
THC{h16hw4y5_4r3_50_0h10}
```

The flag tells on the genre: *"highways are so OH IO"* — yes, OSINT on highways.

## 5. Methodology

- **Strip and OCR before reverse-image searching.** Panorama warp ruins similarity-based search; cropping a small horizon strip both helps human reading and lets `tesseract` work.
- **Land "type" before "location".** `aire de`, `300m`, blue panel → French motorway. Once the country and asset class are nailed, OSM Overpass is faster than Google.
- **Always run Overpass with a country bbox.** Without `(42,0,50,8)` the query returns Bellevues from anywhere on the planet — useless.
- **Vegetation + landform discriminate between same-named candidates.** When two A-vs-B motorway aires share a name, the surrounding terrain tells them apart in seconds.
