# Pint Marten — Product & Technical Spec (v0.2, draft)

> **Pint Marten** (a pun on the pine marten, a tree-climbing member of the weasel family).
> Repo name: `pint-marten`. This spec moves to that repository once it's created. See
> [§12 Naming](#12-naming).

## 1. Summary

A local-first PWA (Progressive Web App — a website that installs to the home screen,
works offline, and can use device GPS) that answers one question:

**"What beers, ciders and meads I haven't had, and places I haven't been, can I reach
from here in the time I have?"**

Scope covers **beer, cider and mead** (alcoholic and non-alcoholic). Wine and spirits are
out of scope. In this spec, "beer" in UI copy means any of the three unless stated
otherwise.

The user seeds the app with their drinking history, imported from beer-tracking apps or a
spreadsheet. The app then finds nearby venues within a walking or driving time limit,
ranks them by how new they'd be to the user, and hands off to Apple Maps or Google Maps
for directions. An airport mode does the same inside six specific airports, using gate,
security side and boarding time.

### Non-goals (for now)

- No social feed, check-ins, ratings community, or badges tied to any third-party app.
- No use of, scraping of, or branding resembling Untappd (or any other app's) API,
  website, or embedded menus. Imports are limited to **files the user exported
  themselves**.
- No accounts, servers, or cloud sync in Phase 1.

## 2. Audience & phasing

| Phase | Audience | Scope |
|---|---|---|
| **1 — Personal MVP (minimum viable product)** | Just the owner | Import, local data, map, travel-time search, walk/drive toggle, "new to me" ranking, NA filter, directions hand-off, backup/restore |
| **2 — Personal+** | Owner | Airport mode (6 airports), menu photo OCR (optical character recognition), achievements |
| **3 — Open source release** | Anyone | BYOK (bring your own API key), docs, contributor-maintained airport data, more importers, CLA bot (§9) |
| **4 — Premium (maybe; working name "Business of Ferrets")** | Paying users | Encrypted sync, shared tap-list data, hosted routing — all require hosting and are deferred |

## 3. Core user stories

1. **Import:** As a user, I can import my history from an export file (CSV, JSON or XLSX)
   so the app knows which beers, breweries and venues I've already had or visited.
2. **Find new places:** As a user, I can see venues I've never visited that are within
   *N* minutes of walking or driving.
3. **Find new beers:** As a user, I can see which nearby venues likely have beers I
   haven't had, ranked by how many.
4. **Non-alcoholic:** As a user, I can filter to venues/beers that are non-alcoholic.
5. **Go:** As a user, I can tap a venue and open walking or driving directions in Apple
   Maps or Google Maps.
6. **Airport:** As a user at IAD, DCA, BWI, CLT, ATL or DEN, I can enter my gate, which
   side of security I'm on, and my boarding time, and see what I can reach and still get
   back in time.
7. **Own my data:** As a user, all data stays on my device, and I can export a full backup
   file and restore it.

## 4. Feature specs

### 4.1 Data import

**Supported sources (Phase 1 ⇢ 3):**

| Source | Format | Status / notes |
|---|---|---|
| Untappd | CSV or JSON export (Insiders feature) | Phase 1. User-initiated export file only — no API use. Expected columns include `beer_name`, `brewery_name`, `beer_type`, `beer_abv`, `venue_name`, `venue_lat`, `venue_lng`, `created_at`, `rating_score`. **Verify against a real export.** |
| Generic spreadsheet | CSV, TSV, XLSX | Phase 1. Column-mapping wizard (below). Also covers any other app's export (e.g., BeerAdvocate, RateBeer) via saved mapping presets — no dedicated importers planned. |

**Generic mapper wizard**

1. User drops a file → parsed in a Web Worker (a background thread, so the UI doesn't
   freeze on large files).
2. App previews the first 20 rows and auto-guesses columns by header name
   (`beer`, `brewery`, `style`, `abv`, `venue`, `lat`, `lng`/`lon`, `date`, `rating`,
   `category`). If no category column exists, category is inferred from style (§4.2).
3. User confirms/adjusts mapping; only `beer` + `brewery` are required.
4. Mapping is saved as a named preset for re-imports.

**Normalization & de-duplication**

- Normalize names: lowercase, Unicode-fold accents, strip punctuation and corporate
  suffixes ("Brewing Company", "Brewery", "Co.", "Beer Co"), collapse whitespace.
- Beer identity key = `normalize(brewery) + "|" + normalize(beer)`. Exact key match merges
  automatically; near matches (string-similarity ≥ threshold) go to a review queue rather
  than merging silently.
- Re-importing the same file is idempotent (same source row → same record).
- Venue matching from import → OpenStreetMap: within 150 m **and** name similarity above
  a threshold → "visited". Imports without coordinates match on name + city, falling back
  to the review queue.

### 4.2 Local data model

All data lives on-device in IndexedDB (the browser's built-in database).

```
Drink       { id, producerId, name, category: beer|cider|mead, style, abv, isNA, sourceIds[] }
Producer    { id, name, kind: brewery|cidery|meadery|mixed, city, region, country, osmId? }
Venue       { id, osmId?, airportCode?, name, kind, lat, lng, tags, lastFetched }
Consumption { id, drinkId, venueId?, date, rating?, source, sourceRowHash }
Visit       { venueId, firstDate, count }               // derived
Sighting    { id, drinkId, venueId, seenAt, source: manual|ocr|paste }  // "it's on tap here"
ImportJob   { id, source, fileName, importedAt, rowCount, mappingPreset }
Place       { id, label: home|work|custom, name, lat, lng }      // saved locations, local only
Settings    { mode: walk|drive, maxMinutes, mapsApp, naFilter, categories[], routingKey?,
              airport: { dwellMin, bufferMin, securityMin }, ... }
```

**Category inference:** style containing "cider"/"perry" → cider; "mead"/"melomel"/
"cyser"/"metheglin"/"braggot" → mead (cyser and braggot are hybrids, so they are tagged mead
with the hybrid noted in style); everything else → beer. The user can correct any record.

### 4.3 Location & travel-time search

- **Location:** browser Geolocation API, requested only when the user taps "Find near me"
  (no location prompt on launch). Manual "search from here" by dropping a pin is also
  supported.
- **Mode toggle:** Walking ⇄ Driving, persistent setting, also a toggle on the map.
- **Time limit:** slider, 5–60 min (walking default 15, driving default 20).
- **How it works:**
  1. Request an **isochrone** (a polygon of everywhere reachable within *N* minutes by
     the chosen mode) from the routing provider.
  2. Fetch candidate venues inside that polygon's bounding box from OpenStreetMap
     (cached locally per area for 7 days).
  3. Keep venues inside the polygon (point-in-polygon test, done on-device).
  4. For the top ~25 candidates, request a **travel-time matrix** (exact minutes from
     the user to each venue) for accurate sorting and display.
- **Category filter:** Beer · Cider · Mead chips (multi-select, all on by default).
- **Venue types** (OpenStreetMap tags): `amenity=bar|pub|biergarten`,
  `craft=brewery`, `microbrewery=yes`, `craft=cider` (cideries), meaderies (OSM tagging
  is inconsistent — often `craft=winery` or `craft=brewery` plus a name match; verify
  during M3), `shop=alcohol|beverages`, and optionally
  `shop=supermarket|convenience` (relevant for the NA filter — see §4.5).

### 4.4 "New to me" ranking

Each venue gets a novelty score:

| Signal | Weight | Source |
|---|---|---|
| Venue never visited | High | Imports + manual visits |
| Known beers here I haven't had (count) | High | Sightings |
| Brewery/cidery/meadery whose drinks I've barely had | Medium | Imports: (# drinks from this producer I've had) vs. typical lineup size |
| Venue kind (taproom > bottle shop > bar > grocery) | Low | OSM tags |
| Travel time | Penalty | Matrix |

Filters: **Unvisited only**, **Has known new beers**, **Producers only** (breweries,
cideries, meaderies), **Open now**
(OSM `opening_hours` where present; shown as "unknown" otherwise).

This works on day one without any tap-list data: an unvisited brewery taproom is almost
guaranteed to be full of new beers. Beer-level precision improves as sightings
accumulate (§4.6).

### 4.5 Non-alcoholic (NA) filter

- Three states: **All** · **NA only** · **Hide NA**. Applies across beer, cider and mead
  (NA cider is common; NA mead is rare but handled the same way).
- A drink is NA if any of the following is true: ABV (alcohol by volume) ≤ 0.5% (the US
  legal threshold for "non-alcoholic"), its style contains "non-alcoholic" / "NA" /
  "alcohol-free", or its producer is on a curated list of NA-only producers.
- "NA only" mode also widens venue types to include supermarkets and convenience stores,
  since that's where most packaged NA beer is sold.
- Optional enrichment: Open Food Facts (an open product database with barcodes and ABV)
  for barcode-scanned packaged beers. Phase 2.

### 4.6 Knowing what's on tap (sightings)

There is no free, licensable, national "what's on tap" feed. The data strategy is:

1. **Phase 1 — Manual and paste.** At a venue, the user pastes menu text (e.g., copied via
   iOS Live Text or Android Lens) or types beers. A parser splits lines into
   brewery/beer/style/ABV candidates for the user to confirm. Each confirmed beer becomes
   a dated `Sighting`; sightings older than 30 days are shown as "stale".
2. **Phase 2 — Photo OCR on-device.** Tesseract.js (an open-source OCR engine that runs in
   the browser), so photos never leave the phone. Lazy-loaded only when used.
3. **Phase 3+ — Venue tap-list platforms** (Taplist.io, DigitalPour, Arryved, etc.) **only
   with written permission or published terms that allow it.** Explicitly excluded:
   scraping BeerMenus, Untappd for Business menu embeds, or any site whose terms forbid
   it. Many venues' web menus are powered by Untappd for Business, so menu-scraping from
   venue websites is off the table in general.

### 4.7 Directions hand-off

Tapping **Go** opens the user's preferred maps app (setting: Apple / Google / Ask):

- Apple: `https://maps.apple.com/?daddr={lat},{lng}&dirflg={w|d}`
- Google: `https://www.google.com/maps/dir/?api=1&destination={lat},{lng}&travelmode={walking|driving}`

The current mode toggle carries through to the maps app.

### 4.8 Airport mode

**Airports (Phase 2):** IAD, DCA, BWI, CLT, ATL, DEN.

**Inputs**

- Airport (auto-suggested when GPS is inside an airport boundary).
- Where I am: terminal/concourse and nearest gate.
- **Airside or landside** (past security or not).
- Boarding time (or "I have N minutes").
- Optional: destination gate, if different (connections).

**Logic**

- Time budget = boarding time − now − safety buffer.
- A venue is reachable if
  `walk(here → venue) + dwell + walk(venue → departure gate) ≤ budget`.
- **All three timing values are configurable**, with global defaults in Settings and a
  per-trip override on the airport screen (e.g., "quick one: 10 min dwell"):

  | Setting | Default | Range |
  |---|---|---|
  | Dwell time at venue | 20 min | 5–90 min |
  | Boarding safety buffer | 10 min | 0–45 min |
  | Security estimate (landside → airside) | 20 min | 5–90 min |

  Per-airport security overrides are also allowed (e.g., ATL longer than CLT).
- Landside users see landside venues, plus airside venues only if the budget can absorb
  the security estimate above (no live wait-time feed in Phase 2).
- Airside users never see landside venues, since they'd have to clear security again.
- Ranking reuses §4.4 (new-to-me first), then by slack time remaining.

**Airport data (hand-curated, versioned JSON in the repo)**

```
Airport   { code, name, timezone, boundaryPolygon }
Node      { id, kind: gate|checkpoint|train_stop|junction, concourse, side, label }
Edge      { from, to, walkMinutes, mode: walk|train|shuttle|moving_walkway }
Venue     { id, name, concourse, nearestNodeId, side, hours?, servesBeer, hasNA?,
            lastVerified }
```

- Walking times come from a small graph per airport (gates grouped into clusters) rather
  than a routing API. Routing APIs can't route indoors through airports.
- Inter-concourse trains (e.g., IAD AeroTrain, ATL Plane Train, DEN train) are edges with
  fixed times plus average wait.
- Each airport's layout, which concourses connect airside, and venue list are
  **verified at curation time** against the airport's official map and directory (facts
  only; descriptions aren't copied), plus personal visits. Each record has a
  `lastVerified` date, and anything older than 6 months is flagged in the UI.
- In-airport directions: show a text route ("Walk to Plane Train → Concourse B → toward
  B12, ~9 min"). Apple/Google Maps indoor routing is unreliable, so there's no hand-off.

### 4.9 Saved places

- The user can save **Home**, **Work**, and any number of custom places. They're stored
  locally only and included in backups.
- Any saved place can be used as the search origin instead of current GPS ("what's new
  within 20 min of home"), which also avoids a location prompt.
- Powers home-relative features: "Unvisited near home" filter and home-based
  achievements (§4.11).
- Saved places are never sent anywhere except as a routing origin (rounded, per §6).

### 4.10 Backup & restore

- **Export:** a single JSON file of all data plus settings, with a schema version.
- **Restore:** merge or replace.
- Reminder banner if no backup in 30 days (local-only data can be lost if the browser is
  cleared).

### 4.11 Achievements (Phase 2 — concept)

An original, game-style "achievement unlocked" system inspired by the LitRPG genre (a
fiction genre with video-game-like stats and announcements), with its own voice and names.

**Copyright/trademark guardrails:** game mechanics and genre tropes (achievements, tiered
reward boxes, a sardonic announcer) aren't protectable, but specific expression is. So:

- No character names, quotes, catchphrases, achievement titles or reward-box names from
  *Dungeon Crawler Carl* or any other work.
- No "Dungeon Crawler" or "Crawler" in the name or UI.
- The announcer voice is written fresh, and tiers and rewards use our own mustelid theme
  (below).
- Real-animal names are generic and free to use, but no memes or catchphrases attached to
  them (e.g., the "honey badger" viral video lines).

**Mustelid theme**

- **Tiers by animal size:** Least Weasel → Stoat → Mink → Marten → Otter → Giant Otter.
- **Unlock animation:** a "war dance" (the excited sideways hopping ferrets and weasels
  actually do) by the marten mascot.
- **Unlock sound:** a "dook" (the chirp ferrets make when happy). Mutable in Settings.
- **Announcer:** an original, dry, mildly sarcastic marten voice. All lines written
  in-house.

**Mechanics**

- Rules engine runs locally on events: new drink, new venue, new style, new producer,
  new category (first cider/mead), NA drink, airport visit, import completed.
- Full-screen "NEW ACHIEVEMENT" card with an announcer one-liner and a tier reward.
  Rewards are cosmetic: map themes, title flair, fog-of-war reveal effects.
- Retroactive: on first enable, replays imported history to award earned achievements.

**Example (original) achievements**

| Name | Trigger |
|---|---|
| *Terra Brewgnita* | First beer in a city you've never had one in |
| *Terminal Velocity* | A beer airside at 3 different airports |
| *Gate Expectations* | Visited an airport venue and reached the gate with ≥10 min to spare |
| *Designated Legend* | 10 different NA drinks |
| *Orchard Run* | First cider at 5 different cideries |
| *Honey, I'm Home* | First mead within 30 min of home |
| *Style Points* | 25 distinct styles |
| *Hometown Tourist* | 5 never-visited venues within 15 min of home |
| *Stout Stoat* | 10 different stouts |
| *Honey Badger* | 5 different meads |
| *Ferreted Out* | Visit a venue whose sighting you logged from a menu before going |
| *Pine Marten* | First drink at a venue in a forest or park (OSM land use) |

**Responsible-drinking rule:** no achievements for volume or speed in a single session
(e.g., "5 beers in an hour"). Breadth (new places/styles) and NA achievements only.

**Fog of war (optional):** the map is dimmed except for areas around venues the user has
visited, so exploring literally reveals the map.

## 5. Architecture & stack

Choices optimize for **performance** (fast load on mobile data, small bundles) and
**security** (minimal attack surface, no servers holding user data).

| Layer | Choice | Why |
|---|---|---|
| Language | TypeScript (strict) | Type safety across parsers and data model |
| UI framework | **SvelteKit** (static/SPA build) | Compiles to small, fast bundles with no virtual-DOM runtime; first-class static output |
| Build/PWA | Vite + `@vite-pwa/sveltekit` (Workbox) | Service worker, offline caching, installability |
| Local DB | IndexedDB via **Dexie** | Mature, small, works on iOS Safari; indexed queries for history lookups |
| Map rendering | **MapLibre GL JS** | Open-source vector maps, GPU-rendered, no vendor lock-in |
| Map tiles | **OpenFreeMap** (free, no API key); Protomaps self-hosted later | No keys or tracking in Phase 1 |
| Venues | OpenStreetMap via **Overpass API** | Free, open data; cached per area |
| Breweries (enrichment) | Open Brewery DB | Free, public-domain-style brewery list |
| Routing (isochrone + matrix) | **openrouteservice (ORS)** hosted API, free tier | Open source and OSM-based, so it can be self-hosted later (or swapped for Valhalla) without changing data sources |
| Geometry | Turf.js (modular imports only) | Point-in-polygon, distances |
| File parsing | Papa Parse (CSV), SheetJS (XLSX) in a Web Worker | Off main thread; handles large exports |
| OCR (Phase 2) | Tesseract.js, lazy-loaded | On-device; only downloaded when used |
| Testing | Vitest (unit), Playwright (end-to-end, incl. mobile viewport + mocked geolocation) | |
| Hosting | Static only: Cloudflare Pages or GitHub Pages | Serves app files only. **No user data is stored server-side.** Free. |

**About "no hosting":** the app files still need to be served from somewhere, but static
hosting has no backend, no database and no user data, and costs nothing. It's equivalent
to publishing a website, not running a service.

### Network calls (complete list, Phase 1)

| Destination | Sends | When |
|---|---|---|
| Static host | Nothing user-specific | App load/update |
| OpenFreeMap | Map tile coordinates (reveals approximate viewing area) | Map pan/zoom |
| Overpass API | Bounding box | Venue search (cached 7 days) |
| openrouteservice | User coordinates (rounded to ~100 m) + venue coordinates | Travel-time search |
| Open Brewery DB | Brewery name query | Optional enrichment |

Nothing else. No analytics or third-party scripts in Phase 1.

### Rate limits / quotas

ORS's free tier is roughly a few hundred isochrone and matrix requests per day (check
current limits). For a single user this is plenty. Results are cached by
(rounded location, mode, minutes) for 24 hours.

## 6. Security & privacy

- **Local-first:** history, sightings and settings never leave the device except in a
  user-initiated backup file.
- **CSP (Content Security Policy — a browser rule limiting what the page may load):**
  `default-src 'self'`; explicit allow-list for the tile, Overpass, ORS and Open Brewery
  DB hosts; no `unsafe-inline` or `unsafe-eval`; `frame-ancestors 'none'`.
- **Untrusted input:** imported files and pasted menus are treated as hostile. They're
  rendered as text only (never `innerHTML`), size-limited (50 MB), and parsed in a worker
  with a row cap.
- **Export safety:** CSV exports escape leading `= + - @` characters to prevent spreadsheet
  formula injection.
- **API keys:** the ORS key is stored locally (BYOK) and never committed. In Phase 3 open
  source builds, users supply their own key; docs explain how to restrict it.
- **Location:** requested only on explicit action; never stored with history unless the
  user logs a visit; rounded before sending to routing.
- **Supply chain:** lockfile committed, pinned versions, Dependabot/Renovate, minimal
  dependencies, Subresource Integrity for any CDN asset (prefer bundling instead).
- **Storage durability:** call `navigator.storage.persist()`; recommend installing to the
  home screen (iOS can evict storage for sites that aren't installed after a period of
  non-use).

## 7. PWA / platform notes

- Installable on iOS (Safari → Share → Add to Home Screen) and Android/desktop Chrome.
- Offline: app shell, last venue results, imported data, and airport JSON work offline.
  Travel-time search needs a network connection.
- iOS PWAs can't do background location or geofenced alerts, so "you're near a new
  brewery" notifications are out of scope until a native app or Phase 4.
- Airport auto-detection uses foreground GPS only.

## 8. UX outline

1. **Onboarding:** import file (or skip) → choose maps app → done. No location prompt yet.
2. **Home (map + list):** big "Find near me" button; Walk/Drive toggle; minutes slider;
   NA tri-state; filter chips (Unvisited, New beers, Breweries, Open now). List cards show
   name, kind, minutes, novelty reason ("Never visited · ~8 new beers"), and a Go button.
3. **Venue detail:** sightings (with freshness), your history there, "Add what's on tap"
   (paste/type/photo), Go.
4. **Airport mode:** airport picker → concourse/gate → airside/landside → boarding time →
   reachable list with "slack" minutes per venue.
5. **History:** searchable list of beers/breweries/venues; review queue for ambiguous
   matches.
6. **Settings:** routing key, defaults, backup/restore, data reset.

## 9. Licensing

**Model: open core, following coder/coder** (checked against that repo, 2026-10):

- **Core code: AGPL-3.0** in the root `LICENSE`. Anyone who hosts a modified version must
  publish their changes.
- **Premium code (Phase 4, only if built):** lives in a separate `premium/` directory
  under its own source-available license (`LICENSE.premium`), modeled on Coder's
  `LICENSE.enterprise`. The code is readable and modifiable, but running it requires a
  license key, the key check can't be removed or bypassed, and production use requires
  accepting the project's terms.
- **Contributions: CLA (Contributor License Agreement), decided.** Required before
  merging any outside pull request, so contributed code can also ship in Premium.
  Enforced with the `contributor-assistant/github-action` GitHub Action (the same one
  coder/coder uses): the PR author comments a fixed sentence to sign, and signatures are
  stored in a separate repo (e.g., `pint-marten-cla`). Bots like Dependabot are
  allow-listed. The CLA text gets a lawyer's review before the first outside PR.
- Until outside contributions are accepted, no CLA is needed; the owner holds all
  copyright.
- **Data licenses:** OpenStreetMap and Open Food Facts are ODbL (Open Database License):
  attribution required, and share-alike applies if we redistribute a derived *database*.
  Map attribution is shown in the UI. Curated airport JSON gets its own license
  (recommend CC BY 4.0).
- **Trademarks:** no third-party app names in the product name, icon or marketing.
  Importer labels may name the source format factually ("Import from Untappd export"),
  which is nominative fair use.
- Get a quick legal review of import wording and data use before any public or paid
  release.

## 10. Milestones (Phase 1)

| # | Milestone | Done when |
|---|---|---|
| M1 | Scaffold | SvelteKit + PWA installable, CSP in place, CI running lint/typecheck/tests |
| M2 | Import | Untappd export + generic mapper import into Dexie; review queue; idempotent re-import |
| M3 | Map & venues | MapLibre map, Overpass venue fetch + cache, visited/unvisited styling |
| M4 | Travel time | ORS isochrone + matrix, walk/drive toggle, minutes slider, ranked list |
| M5 | Novelty & NA | Scoring per §4.4, NA tri-state per §4.5 |
| M6 | Sightings | Paste/type menu parser, freshness |
| M7 | Hand-off & backup | Apple/Google deep links; export/restore; persistence prompt |

## 11. Decisions log

| # | Question | Decision |
|---|---|---|
| 1 | Drink scope | Beer, cider and mead; no wine/spirits. Vivino import dropped. |
| 2 | BeerAdvocate / RateBeer importers | None planned; generic mapper covers them if a file ever appears. |
| 3 | Airport timing defaults | 20 min dwell / 10 min buffer / 20 min security, all configurable (§4.8). |
| 4 | Saved home location | Yes; Home, Work and custom places, local only (§4.9). |
| 5 | Code license | AGPL-3.0 core; open-core Premium in a separate directory and license, like coder/coder (§9). |
| 6 | Contribution policy | CLA, enforced by a GitHub Action (§9). |
| 7 | Product name | **Pint Marten** (§12). |

### Still open

- Name clearance: trademark, app store and domain checks (§12).
- Mascot/logo design.

## 12. Naming

**Decided: Pint Marten.** It's a pun on the pine marten, a tree-climbing member of the
mustelid (weasel) family. It reads as beer at a glance, is easy to say and spell, and
gives the app a mascot and an achievement theme (§4.11).

- **Repo:** `pint-marten` (AGPL-3.0).
- **Mascot:** a pine marten. Original artwork only.
- **Feature names in the same theme:** "Ferret it out" (search button copy), "Gate Pour"
  (airport mode).
- **Avoid in names and marketing:** "Badger" (an established UK beer brand, Hall &
  Woodhouse's Badger Beers), "Wolverine" (Marvel / University of Michigan marks), and any
  third-party beer app name.

**Clearance status:** a web search (2026-10) found no app, beer or pub named "Pint
Marten". Nearby but distinct names already in use: PintPath, Pint Please, Real Ale
Finder, TapHunter. A web search is **not** a trademark clearance.

**Before creating the repo:**

1. USPTO (US Patent and Trademark Office) trademark search for "Pint Marten" and close
   variants, in the software class (009) and the app/online-services classes (042, 043).
2. App Store and Google Play search.
3. Domains: `pintmarten.app`, `pintmarten.com`.
4. GitHub username/org availability, if the repo should live under its own org.

**Other candidates considered:** Ferret Out, Pint Ferret, Hop Weasel, Dook, Brewvealer,
Fog of Ale, Terra Brewgnita, Pintrepid, Wanderpint, Hopscout, First Pour, Brewcharted.
