# Feature Specification: Fix Video Playback

**Feature Branch**: `011-fix-video-playback`
**Created**: 2026-03-07
**Status**: Draft
**Input**: User description: "The videos are all unviewable when clicked on. Player appears -> Photo unavailable"

## Clarifications

### Session 2026-03-07

- Q: How should videos be rendered in the viewer (iframe embed vs native `<video>` vs new tab)? → A: Embed as iframe using Google Drive's built-in player. Loses pinch-zoom on videos but retains swipe navigation; Google handles transcoding and cross-browser compatibility.
- Q: How should video thumbnails be visually distinguished from photo thumbnails? → A: Semi-transparent play-button icon overlay centered on the thumbnail.
- Q: What should the viewer display while the video iframe is loading? → A: Show the video's thumbnail as a static background with a centered loading spinner overlay.
- Q: How should swipe/touch gestures interact with the embedded iframe player? → A: Edge-zone overlays at left/right/top edges handle swipe navigation; center area passes touch events through to the iframe for native video controls.
- Q: What aspect ratio should the embedded video iframe use? → A: 16:9 letterboxed (black bars if needed), matching standard video dimensions.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Watch a Video from the Map (Priority: P1)

A visitor clicks on a video thumbnail (from the map, photo wall, or landing page). The photo viewer opens and the video plays inline. The user can play, pause, seek, and control volume using standard video controls.

**Why this priority**: This is the core bug — videos are completely broken. Fixing playback is the entire purpose of this feature.

**Independent Test**: Click any video entry on the map or photo wall and confirm the video loads and plays with working controls.

**Acceptance Scenarios**:

1. **Given** the viewer is closed, **When** the user clicks a video thumbnail, **Then** the viewer opens and displays the video with playback controls (play/pause, seek, volume)
2. **Given** a video is playing in the viewer, **When** the user clicks play, **Then** the video begins playback without errors
3. **Given** a video is displayed in the viewer, **When** the user navigates to the next/previous item, **Then** the current video stops and the next item loads correctly

---

### User Story 2 - Navigate Between Videos and Photos (Priority: P2)

A visitor is browsing through a mix of photos and videos using swipe gestures or arrow navigation. Transitioning between video and photo items works seamlessly — videos stop when navigated away from, and new videos load properly when navigated to.

**Why this priority**: Navigation already works for photos; ensuring it remains smooth when videos are interspersed is essential for a coherent browsing experience.

**Independent Test**: Open a photo adjacent to a video, then swipe/arrow to the video and confirm it loads. Navigate back to the photo and confirm the video stops cleanly.

**Acceptance Scenarios**:

1. **Given** a video is playing in the viewer, **When** the user swipes to the next photo, **Then** the video pauses/stops and the photo displays correctly
2. **Given** a photo is displayed in the viewer, **When** the user swipes to a video, **Then** the video loads and is ready to play

---

### Edge Cases

- What happens when the video source URL is unreachable or returns an error? The viewer should display a clear error message (not "Photo unavailable").
- What happens when the user closes the viewer while a video is playing? The video should stop and release resources.
- What happens on mobile browsers where autoplay may be restricted? The video should display its poster/thumbnail and wait for user interaction to play.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST render videos hosted on Google Drive as embedded players within the viewer (using the provider's preview/embed interface, not a native video element)
- **FR-002**: System MUST display standard playback controls (play/pause, seek bar, volume) for videos
- **FR-003**: System MUST stop any playing video when the user navigates away from it (next/prev or close)
- **FR-004**: System MUST show the video's thumbnail as a poster image before playback begins
- **FR-005**: System MUST display a meaningful error state when a video fails to load (distinct from the photo error message)
- **FR-005a**: System MUST show the video's thumbnail as a static background with a centered loading spinner overlay while the iframe is loading, transitioning to the embedded player once ready
- **FR-006**: System MUST correctly identify video items and render them differently from photo items based on the item's type field in the manifest
- **FR-007**: System MUST display a semi-transparent play-button icon overlay centered on video thumbnails wherever they appear (map markers, photo wall, landing page) to visually distinguish them from photo thumbnails
- **FR-008**: System MUST render the embedded video iframe at a 16:9 aspect ratio, letterboxed with black bars if the viewer dimensions do not match

### Key Entities

- **Video Entry**: A manifest item with `type: "video"`, containing a `web_url` (external provider embed/preview URL), `thumbnail` (poster image), and standard photo metadata (caption, date, tags, coordinates)

## Assumptions

- Video `web_url` values in the manifest point to Google Drive embed/preview URLs (not direct video file streams). Videos will be rendered via iframe embed using these URLs.
- Pinch-zoom gestures are not applicable to embedded video players. Swipe navigation uses transparent overlay zones at the left/right edges and top of the viewer; the center iframe area passes touch events through to the video player's native controls.
- Thumbnails for videos are already generated and hosted on Firebase Storage, same as photos.
- No changes to the manifest data format are required — only the viewer's rendering logic needs to change.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of video entries in the manifest are viewable when clicked (zero "Photo unavailable" errors for valid videos)
- **SC-002**: Users can begin video playback within 3 seconds of clicking a video thumbnail on a standard broadband connection
- **SC-003**: Navigating between photos and videos in the viewer works without errors or visual glitches
- **SC-004**: Video playback works on both desktop and mobile browsers (Chrome, Safari, Firefox)
