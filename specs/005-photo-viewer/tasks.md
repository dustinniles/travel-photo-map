# Tasks: Immersive Photo Viewer

**Input**: Design documents from `/specs/005-photo-viewer/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/photo-viewer-api.md

**Tests**: Not requested — no test tasks generated.

**Organization**: Tasks grouped by user story for independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

---

## Phase 1: Setup

**Purpose**: Create new module files with skeleton structure

- [x] T001 Create `js/photo-viewer.js` with module skeleton — viewer state object (from data-model.md Viewer State), `window.photoViewer` API stubs (`open`, `close`, `isOpen`), and custom event dispatch helpers (`photoviewer:open`, `photoviewer:close`, `photoviewer:navigate`, `photoviewer:favorite`)
- [x] T002 [P] Create `css/photo-viewer.css` with base overlay styles — fixed fullscreen overlay with dark background (`rgba(0,0,0,0.95)`), flexbox centering for media container, responsive breakpoints for mobile vs desktop layout, `z-index: 2000`, and `will-change: transform` on animated elements

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core viewer lifecycle that ALL user stories depend on

**CRITICAL**: No user story work can begin until this phase is complete

- [x] T003 Implement viewer DOM construction in `js/photo-viewer.js` — create overlay element once on first `open()` call, containing: `.pv-overlay` (backdrop), `.pv-media` (image/video container), `.pv-close` (close button), `.pv-nav-prev` / `.pv-nav-next` (navigation arrows), `.pv-info` (info panel shell for caption/date/tags/link/favorite), `.pv-zoom-controls` (zoom +/- buttons). Append to `document.body`.
- [x] T004 Implement FLIP expand-from-thumbnail open animation in `js/photo-viewer.js` — on `open(photos, startIndex, sourceElement)`: capture `sourceElement.getBoundingClientRect()`, position the viewer image at source rect using CSS `transform: translate() scale()`, show overlay with `opacity: 0`, then transition to final centered position and `opacity: 1` using CSS transitions (~250ms). Set `touch-action: none` on media container. Dispatch `photoviewer:open` event.
- [x] T005 Implement close animation and cleanup in `js/photo-viewer.js` — on `close()`: if source element is still in DOM and visible, reverse FLIP animation (shrink back to source rect), otherwise fade out. On transition end: hide overlay, reset zoom/pan state, cancel in-flight preloads, re-enable body scroll, unbind keyboard listeners. Dispatch `photoviewer:close` event.
- [x] T006 Implement `_showPhoto(index)` in `js/photo-viewer.js` — update `currentIndex`, set `<img>` src to `photo.thumbnail` for immediate display, update info panel content (caption, date, tags, favorite state, Google Photos link), show/hide nav arrows based on index bounds (hide if index is 0 or last), reset zoom to 1x. Dispatch `photoviewer:navigate` event.
- [x] T007 Implement background scroll lock in `js/photo-viewer.js` — on open: set `document.body.style.overflow = 'hidden'` and add `pointer-events: none` to `#map` element. On close: restore original values. Prevent `wheel` and `touchmove` events from propagating to the map.
- [x] T008 Wire `index.html` entry points — update `onPhotoClick(e)` (~line 684) to call `window.photoViewer.open(filteredPhotos, index, e.layer._icon || null)` instead of `openLightbox(index)`. Update `onFeedThumbnailClick(evt)` (~line 1025) to build the day's photo array from `dateIndex[date].photos`, find the clicked photo's index within that array, and call `window.photoViewer.open(dayPhotos, dayIndex, evt.target)`. Add `<script src="js/photo-viewer.js"></script>` and `<link rel="stylesheet" href="css/photo-viewer.css">` to `index.html` head.

**Checkpoint**: Viewer opens from map/feed with expand animation, shows thumbnail, navigates with arrow buttons, closes with shrink animation. No gestures or progressive loading yet.

---

## Phase 3: User Story 1 - Tap Photo on Mobile for Immersive View (Priority: P1) MVP

**Goal**: Full mobile gesture support — pinch-to-zoom (no stuck state), swipe navigation, swipe-down dismiss, tap to toggle controls

**Independent Test**: Tap any photo on a mobile device. Verify: expand animation, swipe left/right navigates, pinch zoom works without getting stuck, swipe down dismisses, single tap toggles UI.

### Implementation for User Story 1

- [x] T009 [US1] Implement gesture FSM core in `js/photo-viewer.js` — create `_gesture` object with `mode` (IDLE/PINCHING/PANNING/SWIPING_NAV/SWIPING_DISMISS), `pointers` Map for tracking active pointers by ID, and transition functions. Bind `pointerdown`, `pointermove`, `pointerup`, `pointercancel` on `.pv-media` container. On `pointerdown`: add to pointers map, call `setPointerCapture`. On `pointerup`/`pointercancel`: remove from map, transition to IDLE when all pointers lifted (reset all gesture-specific state). Use 10px movement threshold before committing to a gesture mode.
- [x] T010 [US1] Implement pinch-to-zoom in `js/photo-viewer.js` — in gesture FSM: when 2 pointers active and mode is IDLE, transition to PINCHING. Record initial distance between pointers and current scale. On `pointermove` with 2 pointers: calculate new distance, compute `newScale = initialScale * (currentDist / initialDist)`, clamp to 1.0–5.0, call `_zoomAt(newScale, midpointX, midpointY)`. `_zoomAt` applies CSS `transform: translate(Xpx, Ypx) scale(S)` with `transform-origin: 0 0`. On all pointers up: if scale < 1.2 snap back to 1.0 with transition; always transition to IDLE.
- [x] T011 [US1] Implement swipe-to-navigate in `js/photo-viewer.js` — in gesture FSM: when 1 pointer, zoom is 1x, and horizontal movement > 10px with angle < 30 from horizontal, transition to SWIPING_NAV. Apply horizontal translate to media element following finger. On pointer up: if swipe distance > 80px or velocity > 0.3px/ms, commit navigation (`_showPhoto(index ± 1)`) with slide-out/slide-in CSS transition. Otherwise snap back. Reset translate after transition.
- [x] T012 [US1] Implement swipe-down-to-dismiss in `js/photo-viewer.js` — in gesture FSM: when 1 pointer, zoom is 1x, and vertical downward movement > 10px with angle > 60 from horizontal, transition to SWIPING_DISMISS. Apply vertical translate and scale reduction (e.g., `scale = 1 - dragY/1000`) to media element. Reduce backdrop opacity proportionally. On pointer up: if drag > 150px, call `close()`. Otherwise snap back with CSS transition.
- [x] T013 [US1] Implement single-finger pan when zoomed in `js/photo-viewer.js` — in gesture FSM: when 1 pointer and zoom > 1x, transition to PANNING. Track pointer delta from start position, apply to translate values (`translateX += deltaX`, `translateY += deltaY`). Apply CSS transform. On pointer up: transition to IDLE. Optionally clamp translate to keep image edges within viewport.
- [x] T014 [US1] Implement mobile UI controls toggle in `js/photo-viewer.js` — on single tap (pointerdown+pointerup within 200ms and < 10px movement, not on a control element): toggle `.pv-controls-visible` class on overlay. When visible: show close button, info panel, nav arrows. Start 3-second auto-hide timer. When hidden: hide all controls. Clear timer on any interaction.
- [x] T015 [US1] Add mobile-specific CSS in `css/photo-viewer.css` — fullscreen mode (`width: 100vw; height: 100vh; height: 100dvh`), `touch-action: none` on media container, safe-area padding for notched devices (`env(safe-area-inset-top)`), `.pv-controls-visible` show/hide transitions (opacity 0.2s), large touch targets (min 44px) for close/nav buttons, swipe hint animation for media sliding.

**Checkpoint**: Complete mobile photo browsing — expand, swipe nav, pinch zoom (never stuck), swipe dismiss, tap controls. All on real mobile device or Chrome DevTools emulation.

---

## Phase 4: User Story 2 - Click Photo on Desktop for Google Photos-style View (Priority: P1)

**Goal**: Desktop interaction — hover nav arrows, keyboard shortcuts, scroll-wheel zoom, double-click zoom toggle, backdrop click close

**Independent Test**: Click any photo on desktop. Verify: expand animation, hover edges shows arrows, arrow keys navigate, scroll-wheel zooms at cursor, double-click toggles zoom, Escape closes, backdrop click closes.

### Implementation for User Story 2

- [x] T016 [P] [US2] Implement desktop navigation arrows in `js/photo-viewer.js` — show `.pv-nav-prev` / `.pv-nav-next` on mouse hover within 80px of left/right viewport edges. Hide when mouse moves away. Click handlers call `_showPhoto(index ± 1)`. Style: semi-transparent background, white chevron icon, 48px hit area, `border-radius: 50%`, fade in/out with CSS transition.
- [x] T017 [P] [US2] Implement keyboard navigation in `js/photo-viewer.js` — on `open()`: add `keydown` listener to `document`. ArrowLeft → `_showPhoto(index - 1)`, ArrowRight → `_showPhoto(index + 1)`, Escape → `close()`. On `close()`: remove listener. Guard against index bounds.
- [x] T018 [US2] Implement scroll-wheel zoom in `js/photo-viewer.js` — add `wheel` event listener on `.pv-media` with `{ passive: false }`. Call `evt.preventDefault()` to block page scroll. Calculate zoom factor: `deltaY < 0 ? 1.15 : 1/1.15`. Call `_zoomAt(scale * factor, evt.clientX, evt.clientY)`. Reuse same `_zoomAt` function from T010.
- [x] T019 [US2] Implement double-click zoom toggle in `js/photo-viewer.js` — add `dblclick` listener on `.pv-media`. If scale > 1.05: animate to scale 1.0, reset translate. Else: call `_zoomAt(2.5, evt.clientX, evt.clientY)`. Use CSS transition for smooth animation (0.25s ease-out).
- [x] T020 [US2] Implement mouse-drag pan when zoomed in `js/photo-viewer.js` — on `pointerdown` with `pointerType === 'mouse'` and scale > 1: track drag start. On `pointermove`: apply translate delta. Set cursor to `grab`/`grabbing`. Track `_didDrag` flag (3px threshold) to distinguish from click. On `pointerup`: reset.
- [x] T021 [US2] Implement desktop UI auto-hide in `js/photo-viewer.js` — on `mousemove` over overlay: show controls, reset 2-second hide timer. On timer expiry: hide controls (fade out with CSS transition). Close button always visible for 1s after open. Nav arrows only visible when mouse near edges (handled in T016).
- [x] T022 [US2] Implement backdrop click to close in `js/photo-viewer.js` — on click on `.pv-overlay` element (not children, not during drag): call `close()`. Check `_didDrag` flag to prevent closing after a pan gesture. Check `evt.target === overlay` to avoid closing on media/control clicks.
- [x] T023 [US2] Add desktop-specific CSS in `css/photo-viewer.css` — overlay centered layout (`max-width: 90vw; max-height: 90vh`), cursor states (`.pv-media.zoomed { cursor: grab }`, `.pv-media.zoomed:active { cursor: grabbing }`), nav arrow positioning and hover effects, backdrop click area styling, `@media (hover: hover)` queries for hover-only styles.

**Checkpoint**: Complete desktop photo browsing — expand, arrows, keyboard, scroll zoom, double-click zoom, pan, backdrop close. All mouse/keyboard interactions work.

---

## Phase 5: User Story 3 - Fast Progressive Image Loading (Priority: P2)

**Goal**: Thumbnail shown instantly, full-res crossfades in, adjacent photos preloaded, stale loads canceled on navigation

**Independent Test**: Throttle network to Slow 3G. Open a photo — thumbnail appears instantly. Full-res fades in when loaded (no pop-in). Navigate to next — already loaded from preload.

### Implementation for User Story 3

- [x] T024 [US3] Implement progressive loading in `js/photo-viewer.js` — in `_showPhoto(index)`: set `<img>.src = photo.thumbnail` immediately. Create `_pendingImage = new Image()`, set `_pendingImage.src = photo.web_url`. On load: if still on same index, swap `<img>.src` to full-res with a CSS `opacity` crossfade (add `.pv-loading` class during load, remove on swap; CSS transition on opacity 0.3s). On error: show error placeholder. Set `_currentLoaded = true` on success.
- [x] T025 [US3] Implement adjacent preloading in `js/photo-viewer.js` — after current photo's full-res loads (`_currentLoaded = true`): create `_preloadPrev = new Image()` for `photos[index-1].web_url` (if exists and type !== 'video') and `_preloadNext = new Image()` for `photos[index+1].web_url` (if exists and type !== 'video'). Store references for cancellation.
- [x] T026 [US3] Implement in-flight cancellation in `js/photo-viewer.js` — in `_showPhoto(index)` (called on every navigation): before loading new photo, cancel pending loads by setting `_pendingImage.src = ''`, `_preloadPrev.src = ''`, `_preloadNext.src = ''` and nullifying references. This prevents stale loads from interfering with the current photo and frees bandwidth.
- [x] T027 [US3] Implement error placeholder in `js/photo-viewer.js` and `css/photo-viewer.css` — on `<img>` or `<video>` error: show a styled placeholder (dark gray background, broken-image icon or text "Photo unavailable", centered). Ensure nav arrows still work so user can navigate past the broken entry. Add `.pv-error-placeholder` CSS class.

**Checkpoint**: Progressive loading works on throttled connections — instant thumbnails, smooth crossfade, preloading makes adjacent photos instant, errors handled gracefully.

---

## Phase 6: User Story 4 - Video Playback in Viewer (Priority: P2)

**Goal**: Videos display with correct aspect ratio, clean poster thumbnail, native controls, no distortion at any point

**Independent Test**: Open a video from the map or feed. Verify: clean thumbnail poster (correct aspect ratio), play button visible, video plays with native controls, no distortion during transition.

### Implementation for User Story 4

- [x] T028 [US4] Implement video rendering in `js/photo-viewer.js` — in `_showPhoto(index)` when `photo.type === 'video'`: replace `<img>` with `<video>` element. Set attributes: `poster` = `photo.thumbnail`, `controls`, `playsinline`, `preload="none"`, `src` = `photo.web_url || photo.url`. Use `object-fit: contain` to preserve aspect ratio. Stop any previously playing video before switching.
- [x] T029 [P] [US4] Add video-specific CSS in `css/photo-viewer.css` — `.pv-media video { object-fit: contain; max-width: 100%; max-height: 100%; }`, poster sizing to match container, play indicator overlay (centered play triangle, semi-transparent circle background), video controls visibility.
- [x] T030 [US4] Implement video preloading rules in `js/photo-viewer.js` — in adjacent preloading (T025): skip full-file preload for videos (check `photo.type === 'video'`). Only preload the video's `thumbnail` URL as an Image for poster display. Never set `_preloadNext/Prev.src` to a video file URL.

**Checkpoint**: Videos play correctly — clean poster, correct aspect ratio, native controls, no distortion. Adjacent video thumbnails preloaded but not full video files.

---

## Phase 7: User Story 5 - Preserve Existing Functionality (Priority: P3)

**Goal**: Favorites, captions, dates, tags, Google Photos link, caption/tag editing all work in the new viewer. Old lightbox code removed.

**Independent Test**: Open a photo, toggle favorite (star updates, persists). View caption, date, tags. Click Google Photos link. Editor: edit caption, add/remove tags. Open from both map markers and trip feed thumbnails.

### Implementation for User Story 5

- [x] T031 [US5] Implement info panel content in `js/photo-viewer.js` — in `_showPhoto(index)`: populate `.pv-info` with: caption (from photo object, cloud-merged before passing to viewer), date, tags as styled chips, Google Photos link (if `photo.google_photos_url` exists). Use same visual style as current lightbox info panel (gradient overlay at bottom, white text).
- [x] T032 [US5] Implement favorite button in `js/photo-viewer.js` — add `.pv-favorite` button in controls area. Display filled star (★) when `photo._isFavorite`, hollow star (☆) otherwise. On click: dispatch `photoviewer:favorite` custom event with `{ detail: { photo } }`. Update button state optimistically.
- [x] T033 [US5] Wire favorite event listener in `index.html` — add `document.addEventListener('photoviewer:favorite', function(evt) { ... })` that calls `toggleFavorite(evt.detail.photo)`, then `rebuildPhotoLayer()` and `buildPhotoIndex()`. Match the existing favorite toggle behavior from the old lightbox.
- [x] T034 [US5] Implement caption/tag editor integration in `js/photo-viewer.js` and `index.html` — in viewer: if `window.firebaseAuth && window.firebaseAuth.isEditor`, show caption click-to-edit and tag editor UI. On caption save: dispatch `photoviewer:caption-edit` event with `{ photoId, caption }`. On tag add/remove: dispatch `photoviewer:tag-edit` event with `{ photoId, tags }`. In `index.html`: listen for these events and call `window.cloudData.savePhotoCaption()` / `window.cloudData.savePhotoTags()`.
- [x] T035 [US5] Remove old lightbox JavaScript from `index.html` — delete the old `openLightbox()` function (~lines 241-662), `closeLightbox()` (~lines 664-667), `resetZoom()` (~lines 222-237), `lbZoom` state object (~line 219), `lightboxIndex`/`lightboxOverlay` globals (~lines 217-218), and the keyboard listener (~lines 669-674). Keep `onPhotoClick` and `onFeedThumbnailClick` (already rewired in T008).
- [x] T036 [US5] Remove old `.lightbox-*` CSS rules from `css/map.css` — delete all rules from `.lightbox-overlay` through the end of the lightbox section (~lines 644-836). This includes: `.lightbox-overlay`, `.lightbox-close`, `.lightbox-favorite`, `.lightbox-nav`, `.lightbox-media`, `.lightbox-zoom-controls`, `.lightbox-info`, `.lightbox-img`, `.lightbox-tag`, `.lightbox-link`, and all related hover/active states.

**Checkpoint**: All existing features work in the new viewer. Old lightbox code fully removed. No regressions.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Edge cases, performance, and final validation

- [x] T037 Handle edge cases in `js/photo-viewer.js` — max zoom cap (continue pinching at 5x does nothing, no browser-level zoom triggered via `touch-action: none`), rapid swipe handling (cancel in-flight transitions on new swipe, debounce navigation), single-photo mode (hide nav arrows, disable swipe-to-navigate), orientation change (listen for `resize` / `orientationchange`, re-center photo and reset zoom to fit new viewport).
- [x] T038 Add `<link>` to `css/photo-viewer.css` in `index.html` `<head>` section alongside existing stylesheets, and verify `<script>` tag for `js/photo-viewer.js` loads before the inline script block that references `window.photoViewer`.
- [x] T039 Run quickstart.md validation — walk through every item in the testing checklist (mobile: tap, swipe, pinch, dismiss, controls; desktop: click, arrows, keyboard, scroll zoom, double-click, escape; progressive loading: throttled network; video: poster, aspect ratio, playback). Document any issues found and fix.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 completion — BLOCKS all user stories
- **US1 (Phase 3)** and **US2 (Phase 4)**: Both depend on Foundational. Can proceed in parallel (US1 = touch handlers, US2 = mouse/keyboard handlers in same file but different functions)
- **US3 (Phase 5)**: Depends on Foundational. Can proceed in parallel with US1/US2 (modifies `_showPhoto` loading logic)
- **US4 (Phase 6)**: Depends on US3 (uses progressive loading/preloading infrastructure)
- **US5 (Phase 7)**: Depends on US1 + US2 (needs gesture system complete to test info panel interactions). T035/T036 (old code removal) should be done LAST after all features verified.
- **Polish (Phase 8)**: Depends on all user stories being complete

### User Story Dependencies

- **US1 (P1 Mobile)**: Foundational only — independently testable on mobile
- **US2 (P1 Desktop)**: Foundational only — independently testable on desktop
- **US3 (P2 Loading)**: Foundational only — enhances both US1 and US2
- **US4 (P2 Video)**: US3 (uses preloading infrastructure)
- **US5 (P3 Existing Features)**: US1 + US2 (info panel needs gesture system to not interfere)

### Within Each User Story

- Core logic before CSS refinements
- Gesture FSM before specific gesture implementations (US1)
- Progressive loading before preloading before cancellation (US3)
- Info panel before favorite button before editors (US5)
- Old code removal (T035, T036) is the absolute last step

### Parallel Opportunities

- T001 and T002 (Setup): different files, can run in parallel
- T016 and T017 (US2 nav arrows / keyboard): independent event systems
- T028 and T029 (US4 video JS / video CSS): different files
- US1, US2, and US3 can proceed in parallel after Foundational phase

---

## Parallel Example: After Foundational Phase

```
# These three user stories can proceed in parallel:
Agent A: T009 → T010 → T011 → T012 → T013 → T014 → T015  (US1: Mobile gestures)
Agent B: T016 + T017 → T018 → T019 → T020 → T021 → T022 → T023  (US2: Desktop interactions)
Agent C: T024 → T025 → T026 → T027  (US3: Progressive loading)

# Then sequentially:
T028 → T029 → T030  (US4: Video, depends on US3)
T031 → T032 → T033 → T034 → T035 → T036  (US5: Integration + cleanup)
T037 → T038 → T039  (Polish)
```

---

## Implementation Strategy

### MVP First (US1 Only)

1. Complete Phase 1: Setup (T001, T002)
2. Complete Phase 2: Foundational (T003–T008)
3. Complete Phase 3: US1 Mobile (T009–T015)
4. **STOP and VALIDATE**: Test on real mobile device — expand, swipe, pinch, dismiss
5. This alone fixes the most critical bugs (stuck zoom, poor immersion)

### Incremental Delivery

1. Setup + Foundational → Viewer opens and shows photos
2. Add US1 → Mobile gestures work → Test on phone (MVP!)
3. Add US2 → Desktop interactions work → Test on desktop
4. Add US3 → Progressive loading → Test on throttled network
5. Add US4 → Videos work correctly → Test with .mov files
6. Add US5 → Favorites/captions/editing work → Remove old code
7. Polish → Edge cases handled → Full validation

### Recommended Approach (Single Developer)

Sequential: Phase 1 → Phase 2 → Phase 3 (US1) → Phase 4 (US2) → Phase 5 (US3) → Phase 6 (US4) → Phase 7 (US5) → Phase 8 (Polish)

Each phase produces a testable increment. Stop after any phase to validate.

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- All file paths are relative to repository root `/home/bgural/photoMap/travel-photo-map/`
- Total: 39 tasks across 8 phases
- The old lightbox code stays in place until T035/T036 to allow side-by-side comparison during development
- Commit after each phase checkpoint for easy rollback
