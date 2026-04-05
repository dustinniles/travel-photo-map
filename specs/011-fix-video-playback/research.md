# Research: Fix Video Playback

**Branch**: `011-fix-video-playback` | **Date**: 2026-03-07

## R1: Google Drive Iframe Embed Behavior

**Decision**: Use `<iframe>` with the existing `/preview` URLs from the manifest.

**Rationale**: Video `web_url` values already use the format `https://drive.google.com/file/d/{ID}/preview`, which is Google Drive's designated iframe embed URL. This renders a full video player with play/pause, seek, volume, and fullscreen controls — all handled by Google. No URL transformation needed.

**Alternatives considered**:
- Native `<video>` with direct download URL (`/uc?export=download`): Unreliable — Drive blocks large file downloads without confirmation, and CORS restrictions prevent cross-origin streaming.
- Opening Drive URL in new tab: Breaks the immersive viewer experience (violates Constitution VI).

## R2: 16:9 Letterboxed Iframe Sizing

**Decision**: Wrap the iframe in a container constrained to 16:9 aspect ratio using `max-width: calc((100vh - chrome) * 16/9)` and `max-height: calc(100vw * 9/16)`, centered with flexbox. The `.pv-media` background is `#000` for natural letterbox bars.

**Rationale**: A 16:9 constraint ensures the video player fills available space while maintaining a standard aspect ratio. Black bars appear naturally from the dark background. This matches user expectations from YouTube/Vimeo embeds and was confirmed as the desired behavior in spec clarification.

**Alternatives considered**:
- Let iframe fill 100%/100% with Drive handling its own letterboxing internally: Less predictable — iframe may render at unexpected sizes on different screens.
- `aspect-ratio` CSS property: Not supported in Safari <15, which is still in our target browsers.
- Fixed pixel dimensions: Doesn't adapt to screen sizes.

## R3: Edge-Zone Swipe Overlays for Touch Navigation

**Decision**: Transparent overlay zones at left/right edges (~60px wide) and top edge (~50px tall) capture swipe gestures for next/prev navigation and swipe-to-dismiss. The center area passes touch events through to the iframe for native video controls.

**Rationale**: Iframes create a cross-origin boundary — the parent page cannot listen for or intercept touch events inside the iframe. Edge zones provide reliable swipe targets without blocking video controls. Arrow buttons and keyboard navigation work regardless (they're outside the iframe).

**Alternatives considered**:
- Full overlay with tap-to-toggle mode: Confusing UX, not intuitive for family members.
- Disabling swipe entirely for videos: Breaks navigation consistency between photos and videos.
- Using `postMessage` API: Cross-origin Google Drive iframe won't cooperate.

**Implementation notes**:
- Edge zones use `pointer-events: auto` with `touch-action: pan-x pan-y`.
- The existing swipe handler in photo-viewer.js attaches to these zones when a video is displayed.
- Zones are invisible (transparent) but provide a visual hint on hover/touch (subtle gradient).

## R4: Loading State with Thumbnail + Spinner

**Decision**: Show the video's thumbnail as `background-image` on a wrapper div with a CSS-only spinner overlay. When `iframe.onload` fires, fade in the iframe (opacity transition) and hide the spinner.

**Rationale**: The thumbnail is already available in every manifest entry. Using it as a background-image means the iframe can be layered on top and faded in, creating a smooth transition without layout shifts. This addresses FR-005a.

**Implementation notes**:
- Wrapper div: `position: relative; background: url(thumbnail) center/contain no-repeat; background-color: #000`
- Spinner: CSS-only `@keyframes` rotation, absolutely positioned center, white semi-transparent
- Iframe starts with `opacity: 0; transition: opacity 300ms`
- On `iframe.onload`: add class that sets `opacity: 1`, hide spinner
- If timeout (15s) fires before `onload`, show "Video unavailable" error

## R5: Iframe Cleanup on Navigation/Close

**Decision**: On navigation or close, set `iframe.src = 'about:blank'` before removing from DOM, then clear `$media.innerHTML`.

**Rationale**: Simply removing the iframe from DOM should stop playback, but some browsers may continue loading in background if the iframe was mid-request. Setting src to blank first ensures clean teardown. The existing `finalize()` function already handles this pattern.

**Alternatives considered**:
- Using `iframe.contentWindow.postMessage()` to pause: Not possible — Drive iframe is cross-origin.

## R6: Landing Page Video Thumbnails

**Decision**: Add play-button overlay to video thumbnails in the landing page detail grid, matching the existing photo-wall pattern.

**Rationale**: Photo-wall (line 773) and map markers (Leaflet.Photo line 113) already show play icons. The landing page detail grid is the only rendering context missing this visual distinction. Adding it fulfills FR-007.

**Implementation notes**:
- In `landing-page.js` around line 256, check `photos[p].type === 'video'` and wrap the thumbnail in a container with a play icon overlay.
- Reuse the same visual style: semi-transparent dark overlay, centered white triangle/play symbol.

## R7: Current Bug Root Cause

**Decision**: The "Photo unavailable" error for videos has two causes:
1. The `errPlaceholder()` previously showed "Photo unavailable" for all media types (now fixed in working tree to differentiate).
2. The 10-second iframe load timeout is too aggressive for Google Drive embeds, especially on mobile — increase to 15 seconds.

**Rationale**: The code path from type detection (line 475) through `renderVideo()` (line 512) is structurally correct. The iframe creation and `web_url` usage are sound. The issue is the timeout racing against slow Google Drive responses plus the non-descriptive error message.
