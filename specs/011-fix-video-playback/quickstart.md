# Quickstart: Fix Video Playback

**Branch**: `011-fix-video-playback` | **Date**: 2026-03-07

## Setup

```bash
git checkout 011-fix-video-playback
python3 -m http.server 8000
```

Open `http://localhost:8000` in a browser.

## Files to Modify

1. **`js/photo-viewer.js`** — `renderVideo()` function (~line 512-532)
   - Add 16:9 wrapper container with thumbnail background + loading spinner
   - Create iframe with `opacity: 0`, fade in on `onload`
   - Add edge-zone swipe overlay divs (left/right/top) for touch navigation
   - Increase load timeout from 10s to 15s
   - Update `finalize()` to set `iframe.src = 'about:blank'` before removal

2. **`css/photo-viewer.css`** — Video-specific styles (~line 363+)
   - Add `.pv-video-wrap` — 16:9 letterbox container, centered, thumbnail background
   - Add `.pv-video-spinner` — CSS-only loading spinner
   - Add `.pv-iframe` — opacity transition for fade-in
   - Add `.pv-swipe-zone` — transparent edge overlays for touch navigation

3. **`js/landing-page.js`** — Detail grid thumbnails (~line 256)
   - Wrap video thumbnails with play-button icon overlay
   - Check `photos[p].type === 'video'` in the grid rendering loop

## Verification

1. Navigate to a region with videos (e.g., Copenhagen — videos from 2026-01-30)
2. Click a video thumbnail on the map or photo wall
3. Confirm: thumbnail with spinner shows while loading
4. Confirm: video fades in with Google Drive's player controls
5. Confirm: play/pause/seek/volume work (tap center of video)
6. Confirm: swipe left/right at edges navigates to adjacent items
7. Confirm: video maintains 16:9 aspect ratio with black letterbox bars
8. Confirm: closing viewer stops the video
9. Confirm: landing page detail grid shows play icon on video thumbnails
10. Test on mobile viewport (375px width)

## Video Entries for Testing

Videos are in the manifest with filenames like `IMG_7419.MOV`, `IMG_7422.MOV`, `IMG_7427.MOV` (dates around 2026-01-30, Copenhagen region).
