# video-player-web — Draft Spec

Widget: `video-player-web`
Package: `packages/pluggableWidgets/video-player-web/`
Agent: worker
Date: 2026-05-09

---

## src/VideoPlayer.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. Extracts URL and poster from either expression or text template props based on `type`, derives a `key` to force remount on URL change, and wraps everything in `<SizeContainer>` and `<Video>`.

**2. What kind of logic is described in this file?**
- `type === "expression"`: `url = urlExpression?.value`, `poster = posterExpression?.value`.
- `type === "dynamic"`: `url = videoUrl?.value`, `poster = posterUrl?.value`.
- `key = poster ? "${url}-${poster}" : url` — React key on `<Video>` forces full remount when URL or poster changes. This prevents stale video player state when the user navigates between pages or the URL attribute changes.
- `title = iframeTitle?.value` — optional iframe accessibility title.
- Class-based component (`extends Component`) — not a functional component.

**3. What part of behavior can be documented from this file?**
- Changing the video URL causes a full remount of the `<Video>` component (via `key` prop). Without this, HTML5 `<video>` elements don't update when `src` changes (browser caches the player state).
- Both `url` and `poster` together form the key — changing only the poster also remounts the player.
- `preview={false}` is always passed to `<Video>` in runtime (preview mode has its own component).

**4. Is it user-facing?**
No — internal Mendix adapter.

**5. What new did you learn from this file?**
The `key` prop pattern (v3.1.1 fix) reveals that HTML5 video players don't respond to `src` attribute changes without a remount — the browser's media element holds its own internal state. Using `key` is a clean React solution that forces the correct lifecycle (destroy + recreate) rather than trying to imperatively reset the video element.

---

## src/VideoPlayer.xml

**1. What is the purpose of this file?**
Mendix widget descriptor for Video Player — configures URL source, display controls, dimensions with aspect ratio support, and an accessibility title.

**2. What kind of logic is described in this file?**
Tabs: General, Accessibility, Controls, Dimensions.

General/Data source group:
- `type`: dynamic (text template, default) | expression (Mendix expression).
- `urlExpression`: String expression (for `expression` type).
- `posterExpression`: String expression (for `expression` type).
- `videoUrl`: text template (for `dynamic` type).
- `posterUrl`: text template (for `dynamic` type).

Accessibility group:
- `iframeTitle`: optional text template — describes purpose of the video (e.g., "Video tutorial on accessibility").

Controls group:
- `autoStart`: boolean (default: false).
- `showControls`: boolean (default: true) — for YouTube, Dailymotion, HTML5.
- `muted`: boolean (default: false).
- `loop`: boolean (default: false) — for YouTube, Vimeo, HTML5 (not Dailymotion).

Dimensions group:
- `widthUnit`: percentage (default) | pixels; `width`: default 100.
- `heightUnit`: aspectRatio (default) | percentageOfParent | percentageOfWidth | pixels; `height`: default 500.
- `heightAspectRatio`: sixteenByNine (default) | fourByThree | threeByTwo | TwentyOneByNine | oneByOne.

System properties: Name, TabIndex.

**3. What part of behavior can be documented from this file?**
- `needsEntityContext="true"`, `offlineCapable="true"`, `supportedPlatform="Web"`.
- Category: "Images, videos & files".
- Default height unit is `"aspectRatio"` with `"sixteenByNine"` — the widget defaults to a responsive 16:9 container.
- The poster URL is a preview image shown before the user starts the video.
- `showControls` description notes it's "Available for YouTube, Dailymotion and external videos" — not Vimeo (which doesn't expose a controls parameter in embed URLs).
- `loop` description notes it's "Available for YouTube, Vimeo, and external videos" — not Dailymotion.

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
The XML makes feature availability explicit in property descriptions: `showControls` says "not for Vimeo," `loop` says "not for Dailymotion." These constraints come from the iframe embed API limitations of each platform, not widget limitations — Vimeo's embed API doesn't expose a controls toggle, and Dailymotion's embed API doesn't support loop.

---

## typings/VideoPlayerProps.d.ts

Not read directly, but inferred from editorConfig.ts usage. Key types:
- `TypeEnum = "dynamic" | "expression"`.
- `HeightUnitEnum = "aspectRatio" | "percentageOfParent" | "percentageOfWidth" | "pixels"`.
- `HeightAspectRatioEnum = "sixteenByNine" | "fourByThree" | "threeByTwo" | "TwentyOneByNine" | "oneByOne"`.
- Container props have `class`, `style` — CSS customization supported.

---

## src/components/Video.tsx

**1. What is the purpose of this file?**
URL-based player dispatcher — determines which player component to use by checking each provider's `canPlay(url)` static method.

**2. What kind of logic is described in this file?**
Priority-ordered dispatch:
1. `Youtube.canPlay(url)` → renders `<Youtube>`.
2. `Vimeo.canPlay(url)` → renders `<Vimeo>`.
3. `Dailymotion.canPlay(url)` → renders `<Dailymotion>`.
4. Fallback → renders `<Html5>`.

Supported props vary by player:
- YouTube: `showControls`, `autoPlay`, `muted`, `loop`.
- Vimeo: `autoPlay`, `muted`, `loop` (no `showControls`).
- Dailymotion: `autoPlay`, `muted`, `showControls` (no `loop`).
- HTML5: all props plus `poster` and `preview`.

**3. What part of behavior can be documented from this file?**
- All unrecognized URLs fall through to HTML5 player — supports any direct video file URL.
- `preview` prop is only passed to `Html5` — the other providers don't need special preview handling (they're iframes that won't load in preview mode anyway).
- Class-based component — uses private bound methods for each render path.

**4. Is it user-facing?**
No — dispatcher.

**5. What new did you learn from this file?**
The `poster` prop is only passed to `Html5` — embedded iframe players (YouTube, Vimeo, Dailymotion) don't support external poster images. Each platform has its own thumbnail. The poster is exclusively for HTML5 direct video files.

---

## src/components/Html5.tsx

**1. What is the purpose of this file?**
Renders a native HTML5 `<video>` element for direct video file URLs (MP4, etc.) with error handling and preview support.

**2. What kind of logic is described in this file?**
DOM structure:
- `<div class="widget-video-player-html5-container [widget-video-player-show-controls]">`:
  - In preview mode: play button SVG (dark circle with white triangle).
  - In runtime: `<div class="video-error-label-html5">The video failed to load :(</div>` — initially hidden, shown on error.
  - `<video class="widget-video-player-html5" controls={showControls} autoPlay muted loop poster>`:
    - `height={!aspectRatio ? "100%" : undefined}` — no explicit height in aspect-ratio mode.
    - `preload={poster ? "metadata" : "auto"}` — metadata only if poster exists.
    - (runtime only) `<source src={url} type="video/mp4" onError={handleError} onLoad={handleSuccess} />`.

Error/success handling (class-based, `createRef`):
- `handleError()`: adds `"hasError"` class to error div (making it visible), sets `video.controls = false`.
- `handleSuccess()`: removes `"hasError"` class, restores `video.controls = showControls`.

**3. What part of behavior can be documented from this file?**
- Error message "The video failed to load :(" is always in DOM — controlled via CSS class `"hasError"`.
- When video load fails: controls are hidden (to avoid a broken player UI with controls but no video).
- Preview mode: no `<source>` element — prevents the browser from fetching the video URL in design mode. Shows a static play button SVG instead.
- `preload="metadata"` when poster is set — loads enough to get video duration/dimensions without downloading the full video.

**4. Is it user-facing?**
Yes — renders the visible video player for direct video URLs.

**5. What new did you learn from this file?**
The `onError`/`onLoad` events are on the `<source>` element (not the `<video>` element). The `<video>` element's `error` event doesn't fire for source errors in the same way. Using `<source onError>` is the correct pattern for detecting MP4 load failure when using multiple sources or a single source element.

---

## src/components/Youtube.tsx

**1. What is the purpose of this file?**
Renders a YouTube embed `<iframe>` with URL normalization and query parameter construction.

**2. What kind of logic is described in this file?**
URL normalization (three input formats):
- `youtube.com/embed/{id}` → appends attributes directly.
- `youtu.be/{id}` or `youtube.com/v/{id}` → extracts last path segment as video ID.
- `youtube.com/watch?v={id}` → splits on `watch?v=` and takes last part.

Query params: `?modestbranding=1&rel=0&autoplay={0|1}&controls={0|1}&mute={0|1}&loop={0|1}`.
- `modestbranding=1`: removes YouTube logo from player.
- `rel=0`: no related videos at end.
- `mute` (not `muted`) — YouTube's embed API uses `mute`.

`canPlay(url)`: URL contains `"youtube.com"` or `"youtu.be"` AND `validateUrl(url)` returns non-empty.

**3. What part of behavior can be documented from this file?**
- YouTube `muted` prop maps to `mute` param (not `muted`) in the iframe URL — this was fixed in v3.2.1.
- `modestbranding=1` and `rel=0` are always set — minimal YouTube branding, no related video suggestions.
- All URL formats accepted: embed URL, short URL (youtu.be), full watch URL, v-format URL.

**4. Is it user-facing?**
Yes — renders the visible YouTube video player.

**5. What new did you learn from this file?**
The URL normalization handles all common YouTube URL formats users might paste. The fallback (`return url`) at the end means if an already-embedded URL format is passed, it passes through unchanged — backward compatible with users who already use the embed URL format.

---

## src/components/Vimeo.tsx

**1. What is the purpose of this file?**
Renders a Vimeo embed `<iframe>` with URL normalization and query parameter construction.

**2. What kind of logic is described in this file?**
URL normalization:
- `player.vimeo.com` URLs → appends attributes directly.
- Other vimeo.com URLs → extracts last path segment; validates it's a finite positive number; constructs `https://player.vimeo.com/video/{id}`.

Query params: `?dnt=1&autoplay={0|1}&muted={0|1}&loop={0|1}`.
- `dnt=1`: "Do Not Track" — privacy parameter.
- `muted` (not `mute`) — Vimeo's API uses `muted`.

No `showControls` prop — Vimeo's embed API doesn't expose a controls toggle.

`canPlay(url)`: URL contains `"vimeo.com"` AND `validateUrl(url)` returns non-empty.

**3. What part of behavior can be documented from this file?**
- Vimeo uses `muted` (not `mute`) — different from YouTube's `mute` parameter.
- `dnt=1` is always set — Vimeo's Do Not Track flag, prevents Vimeo from tracking user data via the embed.
- Vimeo numeric ID validation: `Number(id) > 0 && isFinite(Number(id))` — ensures the URL path ends with a valid Vimeo video ID.

**4. Is it user-facing?**
Yes — renders the visible Vimeo video player.

**5. What new did you learn from this file?**
`dnt=1` is always enabled — Vimeo's Do Not Track parameter. This is a privacy-conscious default that prevents Vimeo from tracking widget users via the embedded player. This may be important for GDPR compliance in apps.

---

## src/components/Dailymotion.tsx

**1. What is the purpose of this file?**
Renders a Dailymotion embed `<iframe>` with URL normalization and query parameter construction.

**2. What kind of logic is described in this file?**
URL normalization:
- `dailymotion.com/embed` URLs → appends attributes directly.
- Other dailymotion.com URLs → extracts last path segment as video ID.

Query params: `?sharing-enable=false&autoplay={true|false}&mute={true|false}&controls={true|false}`.
- Note: string `"true"/"false"` (not `"1"/"0"` like YouTube/HTML5).
- `sharing-enable=false`: disables sharing button.

No `loop` prop — Dailymotion's embed API doesn't support loop.

`canPlay(url)`: URL contains `"dailymotion.com"` AND `validateUrl(url)` returns non-empty.

**3. What part of behavior can be documented from this file?**
- `sharing-enable=false` is always set — removes the sharing button from the Dailymotion player.
- Uses string "true"/"false" not "1"/"0" — Dailymotion's embed API differs from YouTube.
- Dailymotion doesn't support `loop` — this is a platform limitation reflected in the component's props interface.

**4. Is it user-facing?**
Yes — renders the visible Dailymotion video player.

**5. What new did you learn from this file?**
Dailymotion's embed API uses string boolean values (`"true"/"false"`) while YouTube uses integer (`"1"/"0"`). This is an API inconsistency between platforms. The widget handles this correctly per-platform in each component.

---

## src/components/SizeContainer.tsx

**1. What is the purpose of this file?**
Handles all dimension calculations for the video player container, translating widget configuration to CSS styles.

**2. What kind of logic is described in this file?**
`setDimensions()` returns a `CSSProperties` object:
- Width: `widthUnit === "percentage"` → `${width}%`; else `${width}px`.
- Height via `paddingTop` trick (zero-height div + padding-top creates height proportional to width):
  - `"pixels"`: `paddingTop: height`.
  - `"percentageOfWidth"`: `paddingTop: ${(height/100)*width}%` (or pixel value if fixed width).
  - `"percentageOfParent"`: `height: ${height}%` (actual height, no padding trick).
  - `"aspectRatio"`: `paddingTop: ${width * ratio}%` where ratio = height/width for the selected aspect.
    - 16:9 → `9/16 = 0.5625`
    - 4:3 → `3/4 = 0.75`
    - 3:2 → `2/3 ≈ 0.6667`
    - 21:9 → `9/21 ≈ 0.4286`
    - 1:1 → `1`

DOM: `<div class="size-box" style={dimensions}><div class="size-box-inner">{children}</div></div>`.

**3. What part of behavior can be documented from this file?**
- The `paddingTop` percentage trick is relative to the element's **width** — this is a well-known CSS technique for responsive video embeds.
- `percentageOfParent` uses actual `height` property (not padding) — suitable for full-height containers.
- The `size-box-inner` div holds the actual video content — the outer div provides the sized box via padding.
- `heightUnit === "aspectRatio"` and `widthUnit === "pixels"`: `paddingTop` is a pixel value (e.g., `width: 640px` → `paddingTop: 360`).

**4. Is it user-facing?**
No — container component.

**5. What new did you learn from this file?**
The aspect ratio implementation uses `paddingTop` rather than the modern `aspect-ratio` CSS property. This is a classic responsive embed technique (predating CSS `aspect-ratio`) that works by setting `height: 0` on the outer div and using `padding-top` percentage to create a box with intrinsic dimensions based on its width. The inner div is positioned absolutely within this box (or fills it via `height: 100%`).

---

## src/utils/Utils.ts

**1. What is the purpose of this file?**
Validates URLs to prevent non-URL strings from being passed to video player components.

**2. What kind of logic is described in this file?**
`validateUrl(url)`: regex `/^(?:http(s)?:\/\/)?[\w.-]+(?:\.[\w\.-]+)+[\w\-\._~:/?#[\]@!\$&'\(\)\*\+,;=.]+$/g.test(url)` — returns `url` if valid, `""` if invalid.

**3. What part of behavior can be documented from this file?**
- Called by each provider's `canPlay` method — ensures only valid URLs trigger provider-specific behavior.
- Allows HTTP and HTTPS URLs; requires domain-like structure.
- Returns empty string (falsy) for invalid URLs — `canPlay` returns `false`, player falls back to HTML5 (which handles the empty URL gracefully).

**4. Is it user-facing?**
No — utility function.

**5. What new did you learn from this file?**
`validateUrl` uses a global regex flag (`/g`) — this can cause stateful behavior in JavaScript (`lastIndex` tracking) if called in a loop. Each call with a new string resets `lastIndex` due to `test()` being called on a regex literal each function call, so it works correctly here. The validation is intentionally permissive — it accepts any URL-like string, not just video platform URLs.

---

## src/VideoPlayer.editorConfig.ts

**1. What is the purpose of this file?**
Studio Pro property visibility, validation, structure preview, and custom caption for the Video Player widget.

**2. What kind of logic is described in this file?**
`getProperties`:
- `heightUnit === "aspectRatio"` → hides `height`; else hides `heightAspectRatio`.
- `type === "dynamic"` → hides `urlExpression`, `posterExpression`.
- `type === "expression"` → hides `videoUrl`, `posterUrl`.
- `platform === "web"` → `transformGroupsIntoTabs`.

`check` validation:
- `type === "dynamic" && !videoUrl` → error on `videoUrl`: "Providing a video URL is required".
- `type === "expression" && !urlExpression` → error on `urlExpression`: "Providing a video URL is required".

`getPreview`: Returns SVG image — `StructurePreviewWithControlsSVG` or `StructurePreviewWithoutControlsSVG` based on `showControls`. Fixed size (375×211px).

`getCustomCaption`:
- `type === "dynamic"`: extracts hostname from `videoUrl` using naive regex (`/^(?:https?:\/\/)?(?:www\.)?([^:/\n?]+)/`); falls back to full URL.
- `type === "expression"`: returns the expression string.
- Default: "Video Player".

**3. What part of behavior can be documented from this file?**
- `heightAspectRatio` is hidden when not using aspect ratio height mode — prevents orphaned property.
- Both URL formats require a URL — no "optional" URL scenario.
- Custom caption shows the domain name (not full URL) — e.g., "youtube.com" instead of the full embed URL.

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
The `getCustomCaption` uses a regex instead of `new URL()` — the comment explains this: `new URL` doesn't work in the Studio Pro preview environment. This is a known limitation of the editor config context (no browser APIs), requiring a manual regex for URL parsing.

---

## src/VideoPlayer.editorPreview.tsx

**1. What is the purpose of this file?**
Live React preview in Studio Pro design mode — renders the player in preview mode (no actual video loading).

**2. What kind of logic is described in this file?**
- Class-based `preview` component.
- Renders `<SizeContainer>` + `<Video preview={true}>` — no URL passed (undefined).
- `width ?? 0` and `height ?? 0` — defensive against null in preview context.
- `getPreviewCss()` exports `VideoPlayerPreview.scss` for design mode styling.
- No URL is passed to `<Video>` — with `preview={true}`, `Html5` renders a play button SVG and no `<source>`.

**3. What part of behavior can be documented from this file?**
- Design mode shows a play button SVG placeholder inside the sized container.
- The aspect ratio/dimensions are accurately reflected in design mode.
- No video content is loaded (no network requests in design mode).

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
With `preview={true}` and no URL, `Video.render()` falls through to `Html5` (since no provider's `canPlay("")` returns true), and `Html5` renders the SVG play button. This means the preview always shows the HTML5 play button style regardless of what video platform would be used at runtime — YouTube/Vimeo/Dailymotion icons are not shown in design mode.

---

## e2e/VideoPlayer.spec.js

**1. What is the purpose of this file?**
Playwright E2E tests verifying iframe src URLs, video element rendering, and aspect ratio calculations.

**2. What kind of logic is described in this file?**
Grid page tests:
- YouTube: iframe src contains `"youtube.com"`, `"autoplay=1"`, `"controls=0"`, `"mute=0"`, `"loop=0"`.
- Vimeo: iframe src contains `"vimeo.com"`, `"autoplay=1"`, `"muted=0"`, `"loop=0"` — note `muted` not `mute`.

Tab page tests (each provider on separate tab):
- YouTube, Vimeo, Dailymotion: iframe presence confirmed.
- HTML5: `<video class="widget-video-player-html5">` confirmed; source URL contains `"file_example_MP4_640_3MG.mp4"`.

Error page test: empty URL → no visible `<source>` element.

External video test: poster image screenshot (1% threshold).

Aspect ratio test: `boundingBox()` width/height ratio:
- 16:9 confirmed with `toBeCloseTo(16/9, 0.1)`.
- 3:2 confirmed.
- 1:1: `width === height`.

**3. What part of behavior can be documented from this file?**
- Confirmed: YouTube embed uses `mute` (not `muted`) in URL; Vimeo uses `muted` (not `mute`).
- Confirmed: `autoplay=1` for configured auto-start.
- Confirmed: HTML5 video source type is always `video/mp4` (regardless of file extension).
- Aspect ratio is tested by comparing actual bounding box dimensions — confirms the `paddingTop` technique produces correct proportions.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The E2E tests verify the actual iframe `src` attribute values — this is a strong behavioral test since it confirms the URL generation logic in each provider component. The `muted` vs `mute` discrepancy between Vimeo and YouTube is explicitly captured in the test expectations, documenting platform-specific API differences.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history from v1.1.0 to v3.2.4.

**2. What kind of logic is described in this file?**
- **v3.2.3 (2025-05-22)**: Added `iframeTitle` accessibility property for embedded video descriptions.
- **v3.2.2 (2023-09-27)**: Removed redundant code, improved load time.
- **v3.2.1 (2023-08-07)**: Fixed `muted` not working for YouTube — was using `muted=` but YouTube requires `mute=`.
- **v3.2.0 (2023-06-06)**: Updated page explorer caption (datasource display); updated icons/tiles.
- **v3.1.1 (2022-03-29)**: Fixed app navigation breaking when URL changes (→ `key` prop on `<Video>`).
- **v3.1.0 (2021-12-23)**: Added dark icons.
- **v3.0.0 (2021-09-28)**: Added toolbox category/tile.
- **v2.0.0 (2021-07-12)**: Added structure preview.
- **v1.2.0 (2021-07-12)**: Added text template support (the `"dynamic"` type).
- **v1.1.0 (2021-07-02)**: Added aspect ratio configuration.

**3. What part of behavior can be documented from this file?**
- v3.2.1 YouTube muted fix: the parameter name matters — `muted=1` was silently ignored by YouTube's embed API, requiring `mute=1`.
- v3.1.1 navigation fix: changing the URL triggered React component mismatch causing Mendix navigation to break — `key` prop forces proper remount.
- v1.2.0: text template support added separately from the original expression support — the two URL input types (`"dynamic"` and `"expression"`) were introduced at different times.

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
The navigation-breaking bug (v3.1.1) happened because changing the video URL without remounting the player caused React to patch the DOM in ways that conflicted with Mendix's navigation system. This explains why the `key` prop is derived from both URL and poster — it's a deliberate defensive measure against these lifecycle issues, not just about video state refresh.

---

## Summary of Key Findings

- **Purpose**: Embeds video players for YouTube, Vimeo, Dailymotion, and direct HTML5 video files. Auto-detects platform from URL and renders the appropriate embed.
- **Player dispatch**: `Video` component tries `canPlay(url)` for each provider in order (YouTube → Vimeo → Dailymotion → HTML5 fallback). All providers use `validateUrl` to avoid matching non-URL strings.
- **URL types**: `"dynamic"` (text template) | `"expression"` (Mendix expression). Same props, different Mendix binding types.
- **HTML5 player** (`Html5`): Native `<video>` with `<source type="video/mp4">`. Error handling via `onError`/`onLoad` on `<source>`. Preview shows SVG play button with no network requests.
- **YouTube** (`Youtube`): iframe embed with URL normalization (watch, embed, short, v-format). Params: `mute` (not `muted`), `modestbranding=1`, `rel=0`.
- **Vimeo** (`Vimeo`): iframe embed. Params: `muted` (not `mute`), `dnt=1` (Do Not Track always on). No `showControls` support.
- **Dailymotion** (`Dailymotion`): iframe embed. Params use string `"true"/"false"`. `sharing-enable=false` always set. No `loop` support.
- **Dimension system** (`SizeContainer`): `paddingTop` trick for responsive aspect ratios. Five aspect ratio presets: 16:9, 4:3, 3:2, 21:9, 1:1. Default: 16:9 aspect ratio.
- **Key remount**: `key={url + poster}` on `<Video>` — forces remount when URL changes, preventing stale player state and app navigation issues.
- **Accessibility**: Optional `iframeTitle` text template (added v3.2.3) — sets `title` attribute on all iframes and `<video>`.
- **Preview mode**: In design mode, always shows HTML5 play button SVG; no network requests; aspect ratio/dimensions are accurate.
- **offlineCapable**: `true` (though streaming video requires internet).
- **CSS customization**: Supports `class` and `style` props.
- **Testing**: E2E tests verify iframe src URL attributes, aspect ratio via `boundingBox()` dimensions, poster screenshot. No unit tests for the main widget components (only snapshot tests in component subdirectory).
- **Platform API differences**: YouTube (`mute`), Vimeo (`muted`, `dnt=1`), Dailymotion (string booleans, no loop) — each platform has distinct embed parameter naming conventions.
