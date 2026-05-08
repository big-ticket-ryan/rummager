# Rummager — Plan

A single-file HTML app to help you work a rummage-sale tour: see all stops on a map, mark which you've been to, flag ones to revisit, take notes, attach photos, and quickly drop a pin at your current GPS location for stops you discover ad-hoc.

Status: **READY TO BUILD** — one hosting decision still open (see §7).

---

## 1. Goals

- **Single self-contained HTML file** — no build step, no backend, no install.
- Mobile-first (Android primary), phone-friendly. Map and list both collapsible.
- Pre-loaded with the 65 Belgium 2026 rummage sale stops, using the curated addresses + manual lat/lon overrides already proven in `belgium-rummage-map.html`.
- Track per-address: **status** (Unvisited / Visited / Revisit), **note** (free text), **photos** (0..N, compressed).
- Add new addresses two ways: (a) manual form, (b) GPS quick-add.
- Persist everything in `localStorage` — no server, no account.
- Works offline once page is loaded (after tile pre-cache).

## 2. Non-goals (v1)

- No multi-user sync, no cloud backup, no export/import (deferred).
- No route optimization / TSP solver.
- No barcode/QR or price tracking.
- No "reset to factory defaults" button.

## 3. Primary device & UX shape

- **Android phone first.** iPhone supported as long as it doesn't compromise the Android UX or the single-file constraint (see §8 — no features lost on iOS).
- Layout is a vertical stack with collapsible sections:
  1. **Header bar** (always visible): title, status counts (e.g. `12 visited · 4 revisit · 49 to go`), filter chips, GPS quick-add FAB.
  2. **Map panel** (collapsible — "hide map" button).
  3. **List panel** (collapsible — "hide list" button).
- On desktop the map and list go side-by-side via media query.
- A **bottom sheet** opens for "Add address" / "Edit address" / "Detail view" so it feels native on phone.

## 4. Data model

Single localStorage key: `rummager.v1`.

```jsonc
{
  "version": 1,
  "stops": [
    {
      "id": "uuid-or-slug",
      "addr": "5780 Lake Church Rd",
      "area": "Lake Church",         // optional
      "items": "Honey, toys, books…", // original flyer description (preloaded only)
      "lat": 43.4995,
      "lon": -87.8207,
      "source": "preloaded",          // "preloaded" | "manual" | "gps"
      "status": "unvisited",          // "unvisited" | "visited" | "revisit"
      "note": "",                     // user's free-text note
      "photos": ["data:image/jpeg;base64,…"], // compressed before storage
      "createdAt": 1715000000000,
      "updatedAt": 1715000000000
    }
  ]
}
```

- All fields editable, including on preloaded stops.
- Editing `addr` triggers re-geocoding (Nominatim → Census fallback, same chain as the existing map).

## 5. Statuses & visuals

| Status      | Marker color | Sidebar dot |
|-------------|--------------|-------------|
| Unvisited   | red          | red         |
| Visited     | green (✓)    | green       |
| Revisit     | orange (★)   | orange      |

Status changes from the detail sheet (3-button selector) or via a quick-action on the marker popup.

## 6. Features (v1)

### 6.1 Map
- Leaflet + OSM tiles (same stack as `belgium-rummage-map.html`).
- Custom SVG markers per status.
- Tap marker → opens the **detail sheet** (not a popup) so you can edit on phone.
- "Locate me" control → centers map and shows a **live GPS dot** that updates as you move (`watchPosition`).
- **Pre-cached tiles** for the Belgium WI bounding box (see §6.7).

### 6.2 List / sidebar
- Each row: status dot, address, area, truncated note/items, small thumbnail if a photo exists.
- Tap row → opens the detail sheet and pans the map to the marker.
- **Filter chips** at the top: `All`, `Unvisited`, `Visited`, `Revisit`. Multi-select.
- Search box filters by address, area, items, or note.
- Sort menu: `Default order` | `Status` | `Distance from me` (requires GPS permission).

### 6.3 Detail sheet (per address)
- Address (always editable; on save, if changed, re-geocode in background).
- Area, Items, Note (all editable, multiline note autosaves on blur).
- Status selector (3 buttons).
- Photos: thumbnail grid + "Add photo" button (`<input type="file" accept="image/*" capture="environment">` so phone offers camera or gallery).
  - **On-the-fly compression:** each incoming image is drawn to a `<canvas>`, scaled so longest edge ≤ 1280 px, exported as JPEG quality 0.7. Typical result: 150–300 KB per photo.
  - **No hard cap** thanks to compression. A soft warning appears when total `localStorage` use exceeds ~4 MB (browser cap is ~5 MB), suggesting cleanup.
  - Tap thumbnail → full-size lightbox with delete button.
- **Navigate here** button → opens `https://www.google.com/maps/dir/?api=1&destination=LAT,LON` (iOS handles this URL → Apple Maps).
- Delete button (only for `source: manual` and `source: gps` stops; preloaded stops can be edited but not deleted).

### 6.4 Add address — manual
- Bottom-sheet form: address, optional area, note. Geocodes on save (Nominatim → Census fallback). If both fail, prompts user to long-press the map to drop the pin manually.

### 6.5 Add address — GPS quick-add (the "FAB")
1. Tap the floating **+ GPS** button.
2. Browser asks for location permission; capture lat/lon.
3. Reverse-geocode via Nominatim → prefill address field.
4. Show a quick bottom-sheet: address (prefilled, editable), note (focused).
5. Save → marker appears, list updated, **status defaults to `visited`** (you're standing there).
6. User can flip to `revisit` or `unvisited` later as normal.

### 6.6 Live position
- "Locate me" toggle in map controls. When on, `watchPosition` renders a blue pulsing dot that updates in real time.

### 6.7 Tile pre-caching (offline support)
- On first load (or via "Download offline tiles" button), the app pre-fetches OSM tiles for the Belgium WI bounding box at zoom levels ~12–17 and stores them in **IndexedDB** (separate budget from localStorage; typically 50+ MB available).
- Custom Leaflet `TileLayer` reads from IndexedDB first, falls back to network.
- Estimated size: ~5–15 MB depending on zoom range.
- Progress bar shown during pre-cache; user can skip if they have strong signal.

## 7. Open question — hosting

The single-file constraint **conflicts** with how mobile browsers gate `geolocation`:

- Android Chrome and iOS Safari **block geolocation on `file://`**. It requires HTTPS or `localhost`.
- Opening `rummager.html` directly from your phone's Downloads folder gives you the map and list, but GPS quick-add and live position will silently fail.

These three options preserve "single file" and avoid running a backend — pick one:

| Option | What you do | Pros | Cons |
|--------|-------------|------|------|
| **A. Netlify Drop** | Drag `rummager.html` onto https://app.netlify.com/drop. Get an HTTPS URL in ~30 s. Bookmark on phone. | Zero account, zero setup. Truly drag-and-drop. | URL is random; redrop to update. |
| **B. GitHub Pages** | Enable Pages on this repo (or a small public one). Auto-publishes on git push. | Stable URL forever, version-controlled, you already have repos. | One-time 5-min setup; repo must be public OR Pages-enabled on a private repo. |
| **C. Drop GPS, accept `file://`** | Skip live GPS / GPS quick-add. Manual add only. Open `.html` from Downloads. | Truly zero hosting. | Loses the headline GPS quick-add feature. |

**Recommendation: A (Netlify Drop)** — best simplicity match. B if you want a stable URL.

→ **Tell me A, B, or C and I'll proceed to Phase 1.**

## 8. iOS Safari notes

If iPhone friends use the same hosted URL (option A or B), expected behavior:

| Feature | iOS Safari behavior |
|---------|--------------------|
| localStorage | Works, ~5 MB cap (same). |
| Geolocation | Works on HTTPS. |
| Photo capture | `accept="image/*" capture="environment"` opens camera. Works. |
| Canvas image compression | Works. |
| Navigate-to link | Opens Apple Maps. Works. |

**One caveat:** if an iOS user adds the page to their Home Screen as a PWA, Safari may use a separate localStorage scope from the regular Safari tab. Worth a one-time mention in a help blurb. **No features lost on iOS** — proceeding with iOS support included.

## 9. Technical notes / risks

- **Nominatim usage policy:** 1 req/sec, custom `User-Agent` (already done in the existing map). One reverse-geocode per quick-add — well under limit.
- **Census Bureau geocoder fallback:** already proven on the existing map for rural WI county roads.
- **localStorage cap:** ~5 MB. Mitigated by canvas compression (§6.3) and IndexedDB for tiles (§6.7).
- **Single-file caveat:** Leaflet is loaded from the `unpkg.com` CDN (same as existing map). This requires internet on first visit even when hosted. If you want **truly** offline-first, we'd inline Leaflet's JS+CSS (~150 KB) into the HTML. Easy to add — say the word.

## 10. File layout

- `local-temp/rummager.html` — the app (single file).
- `local-temp/rummager-plan.md` — this document.

## 11. Build phases

1. **Phase 0 — confirm plan + pick hosting (§7).** ← we are here.
2. **Phase 1 — skeleton.** Mobile layout, collapsible map/list, preloaded 65 stops render with seeded coords (no startup geocoding — coords are baked in from the existing map), status toggle works, persists to localStorage.
3. **Phase 2 — detail sheet.** Note autosave, photo capture + canvas compression + thumbnails + lightbox + delete.
4. **Phase 3 — add flows.** Manual add form (with geocoding chain). GPS quick-add with reverse-geocode. Live GPS dot. Navigate-here button.
5. **Phase 4 — polish.** Filter chips, search, distance sort, status counts in header. Edit-preloaded with re-geocode-on-change. Long-press map to drop pin manually if geocode fails.
6. **Phase 5 — offline tiles.** IndexedDB tile cache + "Download offline tiles" button.
7. **Phase 6 — deploy** per chosen hosting option.

### Future / deferred ideas (not v1)

- Export/Import JSON for backup or device transfer.
- Inline Leaflet for full offline-first.
- Reset-all-to-defaults if preload edits become a problem.
- PWA manifest + service worker for full home-screen-app feel.

---

## Decisions log

_Append-only. Date + decision._

- 2026-05-08 — storage: localStorage only.
- 2026-05-08 — no export/import in v1.
- 2026-05-08 — preloaded: Belgium 65 stops; reuse the curated addresses + `manualCoords` from `belgium-rummage-map.html` (already hand-corrected and known good).
- 2026-05-08 — statuses: Unvisited / Visited / Revisit.
- 2026-05-08 — GPS quick-add reverse-geocodes via Nominatim and prompts for a note. **Defaults to `visited`** (user is on-site).
- 2026-05-08 — preloaded addresses ARE editable; edits re-trigger geocoding chain.
- 2026-05-08 — photos: compress on the fly (canvas → JPEG q=0.7, ≤1280 px). No hard cap. Soft warning at ~4 MB localStorage usage.
- 2026-05-08 — no reset-all button in v1.
- 2026-05-08 — pre-cache OSM tiles for Belgium WI bounding box into IndexedDB.
- 2026-05-08 — primary device: Android phone (mobile-first, collapsible map and list). iOS supported via HTTPS hosting; no feature loss on iOS.
- 2026-05-08 — single-file HTML constraint is firm. No backend logic. Static drop-hosting (Netlify Drop or GitHub Pages) is acceptable since it's still "one file" served statically.
