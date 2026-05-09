# VideoPlayer

## Purpose

The VideoPlayer widget embeds videos from YouTube, Vimeo, Dailymotion, and any direct HTML5-compatible URL (MP4) within a Mendix page. Platform detection is URL-based: the widget examines the configured URL and routes to the appropriate player component (YouTube iframe, Vimeo iframe, Dailymotion iframe, or native `<video>` element). Dimensions are managed by a responsive `SizeContainer` that supports fixed pixels, percentage of parent, percentage of width, and intrinsic aspect ratio modes. The widget is categorized under "Images, videos & files" and supports offline use.

## User Scenarios

### [P1] Embed a YouTube video
**Given** a VideoPlayer with a YouTube URL (e.g., `https://www.youtube.com/watch?v=XXXXX`)  
**When** the page renders  
**Then** a YouTube iframe is rendered with `modestbranding=1` (reduced YouTube branding) and `rel=0` (related videos restricted to same channel)  

#### Edge Cases
- Three YouTube URL formats are recognized: standard watch URL, short `youtu.be` URL, and existing embed URLs. Unrecognized formats fall back to passing the original URL to the iframe.
- `loop = true` has no effect for YouTube single-video embeds because the YouTube API requires an additional `playlist={videoId}` parameter that the widget does not include.
- The `key` prop on the Video component forces a full React unmount/remount when the URL changes, ensuring iframes reload correctly (iframes do not re-navigate on React `src` prop changes alone).

### [P2] Embed a Vimeo video
**Given** a VideoPlayer with a Vimeo URL  
**When** the page renders  
**Then** a Vimeo iframe is rendered with `dnt=1` (Do Not Track privacy parameter); Vimeo controls are always visible regardless of the `showControls` setting (Vimeo always shows its own controls)  

### [P3] Embed an HTML5 video (direct URL)
**Given** a VideoPlayer with an MP4 URL that does not match YouTube, Vimeo, or Dailymotion  
**When** the page renders  
**Then** a native `<video>` element is rendered with type `"video/mp4"`, optionally with a poster image and native browser controls  
**When** the video fails to load  
**Then** an error message "The video failed to load :(" is displayed centered in the container; native controls are disabled on error  

#### Edge Cases
- `preload="metadata"` is set when a poster image is configured (loads only metadata, not full video); `preload="auto"` when no poster.
- Loop, poster, controls, autostart, and muted are all supported for HTML5.

### [P4] Aspect ratio sizing
**Given** a VideoPlayer with `heightUnit = "aspectRatio"` and `heightAspectRatio = "sixteenByNine"` (default)  
**When** the container width changes (responsive layout)  
**Then** the video height maintains the 16:9 ratio using the CSS padding-top intrinsic ratio technique  

#### Edge Cases
- If `heightAspectRatio` is not configured when `heightUnit = "aspectRatio"`, the container collapses to zero height and the video is invisible.
- If the wrong ratio is selected (e.g., 4:3 for a 16:9 source), black bars appear (the iframe fills the container but the video maintains its own intrinsic ratio internally).

## Functional Requirements

- FR-001: Platform detection MUST follow this order: YouTube → Vimeo → Dailymotion → HTML5 (fallback). URLs not matching any platform MUST use the native `<video>` element.
- FR-002: All platform URLs MUST be validated via `validateUrl()` before rendering; invalid URLs MUST produce no rendered video element (empty string treated as "no URL").
- FR-003: The Video component MUST have a `key` equal to `url` (or `${url}-${poster}` when a poster is set) to force re-mount when the URL or poster changes.
- FR-004: YouTube MUST append `modestbranding=1&rel=0` to all embed URLs; booleans (autoplay, controls, mute, loop) MUST be encoded as `1`/`0` integers.
- FR-005: Vimeo MUST append `dnt=1` to all embed URLs; booleans MUST be encoded as `1`/`0` integers; `showControls` MUST NOT be passed to Vimeo (enforced at the TypeScript interface level).
- FR-006: Dailymotion MUST append `sharing-enable=false` to all embed URLs; booleans MUST be encoded as `"true"`/`"false"` strings; `loop` MUST NOT be passed to Dailymotion.
- FR-007: HTML5 player MUST use `type="video/mp4"` on the `<source>` element; error state MUST be handled via direct DOM ref manipulation (not React state re-render, to preserve video element lifecycle).
- FR-008: Dimension container MUST implement the CSS padding-top intrinsic ratio technique for aspect ratio mode, supporting five ratios: 16:9, 4:3, 3:2, 21:9, 1:1.
- FR-009: Studio Pro MUST hide `height` when `heightUnit = "aspectRatio"` and MUST hide `heightAspectRatio` otherwise.
- FR-010: A URL MUST be required — Studio Pro MUST show a validation error if no URL is configured.
- FR-011: The `title` property provides an accessibility label for the iframe/video element; it MUST be a static string (not a dynamic expression) to ensure it is always available.
- FR-012: Two data source modes MUST be supported: `"dynamic"` (text template) and `"expression"` (DynamicValue); default is `"dynamic"`.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `type` | `"dynamic"` \| `"expression"` | `"dynamic"` | Data source type | Source mode for the video URL |
| `videoUrl` | `DynamicValue<string>` (optional) | — | Video URL | Text template URL (dynamic mode) |
| `urlExpression` | `DynamicValue<string>` (optional) | — | URL expression | Expression URL (expression mode) |
| `posterUrl` | `DynamicValue<string>` (optional) | — | Poster URL | HTML5 only: image shown before playback starts (text template, dynamic mode) |
| `posterExpression` | `DynamicValue<string>` (optional) | — | Poster expression | HTML5 only: poster image URL (expression mode) |
| `title` | `string` (optional) | — | Title | Static accessibility label for the iframe title / video aria-label (added v3.2.3) |
| `autoStart` | boolean | `false` | Auto start | Start playback automatically on load |
| `showControls` | boolean | `true` | Show controls | Show native player controls; no effect on Vimeo |
| `muted` | boolean | `false` | Muted | Start playback muted |
| `loop` | boolean | `false` | Loop | Loop playback; no effect on Dailymotion; no effect on YouTube (missing `playlist` parameter) |
| `widthUnit` | `"percentage"` \| `"pixels"` | `"percentage"` (100%) | Width unit | |
| `heightUnit` | `"aspectRatio"` \| `"percentageOfParent"` \| `"percentageOfWidth"` \| `"pixels"` | — | Height unit | |
| `heightAspectRatio` | `"sixteenByNine"` \| `"fourByThree"` \| `"threeByTwo"` \| `"TwentyOneByNine"` \| `"oneByOne"` | `"sixteenByNine"` | Aspect ratio | Intrinsic aspect ratio; only relevant when `heightUnit = "aspectRatio"` |

## Platform-specific Behavior Summary

| Feature | YouTube | Vimeo | Dailymotion | HTML5 |
|---------|---------|-------|-------------|-------|
| `showControls` | ✓ | ✗ (always shows own controls) | ✓ | ✓ |
| `loop` | ✗ (requires `playlist` param) | ✓ | ✗ | ✓ |
| `poster` | ✗ | ✗ | ✗ | ✓ |
| Privacy defaults | `rel=0`, `modestbranding=1` | `dnt=1` | `sharing-enable=false` | N/A |

## Changelog

- **v3.2.4 (2026-02-10)**: License file added.
- **v3.2.3 (2025-05-22)**: Added `title` property for iframe/video accessibility.
- **v3.2.1 (2023-08-07)**: Fixed YouTube muted parameter not working.
- **v3.1.1 (2022-03-29)**: Fixed app navigation breaking when URL changed (added `key` prop for forced re-mount).
- **v3.0.1**: Changed default data source type from `"expression"` to `"dynamic"`.
- **v1.1.0**: Added aspect ratio configuration.
- **v1.0.0 (2021)**: Initial release.

## Open Questions

> Could not be determined from source code alone — requires human review
- [ ] YouTube `loop` parameter requires a `playlist={videoId}` parameter to function for single-video embeds. Should this be added in a future release, or should the `loop` prop description explicitly document this limitation to avoid developer confusion?
- [ ] Does the widget support WebM or Ogv HTML5 video formats? The current implementation hardcodes `type="video/mp4"` — other formats would need additional `<source>` elements.
- [ ] What happens when an invalid (but non-empty) URL is passed — e.g., a relative path? `validateUrl()` allows optional protocol, so this edge case may not be caught.
