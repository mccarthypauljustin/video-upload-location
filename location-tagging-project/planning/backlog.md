# Backlog — Location Tagging

Engineering tickets grouped by epic. Each has acceptance criteria (AC). Estimates are T-shirt sizes (S/M/L). Priorities: P0 = V1 blocker, P1 = V1 nice-to-have, P2 = later.

---

## Epic A — Data model & API

### A1. Add `location` to video entity — P0 · M
Add structured location fields to the video schema and storage.
- **AC:** entity supports `label`, `latitude`, `longitude`, `source`, `country_code`, `city`, `region`; migration is backward-compatible; null location is valid.

### A2. Accept location on upload — P0 · M
Support the `location` object on `POST /videos/upload`.
- **AC:** valid object persists; omitting it succeeds; response echoes stored location.

### A3. Edit & remove location — P0 · S
Support `PATCH /videos/{id}` to update or clear location.
- **AC:** `location: {...}` updates; `location: null` removes; changes reflected on the public page.

### A4. Validation rules — P0 · S
- **AC:** latitude ∈ [-90, 90]; longitude ∈ [-180, 180]; `label` ≤ 200 chars; `country_code` is ISO 3166-1 alpha-2; invalid input → 400 with field-level errors.

### A5. Forward/reverse geocoding fallbacks — P1 · M
- **AC:** label-only input → backend resolves coordinates (limit=1, types=place,region,country); coordinates-only input → backend resolves label; on failure, store what's provided, leave the rest null.

## Epic B — Geocoding service

### B1. Vendor decision — P0 · S
Select and contract the geocoding vendor.
- **AC:** decision documented with pricing model, coverage, and rate limits; recommendation is Mapbox (unified autocomplete + geocoding).

### B2. Internal geocoding service — P0 · L
Thin internal wrapper over the vendor for autocomplete + reverse geocode.
- **AC:** single internal API; keys server-side only; graceful degradation on vendor error.

### B3. Reverse-geocode cache — P1 · M
- **AC:** results cached by lat/lng rounded to ~100 m; cache hit avoids a vendor call; TTL configurable.

## Epic C — Web upload UI

### C1. Shared location picker component — P0 · L
Reusable across web upload, embed, and back-office.
- **AC:** autocomplete with 300 ms debounce and keyboard nav; selectable result populates field; edit/remove supported; optional (omittable).

### C2. Device GPS + consent — P0 · M
- **AC:** consent copy shown before the browser prompt; consent logged with timestamp; `getCurrentPosition` uses `{ timeout: 8000, maximumAge: 60000, enableHighAccuracy: false }`; denied/timeout falls back to search.

### C3. IP soft hint — P1 · M
Server-side IP → city pre-fill.
- **AC:** hint rendered as dismissible, user-overridable; no raw precise coordinates stored from this path; uses MaxMind/CF header server-side.

## Epic D — Public rendering & SEO

### D1. Display location on video page — P0 · S
- **AC:** tagged location visible on the public page; hidden when absent.

### D2. Schema.org VideoObject — P0 · M
- **AC:** `contentLocation` (Place + GeoCoordinates) added to JSON-LD when location set; validates in Google Rich Results test.

### D3. Video sitemap enrichment — P1 · M
- **AC:** `<video:location>` included for tagged videos in sitemaps submitted to Search Console.

## Epic E — Mobile & programmatic

### E1. iOS upload (CoreLocation) — P1 · L
### E2. Android upload (FusedLocation) — P1 · L
### E3. API/SDK structured location — P1 · M
- **AC (E1–E3):** each surface can set a city-level location; shares the autocomplete API; API uploads pass structured JSON with forward-geocode fallback.

## Epic F — Privacy & compliance

### F1. Consent logging design — P0 · M · (needs Legal/Privacy)
- **AC:** consent record schema (user, timestamp, surface) agreed with Legal; GDPR/CCPA compliant; auditable.

### F2. Transparency & data minimization — P0 · S
- **AC:** UI states location is public; only city-level granularity stored for V1; rounded coordinates.

## Epic G — Analytics (later)

### G1. Location in creator analytics — P2 · M
- **AC:** creators can see location-attributed performance; gated on V2 prioritization.

---

## Suggested V1 cut (P0 only)

A1, A2, A3, A4 · B1, B2 · C1, C2 · D1, D2 · F1, F2

Everything else lands in V1.1 / Later per `roadmap.md`.
