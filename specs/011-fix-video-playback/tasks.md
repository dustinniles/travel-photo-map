# Tasks: Fix Video Playback

**Input**: Design documents from `/specs/011-fix-video-playback/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md

**Tests**: Not requested — no test tasks included.

**Organization**: Tasks grouped by user story. Modifies 3 source files: `js/photo-viewer.js`, `css/photo-viewer.css`, `js/landing-page.js`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)
- Include exact file paths in descriptions

---

## Phase 1: Setup

**Purpose**: Verify current broken state before making changes.

- [x] T001 Start local dev server with `python3 -m http.server 8000` and verify current broken state by clicking a video thumbnail (expect "Photo unavailable" or broken playback)

---

## Phase 2: Foundational (CSS for Video Viewer)

**Purpose**: Add all CSS styles needed for the video viewer enhancements before changing JS rendering logic.

- [x] T002 Add `.pv-video-wrap` styles in `css/photo-viewer.css` after the existing `.pv-iframe` rule (~line 370): a 16:9 letterbox container using `position: relative; width: 100%; max-width: calc((100vh - 120px) * 16 / 9); max-height: calc(100vw * 9 / 16); margin: auto; background-color: #000; background-size: contain; background-position: center; background-repeat: no-repeat; display: flex; align-items: center; justify-content: center;`. The `.pv-media` container should use `display: flex; align-items: center; justify-content: center;` when a video is shown.

- [x] T003 [P] Add `.pv-video-spinner` styles in `css/photo-viewer.css`: a CSS-only loading spinner using `@keyframes pv-spin { to { transform: rotate(360deg); } }` with a `40px` circular border spinner (white semi-transparent, `border-top` solid white), absolutely positioned center within `.pv-video-wrap`. Add `.pv-video-wrap.pv-video-loaded .pv-video-spinner { display: none; }` to hide spinner when loaded.

- [x] T004 [P] Update `.pv-iframe` styles in `css/photo-viewer.css` (~line 364): add `opacity: 0; transition: opacity 300ms ease;` as default. Add `.pv-video-wrap.pv-video-loaded .pv-iframe { opacity: 1; }` to fade in iframe when loaded.

- [x] T005 [P] Add `.pv-swipe-zone` styles in `css/photo-viewer.css`: `position: absolute; z-index: 2; pointer-events: auto; touch-action: pan-x pan-y;`. Add `.pv-swipe-zone--left { left: 0; top: 0; width: 60px; height: 100%; }`, `.pv-swipe-zone--right { right: 0; top: 0; width: 60px; height: 100%; }`, `.pv-swipe-zone--top { top: 0; left: 60px; right: 60px; height: 50px; }`. Zones are transparent but `cursor: pointer` on hover.

- [x] T006 Add `iframe` to all existing `.pv-media img, .pv-media video` selectors in `css/photo-viewer.css` so iframe inherits the same max-width/max-height constraints: update the main selector (~line 86), mobile selector (~line 386), and desktop selector (~line 421) to include `.pv-media iframe`.

**Checkpoint**: All CSS ready — JS changes can now render the enhanced video viewer.

---

## Phase 3: User Story 1 — Watch a Video from the Map (Priority: P1) MVP

**Goal**: Rewrite `renderVideo()` to show a 16:9 letterboxed iframe with thumbnail loading state, spinner, and fade-in transition. Fix the core "Photo unavailable" bug.

**Independent Test**: Click any video thumbnail on the map or photo wall → viewer opens → thumbnail + spinner shows → Google Drive player fades in with playback controls at 16:9 aspect ratio.

### Implementation for User Story 1

- [x] T007 [US1] Rewrite `renderVideo(p)` in `js/photo-viewer.js` (~line 512-532): Create a wrapper div with `className = 'pv-video-wrap'` and `style.backgroundImage = 'url(' + p.thumbnail + ')'`. Inside it, create a spinner div with `className = 'pv-video-spinner'`. Create an iframe with `className = 'pv-iframe'`, `src = p.web_url`, `allow = 'autoplay; encrypted-media'`, `allowFullscreen = true`, `frameborder = '0'`. If `p.web_url` is falsy, call `errPlaceholder('video')` and return. Append spinner and iframe to wrapper, clear `$media.innerHTML`, append wrapper. Set `S.loaded = true`.

- [x] T008 [US1] Add iframe load handling in `renderVideo()` in `js/photo-viewer.js`: on `iframe.onload`, add class `'pv-video-loaded'` to the wrapper div (triggers CSS fade-in and spinner hide). Add a 15-second timeout — if the iframe hasn't loaded after 15s, call `errPlaceholder('video')`. Clear the timeout in `onload`. Attach `iframe.onerror` to also call `errPlaceholder('video')`.

- [x] T009 [US1] Add edge-zone swipe overlay divs inside the wrapper in `renderVideo()` in `js/photo-viewer.js`: create three divs with `className = 'pv-swipe-zone pv-swipe-zone--left'`, `'pv-swipe-zone--right'`, and `'pv-swipe-zone--top'`. Append them to the wrapper div after the iframe. These zones allow the existing pointer event handlers on `$wrap` to detect swipe gestures at the edges while letting center touches pass through to the iframe.

- [x] T010 [US1] Ensure `errPlaceholder()` in `js/photo-viewer.js` (~line 534) differentiates video errors: when `type === 'video'` display "Video unavailable", otherwise "Photo unavailable". Verify existing callers in `renderPhoto` pass no argument. (May already be done in working tree — verify and keep.)

- [x] T011 [US1] Playwright visual verification: screenshot the viewer at 1440px desktop width with a video open. Confirm: (1) 16:9 aspect ratio with black letterbox bars, (2) Google Drive player controls visible, (3) no "Photo unavailable" error. Screenshot at 375px mobile width and confirm same.

**Checkpoint**: Videos now play when clicked with loading state, 16:9 letterbox, and fade-in. Core bug is fixed.

---

## Phase 4: User Story 2 — Navigate Between Videos and Photos (Priority: P2)

**Goal**: Ensure clean transitions when swiping/arrowing between photos and videos — videos stop on navigate away, new videos load on navigate to, edge-zone swipes work for touch navigation.

**Independent Test**: Open a photo adjacent to a video, swipe at edge to the video (loads with loading state), swipe back to photo (video stops cleanly), close viewer (clean teardown).

### Implementation for User Story 2

- [x] T012 [US2] Update iframe cleanup in `js/photo-viewer.js`: In `renderVideo()` (~top of function), query `$media.querySelector('iframe')` — if found, set `iframe.src = 'about:blank'` before clearing `$media.innerHTML`. In `finalize()` (~line 238), add iframe cleanup: query `$media.querySelector('iframe')`, if found set `src = 'about:blank'` then remove. Keep existing `<video>` cleanup for safety.

- [x] T013 [US2] Verify swipe gesture handling works with edge-zone overlays in `js/photo-viewer.js`: the existing pointer event handlers on `$wrap` should detect swipes that start on the edge-zone divs (since they have `pointer-events: auto` and are children of the wrapper which is inside `$media`). Test by swiping at the left/right edges on mobile (375px) — confirm navigation triggers. If swipe detection doesn't work through the zones, attach the swipe start handler directly to the zone divs.

- [x] T014 [US2] Playwright visual verification: open a photo, navigate (arrow key) to an adjacent video, then back to a photo, then close. Screenshot at each step at 1440px. Confirm no visual glitches, no layout shifts, and clean transitions. Repeat at 375px mobile.

**Checkpoint**: Navigation between photos and videos works seamlessly with touch and keyboard.

---

## Phase 5: User Story 3 — Video Thumbnails on Landing Page (Priority: P2)

**Goal**: Add play-button icon overlay to video thumbnails in the landing page detail grid, matching the existing pattern in photo-wall and map markers.

**Independent Test**: Navigate to a region detail view that contains videos. Confirm video thumbnails show a centered play-button overlay while photo thumbnails do not.

### Implementation for User Story 3

- [x] T015 [US3] Update the detail grid thumbnail rendering in `js/landing-page.js` (~line 254-258): in the loop that builds `photoHtml`, check `photos[p].type === 'video'`. If video, wrap the `<img>` in a `<div>` with `position: relative; display: inline-block;` and append a play-icon overlay div inside (semi-transparent dark background, centered white triangle `\u25B6`, matching photo-wall's `.photo-wall-play-icon` styling). If photo, render the `<img>` as-is.

- [x] T016 [US3] Playwright visual verification: navigate to the landing page, expand a region that has videos (Copenhagen). Screenshot at 1440px. Confirm video thumbnails show play icons while photo thumbnails do not. Screenshot at 375px mobile.

**Checkpoint**: Video thumbnails are visually distinguished across all rendering contexts (map, photo wall, landing page).

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Final cleanup and full verification.

- [x] T017 [P] Verify zoom functions (`zoomAt`, `animResetZoom`, `animZoomAt`, `applyTx`) in `js/photo-viewer.js` safely no-op when video iframe is displayed: these query `$media.querySelector('img, video')` which returns null for iframes wrapped in `.pv-video-wrap`. Confirm no console errors when a video is open and user scrolls mouse wheel or double-clicks.

- [x] T018 [P] Clean up any unused `.pv-video` CSS class references in `css/photo-viewer.css` if videos no longer use a `<video>` element with that class. Check for `.pv-media video` selectors that may need `iframe` added or that reference non-existent elements.

- [x] T019 Run full quickstart.md validation: test all 10 verification steps from `specs/011-fix-video-playback/quickstart.md` on both desktop (1440px) and mobile (375px) viewports. Capture final screenshots.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — verify broken state
- **Foundational (Phase 2)**: CSS must be in place before JS changes
- **User Story 1 (Phase 3)**: Depends on Phase 2 CSS
- **User Story 2 (Phase 4)**: Depends on Phase 3 (renderVideo must exist before cleanup/swipe logic)
- **User Story 3 (Phase 5)**: Independent after Phase 1 — only modifies `js/landing-page.js`
- **Polish (Phase 6)**: Depends on all user stories complete

### User Story Dependencies

- **User Story 1 (P1)**: Independent after CSS foundation — core playback fix
- **User Story 2 (P2)**: Depends on US1 (renderVideo iframe + edge zones must exist)
- **User Story 3 (P2)**: Independent of US1/US2 — modifies different file (`landing-page.js`)

### Parallel Opportunities

- T003, T004, T005 can run in parallel (different CSS rule blocks, no dependencies)
- T015-T016 (US3) can run in parallel with Phase 3/4 (different file)
- T017, T018 can run in parallel (different concerns)

---

## Parallel Example: User Story 1

```bash
# CSS foundation tasks (all parallel — different rule blocks):
Task T003: "Add .pv-video-spinner styles in css/photo-viewer.css"
Task T004: "Update .pv-iframe opacity transition in css/photo-viewer.css"
Task T005: "Add .pv-swipe-zone styles in css/photo-viewer.css"

# US3 can run in parallel with US1 (different file):
Task T015: "Add play-icon overlay in js/landing-page.js"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. T001: Verify broken state
2. T002-T006: Add all CSS styles
3. T007-T011: Fix renderVideo with loading state, 16:9, swipe zones
4. **STOP and VALIDATE**: Click a video — it should play with loading animation and 16:9 aspect ratio

### Incremental Delivery

1. Phase 1-2: Setup + CSS → Foundation ready
2. Phase 3: US1 → Videos play with polish (MVP!)
3. Phase 4: US2 → Navigation works cleanly with edge swipes
4. Phase 5: US3 → Landing page thumbnails distinguished
5. Phase 6: Polish → Final verification

---

## Notes

- 3 source files modified: `js/photo-viewer.js`, `css/photo-viewer.css`, `js/landing-page.js`
- No manifest changes, no new files, no new dependencies
- Test videos: `IMG_7419.MOV`, `IMG_7422.MOV`, `IMG_7427.MOV` (Copenhagen, 2026-01-30)
- Existing video support in photo-wall.js, Leaflet.Photo.js, ViewportSampler.js requires NO changes
