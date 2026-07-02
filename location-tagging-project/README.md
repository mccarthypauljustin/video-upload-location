# Location Tagging — Video Upload

Adds optional, structured location tagging to Dailymotion's video upload flow. Creators can attach a place to a video via device GPS, autocomplete search, or an IP-based hint. Location is publicly visible, editable after upload, and stored as structured geo-metadata to power search, recommendations, and SEO.

**Owner:** Paul McCarthy · **Status:** Draft / Prototype · **Target:** V1 (city-level)

---

## Project structure

```
location-tagging-project/
├── README.md                     ← you are here
├── docs/
│   ├── location-tagging-spec.md  ← full technical specification (v1.0)
│   └── technical-overview.md     ← how the geolocation code works
├── prototype/
│   └── video-upload-location-prototype.html  ← working UI (open in a browser)
└── planning/
    ├── roadmap.md                ← Now / Next / Later
    └── backlog.md                ← engineering tickets w/ acceptance criteria
```

## Quick start

Open `prototype/video-upload-location-prototype.html` in a browser. It mirrors the real Dailymotion "Video details" page with the location field added, and runs live against free geocoders (Photon, Nominatim, ipapi.co) as stand-ins for the production vendor.

> Note: GPS and the IP hint require an **HTTPS or localhost** origin — browsers block geolocation on `file://`. Autocomplete search works anywhere.

## Scope at a glance

**In:** optional location at upload, autocomplete + GPS + IP hint, structured storage (label + coords + country/city/region), post-upload edit/remove, Schema.org `VideoObject` output.

**Out (V1):** mandatory tagging, real-time tracking, geo-blocking, address-level precision.

## The three location inputs

| Source | Mechanism | Permission |
|---|---|---|
| `device_gps` | W3C Geolocation → reverse geocode | Explicit consent (logged) |
| `user_search` | Autocomplete typeahead | None |
| `device_ip` | Server-side IP lookup (soft hint) | None |

## Key decisions (see spec §12)

- Publicly visible on the video page — **yes**
- Granularity — **city-level for V1**
- Editable after upload — **yes** (UI + API)
- Geocoding vendor — **Mapbox recommended** (unified autocomplete + geocoding)

## Open items needing input

- Consent logging design → Legal/Privacy
- Whether IP hints ship on all surfaces
- Surfacing location in creator analytics

See `planning/backlog.md` for the full ticket breakdown.
