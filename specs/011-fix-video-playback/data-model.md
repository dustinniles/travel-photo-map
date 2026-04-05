# Data Model: Fix Video Playback

**Branch**: `011-fix-video-playback` | **Date**: 2026-03-07

## No Data Model Changes

This feature is a rendering-only bug fix. The manifest data format (`data/manifest.json`) is unchanged.

### Existing Video Entry (reference only)

```json
{
  "lat": 55.620736,
  "lng": 12.564889,
  "url": "photos/IMG_7419.MOV",
  "thumbnail": "https://firebasestorage.googleapis.com/v0/b/.../thumbs%2FIMG_7419.jpg?alt=media",
  "caption": "",
  "date": "2026-01-30",
  "datetime": "2026-01-30T15:01:35",
  "tags": [],
  "google_photos_url": "https://drive.google.com/file/d/{ID}/view?usp=drivesdk",
  "web_url": "https://drive.google.com/file/d/{ID}/preview",
  "type": "video"
}
```

**Key fields for this fix**:
- `type: "video"` — Discriminator used by viewer, photo-wall, map markers, and landing page to render video-specific UI (iframe player, play icon overlay)
- `web_url` — Google Drive `/preview` URL used as iframe `src` for embedded playback
- `thumbnail` — Firebase Storage URL for the video's still frame, used as loading state background and in all thumbnail contexts
