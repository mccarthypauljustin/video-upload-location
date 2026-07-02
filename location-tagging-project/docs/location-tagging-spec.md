# Technical Specification: Location Data in Video Upload Flow

**Version:** 1.0  
**Author:** Paul McCarthy  
**Date:** June 2026  
**Status:** Draft

---

## Table of Contents

1. [Overview](#1-overview)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [User Experience Flow](#3-user-experience-flow)
4. [APIs](#4-apis)
5. [Upload Surfaces](#5-upload-surfaces)
6. [Data Model](#6-data-model)
7. [API Schema & Validation](#7-api-schema--validation)
8. [Post-Upload Editing](#8-post-upload-editing)
9. [SEO Benefits](#9-seo-benefits)
10. [Privacy & Compliance](#10-privacy--compliance)
11. [Implementation Recommendations](#11-implementation-recommendations)
12. [Open Questions & Decisions](#12-open-questions--decisions)

---

## 1. Overview

This specification describes the introduction of **location tagging** into Dailymotion's video upload flow. Users will be able to associate geographic metadata with their uploaded video via two complementary mechanisms:

- **Manual search with autocomplete** — user types a location and selects from suggestions
- **Device geolocation with pre-population** — device GPS or IP is used to suggest a location, with explicit user consent where required

Location data will be:
- **Publicly visible** on the video page
- **Editable after upload** via the video management interface and API
- **Used for future content filtering, search ranking, and recommendations**
- **Structured** (coordinates + human-readable label) to support SEO and downstream systems

---

## 2. Goals & Non-Goals

### Goals
- Allow users to optionally attach a location to a video at upload time
- Provide a smooth, low-friction UX via autocomplete search
- Support automatic location detection with user permission (web and mobile)
- Support IP-based soft hints server-side (no permission required)
- Accept structured location metadata via the upload API
- Store structured geo-metadata (coordinates + human-readable label) on the video entity
- Make location data editable and removable after upload
- Expose location in Schema.org VideoObject structured data for SEO

### Non-Goals
- Mandatory location tagging (it remains optional)
- Real-time location tracking beyond the upload moment
- Location-based content restrictions or geo-blocking (separate concern, informed by this data)
- Address-level precision (city-level is sufficient for V1)

---

## 3. User Experience Flow

```
[Upload Screen]
     |
     ├── "Add a location" field (optional)
     |        |
     |        ├── [📍 Use my current location] button
     |        |        |
     |        |        └── Request browser/device permission
     |        |                 ├── Granted → Reverse geocode → Pre-fill field
     |        |                 └── Denied  → Show manual search fallback
     |        |
     |        ├── [Search field] → User types → Autocomplete suggestions appear
     |        |                                   └── User selects → Field populated
     |        |
     |        └── IP-based soft hint → Pre-fill field with city (user can override)
     |
     └── Continue upload → Location metadata saved with video
```

---

## 4. APIs

### 4.1 Device Geolocation (Auto-Detection)

**API:** W3C Geolocation API (browser-native, no key required)

```js
navigator.geolocation.getCurrentPosition(
  (position) => {
    const { latitude, longitude } = position.coords;
    // Pass to reverse geocoding
  },
  (error) => {
    // Fallback to manual search
  },
  { timeout: 8000, maximumAge: 60000, enableHighAccuracy: false }
);
```

- Works on all modern web browsers
- Requires explicit user permission (browser-native prompt)
- HTTPS is required — blocked on HTTP origins
- `enableHighAccuracy: false` is recommended — city-level is sufficient and avoids battery drain
- On native mobile apps, use platform equivalents:
  - **iOS:** `CoreLocation` framework (`CLLocationManager`)
  - **Android:** `FusedLocationProviderClient` (Google Play Services)

> **Privacy note:** The permission prompt must be preceded by clear in-UI copy explaining why location is being requested. Consent must be logged.

---

### 4.2 Reverse Geocoding (Coordinates to Human-Readable Location)

Once coordinates are obtained, they must be converted to a readable label (e.g. "Paris, France").

| Provider | API Endpoint | Notes |
|---|---|---|
| **Google Maps Geocoding API** | `maps.googleapis.com/maps/api/geocode/json?latlng=...` | Best accuracy; paid beyond free tier |
| **Mapbox Geocoding API** | `api.mapbox.com/geocoding/v5/mapbox.places/{lon},{lat}.json` | Competitive pricing; good global coverage |
| **OpenStreetMap Nominatim** | `nominatim.openstreetmap.org/reverse?lat=...&lon=...` | Free; usage policy limits — dev/testing only |
| **HERE Geocoding & Search API** | `revgeocode.search.hereapi.com/v1/revgeocode` | Strong for precise address-level data |

**Recommendation:** Use **Mapbox** or **Google** for production scale. Use **Nominatim** for development and testing only.

---

### 4.3 Location Search with Autocomplete

When the user types a location manually, suggestions should appear in real-time with a **300ms debounce** to limit API calls.

| Provider | API Endpoint | Notes |
|---|---|---|
| **Google Places Autocomplete API** | `maps.googleapis.com/maps/api/place/autocomplete/json?input=...` | Industry-standard; paid per request |
| **Mapbox Search / Geocoding API** | `api.mapbox.com/geocoding/v5/mapbox.places/{query}.json?autocomplete=true` | Session-based pricing; unified with reverse geocoding |
| **HERE Autocomplete API** | `autocomplete.search.hereapi.com/v1/autocomplete` | Strong city/region resolution |
| **Photon (OpenStreetMap-based)** | `photon.komoot.io/api/?q=...` | Free, open-source; self-hostable |

**Recommendation:** Use **Mapbox** for a unified solution covering both autocomplete and reverse geocoding, reducing vendor sprawl.

---

### 4.4 IP-Based Soft Hint (No Permission Required)

On page load, a coarse city-level location can be inferred from the user's IP address server-side and used to pre-populate the location field. The user can confirm or override.

| Provider | Notes |
|---|---|
| **MaxMind GeoIP2** | Industry standard; accurate to city level; GDPR-safe server-side |
| **ipapi.co** | Simple REST API; free tier available |
| **Cloudflare `CF-IPCountry` header** | Zero-effort if Dailymotion uses Cloudflare; country-level only |

> This approach requires no user permission and does not expose raw coordinates, making it GDPR-compliant as a UX hint.

---

### 4.5 Forward Geocoding (Label to Coordinates)

When an API caller or user provides only a text label (e.g. "Berlin"), the backend should attempt to resolve it to coordinates:

```
GET https://api.mapbox.com/geocoding/v5/mapbox.places/Berlin.json
    ?access_token=...
    &types=place,region,country
    &limit=1
```

The resolved coordinates are stored alongside the label. If resolution fails, the label is stored as-is and coordinates remain null.

---

## 5. Upload Surfaces

### 5.1 Web Browser Uploads

| Input Method | Available | Notes |
|---|---|---|
| Device GPS (W3C Geolocation API) | Yes (with permission) | City-level accuracy on most laptops/desktops |
| IP-based soft hint | Yes (silent, server-side) | Pre-fills field; user can override |
| Manual search + autocomplete | Yes | Primary fallback |

### 5.2 iOS App Uploads

| Input Method | Available | Notes |
|---|---|---|
| Device GPS (CoreLocation) | Yes (with permission) | Higher accuracy than web |
| IP-based soft hint | Optional | Can be added server-side |
| Manual search + autocomplete | Yes | Same shared API |

### 5.3 Android App Uploads

| Input Method | Available | Notes |
|---|---|---|
| Device GPS (FusedLocationProviderClient) | Yes (with permission) | High accuracy |
| IP-based soft hint | Optional | Can be added server-side |
| Manual search + autocomplete | Yes | Same shared API |

### 5.4 API Uploads (Programmatic)

Device geolocation does not apply. Location must be passed explicitly as structured metadata.

| Input Method | Available | Notes |
|---|---|---|
| Structured JSON in request body | Yes | Primary method — see Section 7 |
| Label-only (forward geocoding fallback) | Yes | Backend resolves coordinates from label |
| Auto-detection | No | Not applicable to programmatic uploads |

**API caller scenarios:**

| Scenario | Who provides location | How |
|---|---|---|
| CMS / third-party tool | The tool itself | Caller passes known metadata |
| Mobile SDK upload | Client app | App captures device GPS before upload, passes coordinates |
| Automated pipeline (e.g. broadcast ingest) | Pipeline operator | Location passed in ingest manifest |
| No location provided | Nobody | Field left null; no geocoding attempted |

### 5.5 Surface Summary

| Upload Surface | Auto-detect GPS | IP Hint | Manual Search | API Metadata Field |
|---|---|---|---|---|
| Web browser | Yes (W3C, with permission) | Yes (server-side, silent) | Yes | N/A |
| iOS app | Yes (CoreLocation, with permission) | Optional | Yes | N/A |
| Android app | Yes (FusedLocation, with permission) | Optional | Yes | N/A |
| API (programmatic) | No | No | No | Yes (structured JSON) |
| Mobile SDK upload | App captures GPS | No | Optional | Passed by SDK caller |

---

## 6. Data Model

The following fields should be added to the video metadata schema:

```json
{
  "location": {
    "label": "Paris, France",
    "latitude": 48.8566,
    "longitude": 2.3522,
    "source": "user_search | device_gps | device_ip",
    "country_code": "FR",
    "city": "Paris",
    "region": "Île-de-France"
  }
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `label` | String (max 200) | Yes (if location set) | Human-readable display string |
| `latitude` | Float (-90 to 90) | No | WGS84 coordinate |
| `longitude` | Float (-180 to 180) | No | WGS84 coordinate |
| `source` | Enum | No | Audit trail: how location was set |
| `country_code` | String (ISO 3166-1 alpha-2) | No | e.g. "FR" |
| `city` | String | No | e.g. "Paris" |
| `region` | String | No | e.g. "Île-de-France" |

---

## 7. API Schema & Validation

### 7.1 Upload with Location

```json
POST /videos/upload
{
  "title": "My Video",
  "description": "...",
  "location": {
    "label": "Tokyo, Japan",
    "latitude": 35.6762,
    "longitude": 139.6503,
    "country_code": "JP",
    "city": "Tokyo"
  }
}
```

### 7.2 Validation Rules

- `latitude` must be a float between -90 and 90
- `longitude` must be a float between -180 and 180
- `label` must be a string, max 200 characters
- `country_code` must conform to ISO 3166-1 alpha-2 if provided
- If only `label` is provided, the backend attempts forward geocoding to resolve coordinates
- If only `latitude`/`longitude` are provided, the backend reverse geocodes to generate a `label`
- Location object is fully optional; omitting it is valid

---

## 8. Post-Upload Editing

Location should be editable and removable after upload via both the video management UI and the API.

### 8.1 Update Location

```json
PATCH /videos/{video_id}
{
  "location": {
    "label": "Lyon, France",
    "latitude": 45.7640,
    "longitude": 4.8357,
    "country_code": "FR",
    "city": "Lyon"
  }
}
```

### 8.2 Remove Location

```json
PATCH /videos/{video_id}
{
  "location": null
}
```

---

## 9. SEO Benefits

### 9.1 Local Search Visibility

Search engines prioritise locally relevant results for geo-intent queries (e.g. "best street food Paris" or "Tokyo travel vlog"). Location-tagged videos are more likely to:

- Appear in local search results and the Google "Videos" tab for geo-specific queries
- Surface in Google Discover for users in or interested in that location
- Rank for long-tail local video queries where YouTube currently dominates

### 9.2 Structured Data and Rich Results

Location data enables proper use of **Schema.org VideoObject markup**, making videos eligible for enhanced SERP appearances.

```json
{
  "@context": "https://schema.org",
  "@type": "VideoObject",
  "name": "My Video Title",
  "contentLocation": {
    "@type": "Place",
    "name": "Paris, France",
    "geo": {
      "@type": "GeoCoordinates",
      "latitude": 48.8566,
      "longitude": 2.3522
    }
  }
}
```

Rich results can include visual location labels in SERPs, improving click-through rates (CTR).

### 9.3 Improved Crawlability and Indexing Signals

Location metadata adds semantic context that search engine crawlers can parse:

- **Topical relevance scoring** — a video tagged "Kyoto, Japan" becomes more semantically associated with Japan travel, culture, and tourism queries
- **Entity association** — Google's Knowledge Graph links locations to broader entities (cities, landmarks, events)
- **Freshness signals** — news or event videos tagged to a location can rank higher for breaking local queries

### 9.4 Video Sitemaps

Location can be included in Video Sitemaps submitted to Google Search Console:

```xml
<video:video>
  <video:title>Street Market in Marrakech</video:title>
  <video:location>Marrakech, Morocco</video:location>
  ...
</video:video>
```

At Dailymotion's scale, this ensures location-tagged content is indexed faster and with a trusted, canonical source of geo-metadata.

### 9.5 Voice Search and Conversational Queries

Voice search queries are heavily local in nature (e.g. "show me videos filmed in Barcelona"). Structured location metadata makes content more indexable for natural language and voice-driven queries, which continue to grow on mobile and smart devices.

### 9.6 Behavioural Engagement Signals (Indirect)

Strong location-powered internal search and recommendations improve dwell time, pages per session, and bounce rate — all behavioural signals that positively influence organic rankings over time.

### 9.7 Competitive Differentiation

YouTube has offered location tagging for years and it contributes to their dominance in local video search. Adding structured location data to Dailymotion's catalogue helps close the indexing gap and positions Dailymotion as a more semantically rich alternative for search engines to index.

### 9.8 SEO Impact Summary

| SEO Benefit | Impact Level | Notes |
|---|---|---|
| Local search visibility | High | Direct ranking factor for geo-intent queries |
| Schema.org rich results | High | Improves SERP appearance and CTR |
| Semantic content relevance | Medium-High | Topical clustering and entity association |
| Video sitemap enrichment | Medium | Speeds up indexing at scale |
| Voice search eligibility | Medium | Growing channel, especially mobile |
| Behavioural engagement signals | Medium | Indirect but cumulative ranking benefit |
| Competitive parity with YouTube | High | Location tagging is table stakes for video SEO |

---

## 10. Privacy & Compliance

- **Consent (GPS):** Device GPS collection requires explicit, informed consent. Must comply with **GDPR** (EU) and **CCPA** (US). Consent must be logged with a timestamp.
- **IP-based hints:** Acceptable without consent as a UX aid — no raw coordinates are exposed or stored from this method.
- **Storage:** Consider whether approximate location (city-level) is sufficient vs. precise GPS. V1 recommendation: store city-level label + rounded coordinates.
- **Opt-out / removal:** Users must be able to remove location data after upload, via both UI and API.
- **Transparency:** The upload UI must clearly indicate that location data will be visible on the published video page.
- **Data minimisation:** Only collect the granularity needed. Full GPS precision is not required for V1 use cases.

---

## 11. Implementation Recommendations

| Concern | Recommendation |
|---|---|
| Geocoding + Autocomplete vendor | **Mapbox** (unified SDK, session-based pricing, single vendor) |
| Device location (web) | W3C Geolocation API (native browser, no dependency) |
| Device location (iOS) | CoreLocation / `CLLocationManager` |
| Device location (Android) | FusedLocationProviderClient (Google Play Services) |
| IP-based soft hint | MaxMind GeoIP2 (server-side) or `CF-IPCountry` header if on Cloudflare |
| Debounce on search input | 300ms debounce to limit API calls |
| Reverse geocode caching | Cache results by lat/lng rounded to ~100m |
| Forward geocoding (label only) | Mapbox Places API, limit=1, types=place,region,country |
| Schema.org markup | Add `contentLocation` to VideoObject structured data on video pages |
| Video sitemap | Include `<video:location>` tag for all location-tagged videos |
| Reusable component | Build a shared location picker component usable across web, embed, and back-office |

---

## 12. Open Questions & Decisions

| # | Question | Decision |
|---|---|---|
| 1 | Should location be publicly visible on the video page? | **Yes** |
| 2 | What granularity is required? | **City-level for V1** |
| 3 | Should location be editable after upload? | **Yes** (via UI and API) |
| 4 | Will location be used for content filtering in the future? | **Yes** (store structured coordinates for this) |
| 5 | Which geocoding/autocomplete vendor? | **TBD — Mapbox recommended** |
| 6 | Should IP-based hints be enabled on all surfaces? | **TBD** |
| 7 | How should consent be logged for GPS permission? | **TBD — needs input from Legal/Privacy** |
| 8 | Should location field be surfaced in creator analytics? | **TBD** |

---

*End of specification.*
