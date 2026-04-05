# Contract Update: Photo Viewer — Video Rendering

**Branch**: `011-fix-video-playback` | **Date**: 2026-03-07
**Base contract**: `specs/005-photo-viewer/contracts/photo-viewer-api.md`

## Changes to `renderVideo(p)` Behavior

### Before (broken)

Creates a `<video>` element with `src = p.web_url || p.url`. Fails because `web_url` is a Google Drive HTML preview page, not a video stream.

### After (fixed)

Creates an `<iframe>` element with `src = p.web_url`. The iframe renders Google Drive's embedded video player. Falls back to error state if `web_url` is missing.

### New DOM Structure (video items)

```html
<!-- Inside .pv-media when type === "video" -->
<iframe class="pv-iframe"
        src="https://drive.google.com/file/d/{ID}/preview"
        allow="autoplay; encrypted-media"
        allowfullscreen
        frameborder="0">
</iframe>
```

### CSS Additions

- `.pv-iframe`: Full container sizing (`width: 100%; height: 100%`), `border: none`, `pointer-events: auto`
- `.pv-media iframe` added to selectors alongside `img, video` where applicable

### Behavioral Changes

| Behavior | Photos | Videos (before) | Videos (after) |
|----------|--------|-----------------|----------------|
| Rendering | `<img>` | `<video>` (broken) | `<iframe>` |
| Zoom/pan | Yes | Yes (broken) | No (safe no-op) |
| Swipe nav | Yes | Yes | Yes |
| Swipe dismiss | Yes | Yes | Yes |
| Playback controls | N/A | Native `<video>` | Google Drive player |
| Cleanup on nav | Cancel preload | `video.pause()` | `iframe.src = 'about:blank'` |
| Error state | "Photo unavailable" | "Photo unavailable" | "Video unavailable" |

### Public API — No Changes

`window.photoViewer.open()`, `.close()`, `.isOpen()` signatures unchanged.
Photo object shape unchanged. Events unchanged.

### Integration Points — No Changes

Callers (map markers, photo wall, landing page) do not need updates. The viewer internally detects `type === "video"` and renders accordingly.
