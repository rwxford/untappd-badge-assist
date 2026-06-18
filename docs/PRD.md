# Product Requirements Document — Untappd Badge Assist

> Status: Draft v0.1 · Owner: @rwxford · Last updated: see git history
>
> This document is a starting point. Sections marked **[DECISION NEEDED]** require
> product input before engineering begins. Sections marked **[ASSUMPTION]** state a
> default we will proceed with unless changed.

---

## 1. Summary

Untappd Badge Assist is a companion app that helps Untappd users **earn more badges with
less effort**. It surfaces nearby and expiring badge opportunities, ranks badges by how
easy they are to earn given the user's location and the beers available to them, and lets
users scan or upload a menu to instantly see which badges the listed drinks could unlock.

It is a *companion* to Untappd, not a replacement: users still check in through Untappd
itself. Badge Assist is the planning and discovery layer on top.

## 2. Problem & Opportunity

Untappd has 15,000+ badges, many of which are time-sensitive (monthly windows, seasonal,
day-of-week, venue-specific local badges). Dedicated "badge hunters" actively plan which
check-in will earn the next badge, but Untappd's own app gives limited tooling for:

- Knowing **which badges are realistically within reach right now**.
- Knowing **which badges expire soon** and how long is left.
- Matching **a specific menu or personal inventory** of beers to badge requirements.

Badge Assist targets this planning gap.

## 3. Goals & Non-Goals

### Goals
- Help users discover badge opportunities they would otherwise miss.
- Reduce "wasted" check-ins by recommending high-value drinks for badge progress.
- Turn a physical/printed/online menu into actionable badge recommendations.

### Non-Goals (initial release)
- We will **not** perform check-ins on the user's behalf.
- We will **not** attempt to replicate Untappd's full social graph or feed.
- We will **not** guarantee badge awards (Untappd's awarding logic is authoritative).

## 4. Target Users / Personas

- **The Badge Hunter (primary):** power user, hundreds+ check-ins, actively chases badges,
  will plan a night around earning a specific badge.
- **The Casual Optimizer (secondary):** enjoys badges but won't go out of their way;
  wants a nudge when something easy is nearby or expiring.
- **[DECISION NEEDED]** Is there a B2B angle (bars/breweries promoting their local badges)?
  This materially changes data strategy and monetization.

## 5. Critical Constraint: Untappd Data Access

This is the single biggest risk to the product and must be resolved before committing to
the full feature set. Findings:

- **Untappd public API (v4):** requires a ClientID/ClientSecret granted via an application
  form with manual approval; the program has been effectively closed to most new
  third-party apps for years. Even where available, there is **no documented endpoint that
  exposes a user's badge progress, badge requirements, or badge expiration dates.** It
  exposes check-in feeds, beer/brewery/venue lookups, search, and user info.
- **Untappd for Business (UTFB) API:** available to Premium business subscribers; it is
  **venue/menu-side** (manage and read a venue's menus). It does not expose end-user badge
  state, but could be a legitimate source of *menu/beer availability* for participating
  venues.
- **Scraping the Untappd app/site** to obtain badge data would likely violate Untappd's
  Terms of Service and is **out of scope** as a sanctioned approach.

### Implication
The "automatically pull your active badges and requirements from Untappd" experience is
**not reliably buildable today** through sanctioned means. The product must be designed so
that its core value does **not** depend on privileged Untappd badge data. See the
Data Strategy options in `TECHNICAL-PLAN.md`.

**[DECISION NEEDED]** Pursue an official Untappd partnership / API access in parallel, and
treat deep Untappd integration as a *stretch* dependency rather than an MVP dependency.

## 6. Product Scope by Phase

### Phase 0 — Validation (no app build)
- Confirm Untappd data-access options (formal API request, partnership outreach).
- Validate demand with badge-hunter communities (Reddit, Discord, forums).
- Decide platform target (below).

### Phase 1 — MVP (assumes NO privileged Untappd badge data)
Core value the user controls entirely:
1. **Curated badge catalog** (community/manually maintained dataset of known badges,
   requirements, and any public expiration windows).
2. **Menu scan / upload (OCR):** user photographs or uploads a menu; app extracts drink
   names and matches them against badge requirements in the catalog.
3. **Inventory:** user manually adds beers/styles they have access to.
4. **"What can I earn?" engine:** given catalog + menu/inventory, rank badges by estimated
   effort (single drink vs. multi-step, time-sensitive vs. evergreen).
5. **Expiring-badge awareness:** highlight time-sensitive badges and show countdowns based
   on publicly known windows.

### Phase 2 — Location & Notifications
- Nearby venue/badge surfacing using device location + venue data.
- Local/venue badge support (sourced via UTFB for participating venues, or curated).
- Push notifications for expiring or newly nearby opportunities.
- **[DECISION NEEDED]** notification cadence + user controls.

### Phase 3 — Untappd Integration (stretch, gated on API access)
- OAuth sign-in with Untappd; import the user's check-in history to personalize
  recommendations and infer in-progress badges.
- Two-way: deep-link to Untappd to complete a check-in.

## 7. Feature Requirements (MVP detail)

### 7.1 Menu Scan / Upload
- Accept camera capture or image upload (and optionally a pasted text menu / URL).
- OCR extracts candidate drink lines; user can confirm/correct parsed items.
- Map parsed drinks to badge requirements; show matched badges with confidence.
- **[ASSUMPTION]** Start with a hosted OCR API for speed; revisit on-device later.

### 7.2 Badge Catalog
- Schema for badges: id, name, category, requirement type, requirement detail,
  time-sensitivity (evergreen / monthly / seasonal / day-of-week / venue), known window.
- Admin/curation workflow to add and update badges.
- **[DECISION NEEDED]** source of truth & maintenance model (manual, community-contributed,
  partnership-fed).

### 7.3 Inventory
- Add/remove beers or styles manually; persist per user.
- Optional: import from a user-provided export if available.

### 7.4 Recommendation / Ranking Engine
- Inputs: catalog, inventory, scanned menu, (later) location & time.
- Output: ranked list of "easiest to earn now," with reasoning and any expiry countdown.
- Effort model accounts for multi-step badges and per-window/level cooldowns where known.

## 8. Platform & Tech Direction

**[DECISION NEEDED]** Target platform. Options:
- **Mobile-first native/cross-platform (recommended):** menu scanning + location +
  notifications are core and mobile-native. Suggest React Native or Flutter for one
  codebase, or native iOS first if the audience skews iOS.
- **Web/PWA:** faster to ship, weaker camera/notification/location UX.

See `TECHNICAL-PLAN.md` for a proposed architecture and stack.

## 9. Success Metrics
- Activation: % of new users who complete one menu scan or add inventory.
- Core value: # of badge recommendations surfaced → badges users report earning.
- Retention: weekly active badge hunters; notification opt-in rate.
- **[DECISION NEEDED]** define the north-star metric (e.g., "badges earned attributable to
  the app per active user per month").

## 10. Monetization (placeholder)
**[DECISION NEEDED]** e.g., free core + premium (advanced alerts, unlimited scans), or
B2B venue promotion. No decision required for Phase 1.

## 11. Risks
| Risk | Impact | Mitigation |
|------|--------|-----------|
| No sanctioned Untappd badge data | High | Design MVP to not depend on it; pursue partnership in parallel |
| ToS / legal exposure from scraping | High | Do not scrape; use user-provided data, OCR, curated catalog, UTFB |
| Badge catalog goes stale | Med | Curation/community workflow; clearly label data freshness |
| OCR accuracy on messy menus | Med | Human-in-the-loop confirmation step |
| Badge rules are complex/changing | Med | Model time-sensitivity explicitly; conservative "estimated" framing |

## 12. Open Questions (consolidated)
1. Platform: iOS-first, Android, or cross-platform? Web/PWA acceptable for v1?
2. Is deep Untappd integration a hard requirement, or acceptable as a Phase 3 stretch?
3. B2C only, or is there a B2B venue angle?
4. Who owns/maintains the badge catalog, and where does its data come from?
5. What is the north-star success metric?
6. Monetization expectations, if any, for v1?
7. Geographic scope for launch (single market vs. global)?

---

*Next step: resolve the Open Questions, then convert Phase 1 scope into an engineering
backlog (see `TECHNICAL-PLAN.md`).*
