# Roadmap — Location Tagging

A Now / Next / Later view. V1 targets city-level tagging with the location publicly visible and editable after upload.

---

## Now (V1 — ship the core loop)

Goal: a creator can add, edit, and remove a city-level location on web upload, and it renders publicly with SEO markup.

- **Data model + API** — add `location` object to the video entity; support it on `POST /videos/upload` and `PATCH /videos/{id}` with validation.
- **Vendor decision + integration** — select geocoding vendor (Mapbox recommended), wire autocomplete + reverse geocoding behind a thin internal service.
- **Web upload UI** — shared location picker component: autocomplete search, GPS button with consent, IP soft hint.
- **Public rendering + SEO** — display location on the video page; inject `contentLocation` into `VideoObject` JSON-LD.
- **Privacy baseline** — consent logging for GPS, transparency copy, edit/remove.

## Next (V1.1 — reach + parity)

Goal: extend beyond web and enrich indexing.

- **iOS + Android upload** — native GPS via CoreLocation / FusedLocation, shared autocomplete API.
- **API/SDK uploads** — structured JSON + forward-geocoding fallback for label-only input.
- **Video sitemap enrichment** — include `<video:location>` for tagged videos.
- **Reverse-geocode caching** — cache by lat/lng rounded to ~100 m.
- **Back-office editing** — location editable from the management/back-office tools.

## Later (V2 — leverage the data)

Goal: turn stored geo-metadata into product value.

- **Location-powered search & recommendations** — rank/surface by place.
- **Content filtering hooks** — use structured coordinates to inform (separate) geo-policy work.
- **Creator analytics** — surface location performance to creators.
- **Precision options** — evaluate finer-than-city granularity where use cases justify it.

---

## Sequencing notes

- Vendor decision blocks most of "Now" — resolve first.
- Consent-logging design needs Legal/Privacy sign-off before GPS ships (blocks the web UI's GPS path, not autocomplete).
- SEO output (`VideoObject` + sitemap) is cheap and high-impact; don't let it slip to "Later."
