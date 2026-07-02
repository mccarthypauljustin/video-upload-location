# Technical Overview — How Geolocation Grabbing Works

This describes the three independent location-capture paths in the prototype (`prototype/video-upload-location-prototype.html`) and how they converge into a single structured object. It doubles as implementation reference for the production build.

---

## 1. Device GPS — "Use my current location"

Uses the browser-native **W3C Geolocation API** (no library, no key). On click, the code gates on consent (reveals the checkbox, logs a timestamp), then calls:

```js
navigator.geolocation.getCurrentPosition(
  onSuccess,   // (pos) => reverseGeocode(pos.coords.latitude, pos.coords.longitude, "device_gps")
  onError,     // (err) => err.code 1 = denied, 3 = timeout, else generic
  { timeout: 8000, maximumAge: 60000, enableHighAccuracy: false }
);
```

`getCurrentPosition` is callback-based and asynchronous: the browser shows its own permission prompt, and `onSuccess` only fires after the user grants access, receiving a `coords` object.

Why these options:

- `enableHighAccuracy: false` — requests coarse (WiFi/IP-derived, city-level) location instead of powering up the GPS chip. Faster, lower battery, and sufficient for V1.
- `timeout: 8000` — abandons the attempt after 8 seconds.
- `maximumAge: 60000` — allows a cached fix up to 60s old to be reused.

`onError` distinguishes permission-denied (code 1) from timeout (code 3) and routes the user to manual search in both cases.

**Native mobile equivalents:** iOS `CoreLocation` (`CLLocationManager`), Android `FusedLocationProviderClient`.

## 2. Reverse Geocoding — coordinates → readable label

GPS returns only raw lat/lon, so the success callback calls `reverseGeocode()`, which fetches Nominatim (OpenStreetMap) in the prototype:

```js
fetch(`https://nominatim.openstreetmap.org/reverse?format=json&zoom=10&lat=${lat}&lon=${lon}`)
```

- `zoom=10` requests city-level granularity.
- Parses `data.address`, falling back through `city → town → village → county` because place-name keys vary by country.
- Builds a `"City, Country"` label, rounds coordinates to **4 decimal places** (~11 m — ample for city-level), and stamps `source: "device_gps"`.
- On fetch failure it still stores the raw coordinates, so no data is lost.

**Production:** replace Nominatim with the chosen vendor (Mapbox recommended). Cache results by lat/lng rounded to ~100 m to cut API calls.

## 3. IP Soft Hint — no permission required

On page load, `loadIpHint()` fetches a coarse city from the caller's IP and populates the dismissible "Looks like you're near…" bar:

```js
fetch("https://ipapi.co/json/")  // returns { city, country_name, latitude, longitude, ... }
```

Coordinates are rounded to **2 decimal places** (coarser, ~1 km) to reflect the lower confidence. No permission is needed and no precise coordinates are exposed, which is why it's GDPR-safe as a UX hint.

**Production:** do this **server-side** via MaxMind GeoIP2 or the Cloudflare `CF-IPCountry` header — not a client-side call — so it costs nothing per request and never leaves the edge.

## 4. Convergence — one shape for all three

Every path ends in a single `setLocation()` call that normalizes to the same structured object:

```json
{
  "label": "Paris, France",
  "latitude": 48.8566,
  "longitude": 2.3522,
  "source": "user_search | device_gps | device_ip",
  "country_code": "FR",
  "city": "Paris",
  "region": "Île-de-France"
}
```

`setLocation()` also re-renders the live metadata preview (location JSON / Schema.org / API payload). Because normalization happens here, downstream code never needs to know which capture path produced the data — the `source` field is kept purely as an audit trail.

## Data flow summary

```
[GPS button] → getCurrentPosition → coords → reverseGeocode ─┐
[Search box] → debounce 300ms → Photon autocomplete ────────┤→ setLocation() → state + preview
[Page load]  → IP lookup → soft-hint bar → (user confirms) ─┘
```

## Provider swap for production

| Concern | Prototype (free) | Production (recommended) |
|---|---|---|
| Autocomplete | Photon (OSM) | Mapbox Geocoding (autocomplete=true) |
| Reverse geocode | Nominatim (OSM) | Mapbox Geocoding |
| IP hint | ipapi.co (client) | MaxMind GeoIP2 / CF header (server-side) |
| Device GPS | W3C Geolocation | unchanged (native browser API) |
