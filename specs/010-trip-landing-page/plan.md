# Implementation Plan: Trip Landing Page

**Branch**: `010-trip-landing-page` | **Date**: 2026-03-06 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/010-trip-landing-page/spec.md`

## Summary

Add a cinematic landing page that serves as the default entry point to the travel photo map. The landing page displays a full-screen stats intro ("42 days, 8 regions, 5 countries") that transitions to a grid of 8 region cards with hero photo backgrounds. Clicking a card opens a full-page detail view with a hand-authored summary, places/dates list, mini Leaflet map, and photo thumbnail grid. The landing page coexists with the existing map app in the same HTML page using a show/hide model.

## Technical Context

**Language/Version**: Vanilla JavaScript (ES2020+), CSS3, HTML5
**Primary Dependencies**: Leaflet.js (existing, vendored in `js/`)
**Storage**: Static JSON — extends existing `data/itinerary.json` with `summary` and `heroPhoto` fields per region; reads existing `data/manifest.json` for photos
**Testing**: Manual visual testing via Playwright MCP at 1440px and 375px
**Target Platform**: Static file hosting (GitHub Pages, any HTTP server, `python -m http.server`)
**Project Type**: Web (single-page, no build step)
**Performance Goals**: Intro renders within 500ms of page load; card grid visible within 2s; detail view opens within 1s of click
**Constraints**: No build tools, no npm, no frameworks. CSS transitions preferred over JS animation. Must not break existing map functionality.
**Scale/Scope**: 8 region cards, ~800 photos total, single HTML page

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Compliance | Notes |
| --------- | ---------- | ----- |
| I. Privacy by Default | Pass | No new external services, no analytics, no data transmission. Hero photos use existing URLs. |
| II. Static & Zero-Config | Pass | Pure HTML/CSS/JS, no API keys, no backend. Works with `python -m http.server`. |
| III. Approachable by Everyone | Pass | Large touch targets on cards, plain-language summaries, intuitive tap-to-expand interaction, no jargon. |
| IV. Professional Visual Polish | Pass | Hero photo cards, smooth CSS transitions, dark glass design tokens, cinematic intro. |
| V. Performant at Any Scale | Pass | Intro is text-only (instant). Hero images lazy-loaded. Thumbnail grid capped at ~30. Mini-map lazy-initialized. |
| VI. Unified Media Experience | Pass | Photo thumbnails in detail view. Full browsing deferred to the existing map app (accessible via "Explore the map"). |
| VII. Map-Centric Integration | **Justified exception** | The landing page is a *gateway* to the map, not a parallel surface. It exists to orient visitors before they enter the map-centric experience. The detail view includes a mini-map to maintain geographic context. See Complexity Tracking. |

## Project Structure

### Documentation (this feature)

```text
specs/010-trip-landing-page/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── spec.md              # Feature specification
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
js/
├── landing-page.js      # NEW — landing page module (init, card grid, detail view, transitions)
├── leaflet.js           # Existing — Leaflet map library
├── Leaflet.Photo.js     # Existing — photo marker layer
├── ViewportSampler.js   # Existing — viewport marker clustering
├── route-builder.js     # Existing — smart route lines
├── photo-viewer.js      # Existing — photo lightbox
├── photo-wall.js        # Existing — bottom panel photo grid
└── region-nav.js        # Existing — region grid + itinerary panel

css/
├── landing-page.css     # NEW — landing page styles
├── map.css              # Existing — main app styles
├── photo-wall.css       # Existing — photo wall styles (contains :root design tokens)
└── Leaflet.Photo.css    # Existing — photo marker styles

data/
├── itinerary.json       # MODIFIED — add `summary` and `heroPhoto` per region
├── manifest.json        # Existing (read-only)
└── trip_segments.json   # Existing (read-only)

index.html               # MODIFIED — add landing page container, link CSS, load JS
```

**Structure Decision**: Single-page web app. Two new files (`js/landing-page.js`, `css/landing-page.css`) plus modifications to `index.html` and `data/itinerary.json`. No new directories needed.

## Design Decisions

### D1: Landing Page as Overlay

The landing page is a `<div id="landing-page">` positioned fixed at z-index 2500, covering the entire viewport. The existing map initializes behind it (visibility hidden on sidebar/panels until landing closes). When dismissed, the landing page fades out and existing panels become visible.

**Why**: Minimal changes to existing code. Map can pre-load so transition is instant.

### D2: Module Pattern

`landing-page.js` exposes a single `window.initLandingPage(opts)` function, matching the pattern used by `region-nav.js`, `photo-wall.js`, etc. Called from the main inline script after data loads.

**Options object**:
```javascript
{
  itineraryData,    // loaded itinerary.json
  allPhotos,        // loaded manifest.json photos
  tripSegments,     // loaded trip_segments.json
  map,              // main Leaflet map instance
  onEnterMap        // callback when user clicks "Explore the map"
}
```

The `onEnterMap` callback lets the main app script control what happens when landing closes (show sidebar, show photo wall, etc.) without the landing module needing to know about those components.

### D3: Intro Screen

Full-viewport container with centered text. Three stat lines animate in sequentially with staggered delays. Auto-dismisses after 3.5s or on first user interaction (click/tap/scroll/keypress). Dismissal triggers a CSS `opacity: 0` + `transform: translateY(-100%)` transition, revealing the card grid underneath.

### D4: Card Grid Layout

CSS Grid: `grid-template-columns: repeat(4, 1fr)` on desktop (4x2), `repeat(2, 1fr)` on mobile (2x4). Each card has a fixed aspect ratio (16:9 via `aspect-ratio` or padding-bottom hack). Hero photo as `background-image` with `background-size: cover`. Region name and dates overlaid with text-shadow and a dark gradient scrim at the bottom.

### D5: Card Detail Takeover

On card click:
1. Capture clicked card's bounding rect.
2. Create/show detail container, initially positioned/scaled to match the card rect.
3. Animate to `transform: translate(0,0) scale(1)` filling the viewport (CSS transition, ~400ms).
4. Once transition ends, populate content: summary, places/dates, mini-map, thumbnail grid.

On close:
1. Reverse animation (detail shrinks back to card position).
2. Once collapsed, hide detail container, show card grid.

### D6: Mini-Map

A small Leaflet map instance (300x200 on desktop, full-width on mobile) inside the detail view. Initialized lazily on card open, destroyed on close. Uses the same tile URL as the main map. Shows a single marker at the region center. No zoom/pan controls (static display).

### D7: Thumbnail Grid

CSS grid of photo thumbnails filtered by the region's date range. Uses `thumbnail` URLs from manifest.json. Maximum 30 photos shown; if more exist, display a "+N more — view on map" link. Thumbnails are square-cropped via `object-fit: cover` in a uniform grid.

### D8: "Explore the Map" Button

A persistent button visible on the card grid (and optionally in the detail view as "View on map"). Clicking it:
1. Hides the landing page overlay (`opacity: 0`, then `display: none`).
2. Calls the `onEnterMap` callback which shows the feed sidebar, photo wall, and other panels.
3. If triggered from a region detail, also zooms the map to that region.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
| --------- | ---------- | ------------------------------------ |
| VII. Map-Centric: Landing page is not map-integrated | The landing page serves as a narrative gateway for non-technical visitors. It provides context ("42 days, 8 regions, 5 countries") before the map experience. The map is always one tap away. | Jumping straight to the map lacks the orienting narrative that friends/family need. The landing page is the front door, not a competing surface. |
