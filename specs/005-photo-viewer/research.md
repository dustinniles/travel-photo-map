# Research: Immersive Photo Viewer

**Feature Branch**: `005-photo-viewer` | **Date**: 2026-03-02

## Decision 1: Gesture Handling Approach

**Decision**: Use the Pointer Events API as the unified input model for all interactions (mouse, touch, pen).

**Rationale**: The current implementation uses separate `mousedown/mousemove/mouseup` and `touchstart/touchmove/touchend` handlers with shared closure variables (`_dragStartX`, `_touchDist0`, etc.). This creates the "stuck zoom" bug: when a user transitions between pinch and pan gestures, stale closure variables from the previous gesture phase produce incorrect calculations. Pointer Events unify all input into a single event stream with `pointerId` tracking, eliminating the dual-handler problem entirely.

**Alternatives considered**:
- **Keep separate mouse/touch handlers, fix stale state**: Would require a gesture state machine on top of the existing split handlers — more complexity for the same result.
- **Third-party gesture library (Hammer.js, etc.)**: Adds a dependency, violates constitution (vendored deps, minimal abstraction). Pointer Events are natively supported in all target browsers.

## Decision 2: Gesture State Machine

**Decision**: Implement a finite state machine with explicit states: `IDLE → PINCHING | PANNING | SWIPING_NAV | SWIPING_DISMISS`. Transitions occur on pointer events with guards based on touch count and current zoom level.

**Rationale**: The current code uses boolean flags (`isPinching`, `isPanning`, `_didDrag`) that can get out of sync. A proper FSM ensures only one gesture mode is active at any time, and state resets are explicit on pointer-up events.

**State transitions**:
- `IDLE` + 2 pointers → `PINCHING`
- `IDLE` + 1 pointer + zoom > 1x → `PANNING`
- `IDLE` + 1 pointer horizontal + zoom = 1x → `SWIPING_NAV`
- `IDLE` + 1 pointer vertical down + zoom = 1x → `SWIPING_DISMISS`
- Any state + all pointers up → `IDLE` (with cleanup)

## Decision 3: Expand-from-Thumbnail Animation

**Decision**: Use CSS transforms with `getBoundingClientRect()` to animate the photo from its source thumbnail position to its final viewer position.

**Rationale**: The constitution mandates "CSS transitions preferred over JavaScript animation." The approach:
1. On open: capture source element's bounding rect, create the viewer image at the source rect using CSS `transform`, then transition to the final centered position.
2. On close: reverse the animation back to the source rect (if still visible) or fade out.

This matches iPhone Photos and Google Photos behavior. Using `will-change: transform` and GPU-composited properties (`transform`, `opacity`) ensures 60fps on mobile.

**Alternatives considered**:
- **Web Animations API**: More control but not necessary for a single transform+opacity transition.
- **FLIP technique (First-Last-Invert-Play)**: This IS the FLIP technique — we're using it with CSS transitions rather than JS-driven frame updates.
- **View Transitions API**: Not yet supported in all target browsers (iOS Safari support is limited).

## Decision 4: Module Architecture

**Decision**: Extract the photo viewer into a new `js/photo-viewer.js` ES2020+ module with a companion `css/photo-viewer.css` stylesheet.

**Rationale**: The current lightbox implementation is ~420 lines of JS embedded in the index.html inline script block, tightly coupled to global state. Extracting it follows the established pattern of `js/auth.js` and `js/cloud-data.js` — ES2020+ modules that expose functionality via `window.*`. This makes the code testable, maintainable, and keeps index.html focused on wiring.

**Module interface**:
```javascript
window.photoViewer = {
  open(photos, startIndex, sourceElement),  // Open viewer with a photo set
  close(),                                   // Close viewer
  isOpen()                                   // Check if viewer is currently open
};
```

The `photos` parameter accepts any array of photo objects, enabling context-aware navigation: pass `filteredPhotos` from map clicks, or `dateIndex[date].photos` from feed clicks.

## Decision 5: Progressive Loading Strategy

**Decision**: Keep the existing thumbnail-first approach with these fixes:
1. Use a single `<img>` element that starts with the thumbnail `src`, then swaps to `web_url` on load — eliminating layout shift by never changing element dimensions.
2. Apply CSS `opacity` crossfade on the swap rather than `filter: blur()` removal.
3. For videos: use the thumbnail URL as the `poster` attribute on the `<video>` element, with explicit `width`/`height` matching the container to prevent aspect ratio distortion.
4. Cancel in-flight `Image()` preloads by setting `src = ''` when navigating away.

**Rationale**: The current progressive loading mostly works but has two bugs:
- Video entries use the same thumbnail loading path as photos, causing stretched/distorted thumbnails when the thumbnail aspect ratio doesn't match the video container constraints.
- No cancellation of preloads means navigating rapidly queues up stale high-res loads that consume bandwidth.

## Decision 6: Swipe vs Zoom Disambiguation

**Decision**: Use a 10px movement threshold with angle detection:
- Horizontal movement (angle < 30° from horizontal) at 1x zoom → navigation swipe
- Vertical downward movement (angle > 60° from horizontal) at 1x zoom → dismiss gesture
- Any movement at zoom > 1x → panning
- Two-pointer movement → pinch-zoom (regardless of zoom level)

**Rationale**: iPhone Photos uses a similar approach. The threshold prevents accidental gesture activation. The angle thresholds are generous enough to feel natural but distinct enough to prevent ambiguity.

## Decision 7: UI Controls Auto-Hide

**Decision**:
- **Mobile**: Controls hidden by default. Single tap toggles visibility. Controls auto-hide after 3 seconds of no interaction.
- **Desktop**: Controls visible on mouse movement. Auto-hide after 2 seconds of no movement. Navigation arrows visible only on hover near edges.

**Rationale**: Matches iPhone Photos (mobile) and Google Photos (desktop) conventions. The constitution requires "chrome and controls MUST be minimal and unobtrusive, appearing only when needed."

## Decision 8: Video Handling Simplification

**Decision**: All videos are self-hosted .mov files served via Firebase/Google Drive. Remove all iframe/embedded video code paths. Use native `<video>` element with:
- `poster` attribute set to the video's thumbnail URL
- `preload="none"` (don't preload video data)
- `playsinline` attribute for iOS inline playback
- `controls` attribute for native playback UI

**Rationale**: Per clarification, no embedded/iframe videos exist. The `.mov` files are accessible via `web_url` (Google Drive) or direct URL. Simplifying to a single video path eliminates the iframe code and the distortion bug (which came from applying image-style loading to iframe containers).
