# Gunnar's Vacation Bis — Picture 4

- **Category:** OSINT
- **Challenge ID:** 14
- **URL:** `http://osint-gunnar-s-vacations.ctf.thcon.party/geozint4.html`

```
THC{60774_61v3_cr3d17_70_7h3_516n}
```

## TL;DR

- Unlike the other Gunnar panoramas, this image's EXIF was **not stripped**: it carries `GPS Latitude 45°20'49.46"N`, `GPS Longitude 5°30'39.73"E`.
- Convert DMS → decimal: `lat = 45.34707, lon = 5.51104`. That's **Route des Trois Fontaines, Rives, Isère**, in the Chartreuse foothills.
- Submit; first try wins. The flag itself winks at it: *"gotta give credit to the sign."*

## 1. EXIF immediately wins

```bash
$ curl -sO http://osint-gunnar-s-vacations.ctf.thcon.party/src/views/4.jpg
$ exiftool 4.jpg | grep -iE "GPS|Camera|Make|Model"
Make                : GoPro
Model               : GoPro Max
GPS Latitude        : 45 deg 20' 49.46" N
GPS Longitude       : 5 deg 30' 39.73" E
GPS Altitude        : 364 m Above Sea Level
GPS Img Direction   : 78.44
```

DMS → decimal:

```
lat = 45 + 20/60 + 49.46/3600 = 45.34707222
lon =  5 + 30/60 + 39.73/3600 =  5.51103611
```

Or one-shot via exiftool:

```bash
$ exiftool -GPSLatitude# -GPSLongitude# 4.jpg
GPS Latitude    : 45.3470722222222
GPS Longitude   : 5.51103611111111
```

## 2. Submit

```bash
$ curl -s "http://osint-gunnar-s-vacations.ctf.thcon.party/geozint/4?lat=45.34707222&lon=5.51103611"
THC{60774_61v3_cr3d17_70_7h3_516n}
```

`60774_61v3_cr3d17_70_7h3_516n` ≈ *"gotta give credit to the sign"*: the EXIF *was* the sign.

## 3. Methodology

- **`exiftool` is the first move on every geolocation challenge.** It costs one second and either solves it outright or rules out the EXIF path.
- **DMS conversion is a one-liner.** `deg + min/60 + sec/3600`. Or use `exiftool -GPSLatitude#` to get the decimal directly.
- **Stripped EXIF on every other panorama in a series ≠ stripped on this one.** Authors miss artifacts. Always check; never assume.
- **The flag often acknowledges the trick.** When you see `60774_61v3_cr3d17_70_7h3_516n`, the author is admitting EXIF was the giveaway.
