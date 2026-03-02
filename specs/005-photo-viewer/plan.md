# Implementation Plan: Immersive Photo Viewer

**Branch**: `005-photo-viewer` | **Date**: 2026-03-02 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/005-photo-viewer/spec.md`

## Summary

Replace the current lightbox with an immersive photo viewer modeled after iPhone Photos (mobile) and Google Photos (desktop). The new viewer opens with an expand-from-thumbnail animation, uses Pointer Events for unified gesture handling (fixing the stuck-zoom bug), progressively loads images with preloading, and handles .mov videos with correct aspect ratios. Implemented as a standalone ES2020+ module (`js/photo-viewer.js`) with companion CSS, integrated via custom events.

## Technical Context

**Language/Version**: Vanilla JavaScript (ES2020+), CSS3, HTML5
**Primary Dependencies**: None new (Leaflet.js existing, vendored)
**Storage**: N/A — client-side only; reads from existing photo manifest and cloud data
**Testing**: Manual browser testing (Chrome DevTools device emulation + real mobile devices)
**Target Platform**: Mobile browsers (iOS Safari 15+, Chrome Android) + Desktop browsers (Chrome, Firefox, Safari)
**Project Type**: Single (static frontend, no build step)
**Performance Goals**: 200-300ms viewer open time, 60fps animations and gestures, <200ms navigation to preloaded photos
**Constraints**: No build step, no frameworks, no npm; vendored deps only; CSS transitions over JS animation; static deployment
**Scale/Scope**: Collections of 100-500+ photos; single viewer module ~400-600 lines JS + ~200 lines CSS

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Compliance | Notes |
|-----------|-----------|-------|
| I. Privacy by Default | **PASS** | No new data transmitted. All viewing is client-side. No analytics or tracking added. |
| II. Static & Zero-Config | **PASS** | Pure frontend module. No backend, no API keys, no build step. Works with `python -m http.server`. |
| III. Approachable by Everyone | **PASS** | Core goal. iPhone Photos / Google Photos patterns are universally familiar. Touch targets follow platform conventions. No technical vocabulary in UI. |
| IV. Professional Visual Polish | **PASS** | Core goal. Expand-from-thumbnail animation, smooth 60fps gestures, CSS transitions, consistent dark overlay design. |
| V. Performant at Any Scale | **PASS** | Progressive loading (thumbnail → full-res), adjacent preloading, in-flight cancellation, no DOM bloat (single viewer instance reused). |
| VI. Unified Media Experience | **PASS** | Photos and videos in same viewer. Favorites, captions, tags preserved. Videos play inline with native controls. |
| VII. Map-Centric Integration | **PASS** | Viewer is an overlay on the map, not a separate page. Dismisses back to map/feed. |

**Technology Constraints Check**:
- Plain HTML, vanilla JS, CSS — no framework ✅
- New JS module vendored into `js/` ✅ (it's our own code)
- New CSS in `css/` ✅
- No build step required ✅
- Entry point remains `index.html` ✅

**Post-Phase 1 Re-check**: All principles remain satisfied. The module architecture (custom events for decoupling) adds no unnecessary abstraction — it's the minimum needed to extract 420 lines of embedded code into a testable module.

## Project Structure

### Documentation (this feature)

```text
specs/005-photo-viewer/
├── plan.md              # This file
├── spec.md              # Feature specification
├── research.md          # Phase 0: Technical decisions
├── data-model.md        # Phase 1: Entity definitions and state model
├── quickstart.md        # Phase 1: Developer setup guide
├── contracts/
│   └── photo-viewer-api.md  # Module interface contract
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (repository root)

```text
js/
├── photo-viewer.js      # NEW: Photo viewer ES2020+ module
├── leaflet.js           # Existing: Leaflet map library
├── Leaflet.Photo.js     # Existing: Photo marker plugin
├── ViewportSampler.js   # Existing: Viewport density sampling
├── cloud-data.js        # Existing: Firestore CRUD
├── auth.js              # Existing: Google Sign-in
└── firebase-*.js        # Existing: Firebase SDK

css/
├── photo-viewer.css     # NEW: Photo viewer styles
├── map.css              # MODIFIED: Remove .lightbox-* rules (~lines 644-836)
├── leaflet.css          # Existing
└── Leaflet.Photo.css    # Existing

index.html               # MODIFIED: Remove old lightbox code, wire new module
```

**Structure Decision**: Single static frontend project. Two new files (`js/photo-viewer.js`, `css/photo-viewer.css`) added following the existing vendored-module pattern. Old lightbox code removed from `index.html` and `css/map.css`.

## Complexity Tracking

No constitution violations. No complexity justifications needed.
