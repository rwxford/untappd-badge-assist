# Technical Plan — Untappd Badge Assist

> Companion to `PRD.md`. This plan is intentionally phased so we can ship user-controlled
> value first and treat Untappd integration as an upgrade, not a prerequisite.

## 1. Guiding Principle

**Core value must not depend on privileged Untappd badge data.** Untappd's public API does
not expose badge requirements/progress/expiration, and is hard to obtain; scraping risks
ToS violations. So we build on data the user provides (menu photos, inventory) plus a
catalog we maintain, and we layer Untappd integration on later if/when access is granted.

## 2. Data Strategy Options

| Option | Source | Pros | Cons |
|--------|--------|------|------|
| A. Curated catalog (recommended for v1) | Manually/community-maintained badge dataset | Full control, no ToS risk | Maintenance cost, can go stale |
| B. UTFB API | Untappd for Business (Premium venues) | Sanctioned menu/beer availability for venues | Venue-side only, no user badges, requires venue participation |
| C. Untappd public API v4 | Official ClientID/Secret | Check-ins, beer/venue lookups | Hard to obtain; no badge endpoints |
| D. Official partnership | Direct with Untappd | Could unlock real badge data | Slow, uncertain, may never happen |
| E. Scraping | App/web | "Complete" data | Likely ToS violation — excluded |

**Plan:** ship on **A** (+ optional **B** for participating venues), pursue **C/D** in
parallel as the path to Phase 3.

## 3. Proposed Architecture (Phase 1)

```
[Mobile app]
   |  capture/upload menu, manage inventory, view recommendations
   v
[Backend API]
   - OCR service (menu image -> parsed drink lines)
   - Matching/recommendation engine (drinks x badge catalog)
   - Badge catalog store + admin/curation
   - User store (inventory, preferences)
   v
[Data stores]
   - Badge catalog DB
   - User/inventory DB
   - Object storage for uploaded images
```

## 4. Suggested Stack (to confirm with @rwxford)

- **Client:** cross-platform mobile (React Native or Flutter) — camera, location, and push
  are first-class. *[DECISION NEEDED: platform]*
- **Backend:** a single API service (e.g., Node/TypeScript or Python/FastAPI).
- **OCR:** start with a hosted vision/OCR API; evaluate on-device OCR later for cost/privacy.
- **DB:** Postgres for catalog + user data; object storage for images.
- **Auth:** email/social to start; add Untappd OAuth in Phase 3.

These are defaults to accelerate discussion, not final choices.

## 5. Badge Catalog Data Model (draft)

```
Badge {
  id
  name
  category            // e.g. core, style, venue/local, seasonal, event
  requirement_type    // single_drink | n_distinct_styles | venue_category | nitro | ...
  requirement_detail  // structured criteria for matching
  time_sensitivity    // evergreen | monthly_window | seasonal | day_of_week | venue
  window              // optional start/end or recurring rule
  level_rules         // optional: cooldowns / per-period progress caps
  source / last_verified_at
}
```

Note real badges include rules like moving 30-day windows and per-period level cooldowns;
the model must represent time-sensitivity and cooldowns explicitly, and the UI should frame
outputs as *estimates*.

## 6. Phase Breakdown & Milestones

### Phase 0 — Validation
- [ ] Submit Untappd API request + start partnership outreach.
- [ ] Validate demand with badge-hunter communities.
- [ ] Lock platform decision and success metric.

### Phase 1 — MVP
- [ ] Badge catalog schema + seed dataset + curation workflow.
- [ ] Menu scan/upload + OCR + human-confirm parsing.
- [ ] Drink-to-badge matching engine.
- [ ] Manual inventory management.
- [ ] "What can I earn now" ranked view with expiry countdowns.

### Phase 2 — Location & Notifications
- [ ] Nearby venue/badge surfacing (location).
- [ ] Local/venue badges (UTFB for participating venues or curated).
- [ ] Push notifications + cadence controls.

### Phase 3 — Untappd Integration (gated on access)
- [ ] Untappd OAuth sign-in.
- [ ] Import check-in history to personalize / infer in-progress badges.
- [ ] Deep-link to Untappd to complete check-ins.

## 7. Open Technical Questions
1. Platform target (drives client stack).
2. Backend language preference / existing infra to reuse.
3. OCR: hosted vs. on-device; budget tolerance.
4. Catalog maintenance model (who curates, how often).
5. Hosting/deployment target.
