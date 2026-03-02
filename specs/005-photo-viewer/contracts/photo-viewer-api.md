# Module Contract: Photo Viewer

**Feature Branch**: `005-photo-viewer` | **Date**: 2026-03-02

No REST/GraphQL APIs — this is a pure frontend module. This document defines the JavaScript module interface.

## Public API (`window.photoViewer`)

### `open(photos, startIndex, sourceElement)`

Opens the photo viewer with an expand-from-thumbnail animation.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `photos` | `Photo[]` | Yes | Array of photo objects to navigate through |
| `startIndex` | `number` | Yes | Index of the photo to display first (0-based) |
| `sourceElement` | `Element \| null` | No | DOM element to animate from (thumbnail). If null, uses a fade-in. |

**Behavior**:
- Captures `sourceElement.getBoundingClientRect()` for animation origin
- Creates/shows the viewer overlay
- Displays the thumbnail immediately, loads full-res in background
- Prevents background scrolling
- Binds keyboard listeners

**Preconditions**: `photos.length > 0`, `0 <= startIndex < photos.length`

### `close()`

Closes the viewer with a shrink-to-thumbnail animation (if source element is still in DOM) or fade-out.

**Behavior**:
- Cancels any in-flight image preloads
- Re-enables background scrolling
- Unbinds keyboard listeners
- Resets zoom/pan state

### `isOpen()`

Returns `boolean` — whether the viewer overlay is currently visible.

## Photo Object Shape (input)

The viewer expects each photo object to have these fields:

```javascript
{
  url: string,              // Stable ID path (e.g., "photos/20260129_091401.jpg")
  thumbnail: string,        // Low-res URL for instant display
  web_url: string,          // Full-res URL for progressive loading
  type: string,             // "photo" | "video"
  caption: string,          // Display text (may be empty)
  date: string,             // "YYYY-MM-DD"
  tags: string[],           // Tag labels
  google_photos_url: string,// Link to original (may be empty)
  lat: number,              // GPS latitude
  lng: number,              // GPS longitude
  _isFavorite: boolean      // Favorite status (runtime)
}
```

## Events (dispatched on `document`)

| Event Name | Detail | When |
|------------|--------|------|
| `photoviewer:open` | `{ index, photo }` | Viewer opened |
| `photoviewer:close` | `{}` | Viewer closed |
| `photoviewer:navigate` | `{ index, photo, direction }` | User navigated to a different photo |
| `photoviewer:favorite` | `{ photo }` | User toggled the favorite button |

These custom events allow `index.html` to react to viewer actions (e.g., rebuild photo layers after favorite toggle) without the viewer module needing to know about Leaflet, cloud data, or the map.

## Integration Points (index.html responsibilities)

| Action | index.html code |
|--------|----------------|
| Map marker click | Call `photoViewer.open(filteredPhotos, index, markerImgElement)` |
| Feed thumbnail click | Call `photoViewer.open(dateIndex[date].photos, index, thumbnailElement)` |
| Favorite toggle | Listen for `photoviewer:favorite`, call `toggleFavorite()` + `rebuildPhotoLayer()` |
| Caption/tag display | Viewer reads from photo object; cloud data merges happen before passing to viewer |
| Caption/tag editing | Viewer dispatches edit events; index.html handles Firestore writes |
