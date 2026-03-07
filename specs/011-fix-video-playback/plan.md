# Implementation Plan: Fix Video Playback

**Branch**: `011-fix-video-playback` | **Date**: 2026-03-07 | **Spec**: `specs/011-fix-video-playback/spec.md`
**Input**: Feature specification from `/specs/011-fix-video-playback/spec.md`

## Summary

Fix broken video playback in the photo viewer. Videos currently show "Photo unavailable" when clicked. The viewer already has a `renderVideo()` function that creates an iframe embed pointing to Google Drive `/preview` URLs, but it needs hardening: a loading state with thumbnail + spinner, proper 16:9 letterboxing, edge-zone swipe overlays for touch navigation around the iframe, and a video-specific error message. Additionally, the landing page detail grid needs play-button overlays on video thumbnails (already present in photo-wall and map markers).

## Technical Context

**Language/Version**: Vanilla JavaScript (ES5-compatible IIFE), CSS3, HTML5
**Primary Dependencies**: Leaflet.js (vendored), no new dependencies
**Storage**: N/A — reads `data/manifest.json` at runtime
**Testing**: Manual browser testing (desktop + mobile); Playwright screenshots
**Target Platform**: Web browsers (Chrome, Safari, Firefox) — desktop and mobile
**Project Type**: Single static web application
**Performance Goals**: Video playback begins within 3 seconds of click (SC-002)
**Constraints**: No build step, no npm, no server-side processing; iframe embeds only (Google Drive)
**Scale/Scope**: ~20 video entries in manifest alongside ~500 photos

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Compliance |
|-----------|--------|------------|
| I. Privacy by Default | PASS | No new data collection; videos use existing Google Drive URLs |
| II. Static & Zero-Config | PASS | Pure client-side changes; no server, no API keys |
| III. Approachable by Everyone | PASS | Play-button overlays, loading spinners, plain error messages |
| IV. Professional Visual Polish | PASS | 16:9 letterbox, loading states, smooth transitions |
| V. Performant at Any Scale | PASS | Iframe lazy-loads on click; no preloading of video content |
| VI. Unified Media Experience | PASS | Videos play inline in same lightbox as photos; play overlay distinguishes them |
| VII. Map-Centric Integration | PASS | Video markers already on map; no separate pages |

All gates pass. No violations to justify.

## Project Structure

### Documentation (this feature)

```text
specs/011-fix-video-playback/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
└── tasks.md             # Phase 2 output (/speckit.tasks)
```

### Source Code (files to modify)

```text
js/
├── photo-viewer.js      # Modify renderVideo() — loading state, 16:9, swipe zones, error
└── landing-page.js      # Add play-button overlay to video thumbnails in detail grid

css/
├── photo-viewer.css     # Add .pv-video-wrap, .pv-video-loading, .pv-swipe-zone styles
└── landing-page.css     # Add .detail-play-icon styles (if separate; may be inline)
```

**Already working (no changes needed):**
- `js/photo-wall.js` — play icon overlay already implemented (line 773)
- `js/Leaflet.Photo.js` — video badge already implemented (line 113)
- `js/ViewportSampler.js` — video type propagation already works (lines 191, 251)
- `css/photo-wall.css` — play icon styles exist (line 275)
- `css/Leaflet.Photo.css` — video badge styles exist (line 85)

**Structure Decision**: No new files. All changes are modifications to existing JS and CSS files in the repository root's `js/` and `css/` directories.

## Complexity Tracking

No constitution violations — table not needed.
