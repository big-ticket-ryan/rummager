# Rummager — Fredonia Rezone Plan

Branch: `fredonia-rummager`
Started: 2026-05-11

This document captures the planning conversation for converting Rummager from a Belgium WI seeded app to a Fredonia WI app where every sale is published by its seller (no preloaded data).

---

## Context

- Single-file app at `index.html` (~2068 lines).
- **Backend already wired**: Firebase (anonymous auth + Realtime Database). Project `rummager-37958`.
  - `/groups/{code}/members` — live group location sharing (orthogonal feature, **keep**).
  - `/publicStops/{id}` — anyone-can-publish sale pins. Belgium-era flow had Save/Flag/Ignore decision modal — being replaced (see Phase 2).
- Map: Leaflet + OSM tiles, IndexedDB tile pre-cache.
- Local store key: `rummager.v1` → bumping to `rummager.fredonia.v1`.

---

## Locked decisions

| # | Decision | Notes |
|---|---|---|
| Backend | Firebase (already in place) | Build on top of `publicStops` |
| Single-file constraint | Keep | |
| Group sharing (👥) | Keep as-is | |
| Map center | `43.470, -87.853` zoom 14 | Village of Fredonia |
| Tile pre-cache bbox | `lat 43.30–43.65, lon -88.10 to -87.65` | Wider Ozaukee County |
| Server geofence (rules) | Tighten to wider Ozaukee bbox (above) | Matches tile bbox |
| Hours model | One window per day, multiple dates | `[{ date, open, close }]` |
| "NEW" badge duration | 7 days | Matches publicStops TTL |
| Server validation of new fields | Permissive in v1 | Tighten later if abuse |
| Sales per device | One (`mySaleId` single string) | Multi later if needed |
| Belgium publicStops cleanup | Admin "Wipe ALL" once after deploy | Or via console |
| Role state | **None** — no Seller/Shopper switcher | Owner detected via `mySaleId` |
| Welcome screen | Single intro card, single dismiss | No forking choice |
| New public sale notification UX | **Auto-merge** into local list as a normal stop, small "NEW" tag for 24h or until tapped | Kills the buggy Save/Flag/Ignore artifact |
| Generic price reduction icon | **$⬇** | One-tap, no composer |
| Item-specific price reduction icon | **⚡** | Composer with item rows; both can be active simultaneously |

---

## Data model

### localStorage (key: `rummager.fredonia.v1`)

```jsonc
{
  "version": 2,
  "welcomeSeen": false,
  "stops": [],                 // empty seed; auto-populated from publicStops
  "mySaleId": null,            // id of publicStop this device owns
  "newAcks": {                 // per-publicId: timestamp user first saw it
    "<publicId>": 1715000000000
  }
}
```

`stops[]` shape unchanged from existing app, plus optional:
- `publicId` — set when this stop mirrors a `/publicStops/<id>` entry
- `priceSlash` — mirrored from publicStops (see Phase 6)

### Firebase `/publicStops/{id}` — additive fields (existing required fields kept)

```jsonc
{
  // existing required:
  "addr": "…", "lat": 43.47, "lon": -87.85,
  "createdBy": "<uid>", "createdAt": 1715000000000,
  // new optional:
  "name": "Smith Family Sale",
  "sellerName": "Jane",
  "items": "…",
  "hours": [
    { "date": "2026-06-12", "open": "08:00", "close": "17:00" },
    { "date": "2026-06-13", "open": "08:00", "close": "12:00" }
  ],
  "pinVerified": true,
  "priceSlash": {
    "generic": { "active": true, "ts": 1715000300000 } | null,
    "items":   { "active": true, "ts": 1715000400000, "list": [
      { "name": "Oak dresser", "newPrice": "$40", "photo": "data:image/jpeg;base64,…" }
    ] } | null
  },
  "expiresAt": 1715600000000,
  "flags": { /* existing */ }
}
```

### Firebase `/publicStats` (new — Phase 8)

```jsonc
{
  "totals": { "visits": 1247, "publishes": 3, "slashes": 2, "routesGenerated": 41 },
  "online": { "<uid>": 1715000300000 },
  "daily":  {
    "2026-05-11": {
      "visits": 47,
      "uniques": { "<uid1>": true, "<uid2>": true }
    }
  }
}
```

Counts surfaced via the existing **tap-7-times admin sheet** under a new **📊 Stats** section.

---

## Phases

### Phase 0 — Strip Belgium, rebrand to Fredonia
- Delete the 65-stop seed array in `index.html`.
- Bump localStorage key → `rummager.fredonia.v1` (no migration).
- Replace map default center + tile pre-cache bbox.
- Empty-state UI: "No sales yet — be the first! Tap **+** to add yours."
- Update `README.md`, `PLAN.md`, `SETUP.md` copy: Belgium → Fredonia.

### Phase 1 — Welcome card
- First-load card (when `welcomeSeen=false`):
  - Brief copy: "Browse sales on the map, or tap **+** to publish your own."
  - Single **Got it** button → sets `welcomeSeen=true`.
- No role state. No switcher.

### Phase 2 — Publish-sale flow + auto-merge reconciler
- **+** FAB opens menu: "Add a stop (private)" / "Publish a public sale".
- **Publish a public sale** sheet:
  1. Sale name (required)
  2. Your name (optional)
  3. Items / description
  4. Address (required, geocoded)
  5. **Pin-verify**: floating screen-center pin over map → drag map under pin OR long-press to drop → **Confirm pin**
  6. Hours (Phase 3)
  7. Photos
- Saves `id` to `mySaleId`.
- Edit / Delete actions appear on the detail sheet only if it's `mySaleId`.

**Reconciler (replaces Belgium Save/Flag/Ignore flow):**
- `onValue('/publicStops')` snapshot → upsert each into `state.stops` matched by `publicId`.
- New entries appear as normal pins, status=`unvisited`, with a **"NEW"** tag on the list row.
- `newAcks[publicId]` set on first sight; tag clears 24h later or on first tap.
- Removed remote entries → silently removed from local list.
- `priceSlash` field updates flow through reconciler in real time.

### Phase 3 — Hours editor + display
- Editor: repeatable Day rows = date + open + close + remove button. "+ Add another day".
- Validation: open < close per row.
- Display on detail sheet: compact list, plus an **Open now / Opens at … / Closed** pill computed client-side.
- List row: tiny green "Open now" tag when applicable.

### Phase 4 — Filter dropdown + map-respects-filter + relabels
- Replace chip row with single **Filter ▾** dropdown (single-select).
- Items: **All / Unvisited / Visited / Revisit / Must visit**.
- Search bar takes the freed horizontal space.
- Map removes non-matching markers when filter ≠ All; auto-fits remaining bounds.
- Route button label → **🗺️ Route Must Visits**.

### Phase 5 — Search UX polish
- Wrap search input in a `<form>` so Enter/Go submits → `blur()` → keyboard collapses.
- Add small **🔍** send button inside input (right side) for the same blur.
- Live-as-you-type filtering remains.

### Phase 6 — Price reductions (two icons)
- Detail sheet of `mySaleId` sale gets two buttons:
  - **$⬇ Mark prices reduced** — one-tap, sets `priceSlash.generic.active=true`. Marker shows **$⬇** badge. Banner: "$⬇ Prices reduced!".
  - **⚡ Reduce specific items…** — opens composer (item rows: name + new price + optional photo). Sets `priceSlash.items.active=true` + `list[]`. Marker shows **⚡** badge (with count if >1). Banner: "⚡ Lightning deals!" + item list.
- Both can be active simultaneously; badges stack on marker.
- Independent **Clear** actions per type.

### Phase 7 — Polish, docs, deploy
- Update `PLAN.md` decisions log + `README.md` lead-in + `SETUP.md`:
  - New Firebase rules (geofence + new fields).
  - Belgium publicStops wipe step (admin tap-7 → Wipe ALL).
  - Document seller flow + price reductions.
- Manual smoke test on phone.

### Phase 8 — Usage stats (Firebase RTDB counters)
- On boot: increment `publicStats/totals/visits`, `publicStats/daily/<today>/visits`, set `publicStats/online/<uid>=serverTimestamp()`, register `onDisconnect` to remove.
- `publicStats/daily/<today>/uniques/<uid>=true` for daily-unique counting.
- Counters fired wherever an action runs: `publishes`, `slashes`, `routesGenerated`.
- Admin sheet gets **📊 Stats** section: totals + online count + last 7 days table.
- Rules: `/publicStats/totals/*` and `/publicStats/daily/*` writable by any authed user (increment-only convention; village-scale acceptable). `/publicStats/online/<uid>` writable only by self.

---

## Things explicitly NOT changing

- Group sharing (👥) feature.
- Photo compression pipeline.
- Tile pre-cache mechanism (only bbox values change).
- Admin tap-7-times wipe trigger.
- localStorage backup of locally-added private stops.

---

## Risks

- **One-sale-per-device** in v1. Upgrade to `mySaleIds[]` later if needed.
- **Auto-merge spam vector**: deliberately accepted for village-scale. Flag action retained for emergencies.
- **Rules update** must be published in Firebase console after Phase 7 deploy.
- **Existing Belgium publicStops** must be wiped once after Fredonia deploy (admin tap-7 → Wipe ALL).
