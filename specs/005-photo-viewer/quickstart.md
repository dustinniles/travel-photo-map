# Quickstart: Immersive Photo Viewer

**Feature Branch**: `005-photo-viewer` | **Date**: 2026-03-02

## Prerequisites

- Python 3.10+ (for local server)
- A modern browser (Chrome, Safari, Firefox — recent versions)
- No build tools, npm, or package managers required

## Setup

```bash
# Clone and checkout branch
git checkout 005-photo-viewer

# Start local server
python3 -m http.server 8000

# Open in browser
# Desktop: http://localhost:8000
# Mobile: http://<your-local-ip>:8000 (same WiFi network)
```

## New Files

| File | Purpose |
|------|---------|
| `js/photo-viewer.js` | Photo viewer ES2020+ module — gesture handling, progressive loading, animations |
| `css/photo-viewer.css` | Photo viewer styles — overlay, transitions, controls, responsive layout |

## Modified Files

| File | Changes |
|------|---------|
| `index.html` | Remove old lightbox code (~lines 217-674). Add `<script>` and `<link>` for new module. Update `onPhotoClick` and `onFeedThumbnailClick` to call `window.photoViewer.open()`. |
| `css/map.css` | Remove `.lightbox-*` rules (~lines 644-836). |

## Testing Checklist

### Mobile (use Chrome DevTools device emulation or real device)

1. Tap a photo on the map → viewer expands from thumbnail
2. Swipe left/right → navigates photos
3. Pinch zoom in → photo zooms, release → stays zoomed (NOT stuck)
4. Pan while zoomed → image moves
5. Pinch back to 1x → photo fits screen
6. Swipe down at 1x → viewer dismisses
7. Tap screen → controls toggle
8. Open a video → clean thumbnail, correct aspect ratio, plays on tap

### Desktop

1. Click a photo → viewer expands from thumbnail
2. Hover edges → nav arrows appear, click → navigates
3. Arrow keys → navigates
4. Scroll wheel → zooms at cursor
5. Double-click → toggles 1x/2.5x zoom
6. Escape or click backdrop → closes
7. Favorite button → toggles star
8. Open from trip feed → navigates that day's photos only

### Progressive Loading

1. Throttle network to Slow 3G in DevTools
2. Open a photo → thumbnail appears instantly (< 100ms)
3. Full-res fades in when loaded (no pop-in or layout shift)
4. Navigate to next → already loaded (preloaded)

## Architecture Notes

The viewer is a standalone module that:
- Accepts a photo array + index + source element on `open()`
- Manages its own DOM, events, and state internally
- Exposes `window.photoViewer` for integration with the app
- Uses Pointer Events API for unified gesture handling
- Uses CSS transforms for all animations (GPU-composited)
