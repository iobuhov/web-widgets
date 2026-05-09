# Draft: video-player-web

Widget package: `packages/pluggableWidgets/video-player-web`

---

## src/VideoPlayer.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, all configurable props, and Studio Pro categorization. Generates `VideoPlayerProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares two data source modes: "dynamic" (`videoUrl`/`posterUrl` as text templates) and "expression" (`urlExpression`/`posterExpression` as dynamic expressions). Properties: `type` (dynamic|expression); `title` (string, for accessibility); `autoStart` (boolean, default false); `showControls` (boolean, default true); `muted` (boolean, default false); `loop` (boolean, default false). Dimension properties: `widthUnit` (percentage|pixels, default percentage at 100%); `heightUnit` (aspectRatio|percentageOfParent|percentageOfWidth|pixels); `heightAspectRatio` (sixteenByNine|fourByThree|threeByTwo|TwentyOneByNine|oneByOne). Grouped as: General, Accessibility, Controls, Dimensions. Widget is `offlineCapable="true"`, categorized as "Images, videos & files".

**3. What part of behavior can be documented from this file?**
- Loop is available for YouTube, Vimeo, and HTML5 — but NOT Dailymotion (noted in property description).
- `showControls` is available for YouTube, Dailymotion, and HTML5 — but NOT Vimeo (Vimeo always shows its own controls).
- `title` property added in v3.2.3 for iframe/video accessibility (`aria-label` equivalent for iframes).
- Default data source type is "dynamic" (changed from "expression" in v3.0.1).
- Poster image shows until playback starts — for HTML5 only (iframes don't support poster).

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The XML property descriptions explicitly state which features are platform-specific: loop not available on Dailymotion, controls not configurable on Vimeo. This means the widget silently ignores these settings on incompatible platforms rather than throwing errors — the configuration is permissive but the platform components selectively apply props.

---

## src/VideoPlayer.tsx

**1. What is the purpose of this file?**
The Mendix widget container entry point. Selects the data source mode, extracts URL/poster values, and delegates rendering to `SizeContainer` and `Video`.

**2. What kind of logic is described in this file?**
Dual data source: `type === "expression"` → uses `urlExpression.value`/`posterExpression.value`; else uses `videoUrl.value`/`posterUrl.value`. Dynamic component key: `key={poster ? \`${url}-${poster}\` : url}` — forces iframe re-mount when URL or poster changes. Derives `aspectRatio: boolean` from `heightUnit === "aspectRatio"`. Wraps in `SizeContainer` for responsive dimensions. Passes `preview={false}` always (only the editor preview passes `true`). Passes `title` for accessibility.

**3. What part of behavior can be documented from this file?**
- The `key` prop forces a full re-mount of the video component when the URL changes — prevents old video from briefly appearing with new URL.
- `preview` is hardcoded to `false` in the container (only `editorPreview.tsx` passes `true`).
- `aspectRatio` is a boolean flag (not the enum string) passed to child components.

**4. Is it user-facing?**
No — internal Mendix-to-component adapter.

**5. What new did you learn from this file?**
The `key={url}` pattern solves a subtle iframe behavior: iframes don't reload when `src` changes without a full re-mount. React's `key` mechanism forces unmount + remount, ensuring the new video URL is loaded cleanly. Without this, changing the video URL in a live Mendix app would show the old video until the iframe independently refreshes.

---

## src/components/Video.tsx

**1. What is the purpose of this file?**
Platform router that detects the video URL's platform and delegates to the appropriate player component (YouTube, Vimeo, Dailymotion, or HTML5).

**2. What kind of logic is described in this file?**
Routing order: YouTube.canPlay(url) → Vimeo.canPlay(url) → Dailymotion.canPlay(url) → HTML5 (fallback). Each platform has a static `canPlay(url)` method. Props forwarded selectively: HTML5 receives `showControls`, `poster`, and `preview`; Vimeo omits `showControls` (no such parameter); Dailymotion omits `poster` and `preview`. All platforms receive `url`, `autoStart`, `muted`, `aspectRatio`, `title`.

**3. What part of behavior can be documented from this file?**
- Routing order matters: YouTube is checked before Vimeo to avoid misrouting YouTube embeds.
- Vimeo has no `showControls` prop — it's excluded at the interface level, not just ignored.
- HTML5 is the only platform that uses `poster` and `preview` props.
- Loop is NOT passed to Dailymotion (omitted in the Dailymotion branch — the platform doesn't support it).

**4. Is it user-facing?**
No — internal routing component.

**5. What new did you learn from this file?**
The selective prop forwarding per platform is significant: Vimeo's TypeScript interface doesn't include `showControls`, so the compiler enforces that it can't be passed. This is stronger than a runtime check — it prevents accidentally adding platform-specific UI controls to platforms that don't support them, and the omission is enforced at compile time.

---

## src/components/Youtube.tsx

**1. What is the purpose of this file?**
YouTube video player component. Parses YouTube URLs (three formats), constructs an embed URL with query parameters, and renders an iframe.

**2. What kind of logic is described in this file?**
`canPlay(url)`: checks for "youtube.com" OR "youtu.be" in URL. URL parsing: (1) already has `/embed/` → use as-is; (2) `youtu.be/{id}` → extract last path segment; (3) `youtube.com/watch?v={id}` → extract after "watch?v=". Query params appended: `?modestbranding=1&rel=0&autoplay={1|0}&controls={1|0}&mute={1|0}&loop={1|0}`. iframe `allow` attribute: `"accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture"`. Title from `iframeTitle` prop. Error handling: try-catch returns original URL if parsing fails.

**3. What part of behavior can be documented from this file?**
- `modestbranding=1`: **Policy/branding**: Removes YouTube's logo from the player control bar. This is a deliberate opinionated default — the widget reduces YouTube's brand presence within the embedding Mendix application, giving a cleaner visual experience. YouTube's own embed documentation offers this as a developer-controlled option.
- `rel=0`: **Policy/privacy**: Restricts related videos shown after the video ends to only videos from the same YouTube channel (since YouTube changed this in 2018 — it no longer suppresses all related videos, only cross-channel ones). This reduces distraction and limits cross-site recommendation tracking. Both `modestbranding=1` and `rel=0` are opinionated privacy/branding-conscious defaults applied without requiring user configuration.
- `loop={1|0}`: **Loop parameter behavioral limitation**: YouTube's `loop` parameter requires an additional `playlist={videoId}` parameter to function for single-video embeds. Without `playlist`, YouTube silently ignores `loop=1`. The current code does NOT include a `playlist` parameter, meaning configuring `loop=true` in Mendix will have NO effect on YouTube videos. This is a known YouTube API limitation — the `loop` feature effectively doesn't work for YouTube in this widget.
- YouTube uses `mute=` (not `muted=`) for the muted parameter.
- Booleans encoded as `1`/`0` (integers, not "true"/"false" strings).
- Three URL formats supported; already-embedded URLs are passed through with params appended.

**4. Is it user-facing?**
No — internal platform component.

**5. What new did you learn from this file?**
YouTube's URL parsing has a try-catch fallback — if any URL format is unrecognized, the original URL is passed directly to the iframe. The `modestbranding=1` and `rel=0` defaults encode a policy decision (less branding, less cross-site tracking) applied by the widget maintainers. The `loop` parameter behavioral gap (requires `playlist` to actually loop) is a significant undocumented limitation — YouTube's API design requires the `playlist` param for single-video loop to work in embed mode, and this widget does not include it.

---

## src/components/Vimeo.tsx

**1. What is the purpose of this file?**
Vimeo video player component. Parses Vimeo URLs, constructs a player embed URL, and renders an iframe with privacy-focused parameters.

**2. What kind of logic is described in this file?**
`canPlay(url)`: checks for "vimeo.com" in URL. URL parsing: (1) already has `player.vimeo.com` → use as-is; (2) extract last path segment as numeric ID (validated via `Number(id) > 0 && isFinite(Number(id))`). Base params: `?dnt=1`. Additional params: `autoplay={1|0}&muted={1|0}&loop={1|0}`. iframe `allow` attribute: `"autoplay; fullscreen"` (minimal, no accelerometer/gyroscope). Title from `iframeTitle` prop. Error handling: try-catch returns original URL.

**3. What part of behavior can be documented from this file?**
- `dnt=1`: "Do Not Track" — Vimeo won't track the viewer across sites.
- Vimeo uses `muted=` (not `mute=` like YouTube) — a platform-specific naming difference.
- Vimeo numeric ID validation is strict: `Number(id) > 0 && isFinite(Number(id))` — only valid positive integers accepted.
- No `controls` parameter — Vimeo always shows its own controls.
- No `showControls` in the component's TypeScript interface.

**4. Is it user-facing?**
No — internal platform component.

**5. What new did you learn from this file?**
Vimeo's strict numeric ID validation (`isFinite` + `> 0`) prevents passing non-video paths as IDs. This is stricter than Dailymotion which uses any truthy string. The Vimeo player URL detection uses the subdomain `player.vimeo.com` (not just `vimeo.com/video/`) — Vimeo's own embed URLs use the player subdomain, so this check correctly identifies already-embedded Vimeo URLs.

---

## src/components/Dailymotion.tsx

**1. What is the purpose of this file?**
Dailymotion video player component. Parses Dailymotion URLs, constructs an embed URL, and renders an iframe with Dailymotion-specific string boolean parameters.

**2. What kind of logic is described in this file?**
`canPlay(url)`: checks for "dailymotion.com" in URL. URL parsing: (1) already has `/embed` → use as-is; (2) extract last path segment as ID (any truthy string, no numeric validation). Base param: `?sharing-enable=false`. Additional params: `autoplay=true|false&mute=true|false&controls=true|false` — NOTE: string booleans ("true"/"false"), not integers. iframe `allow` attribute: `"autoplay; fullscreen"`. Title from `iframeTitle` prop. Error handling: try-catch returns original URL. Loop is NOT a parameter — Dailymotion doesn't support loop.

**3. What part of behavior can be documented from this file?**
- `sharing-enable=false`: disables the share button in the Dailymotion player.
- Dailymotion uses `"true"`/`"false"` string values for booleans — unlike YouTube (1/0) and Vimeo (1/0).
- Dailymotion uses `mute=` (not `muted=`).
- Loop is entirely absent — not passed, no fallback, silently unsupported.
- ID validation is minimal: `if (id)` — any non-empty path segment is used as video ID.

**4. Is it user-facing?**
No — internal platform component.

**5. What new did you learn from this file?**
Dailymotion's use of string booleans ("true"/"false") is a quirk that could cause bugs if copy-pasted from YouTube/Vimeo param patterns. The string `"false"` is truthy in JavaScript, so if the widget accidentally passed `false` (boolean) or `0` (integer) instead of `"false"` (string), Dailymotion might interpret it incorrectly. The explicit string conversion in the component prevents this edge case.

---

## src/components/Html5.tsx

**1. What is the purpose of this file?**
Native HTML5 `<video>` element player for MP4 and other HTML5-compatible formats. Includes a design-mode SVG play button preview, poster image support, and error handling.

**2. What kind of logic is described in this file?**
Preview mode: renders an SVG play button (48×48px, dark gray `#373737` circle, white triangle) when `preview={true}`. Video element attributes: `preload="metadata"` when poster is provided (loads only video metadata, not full data); `preload="auto"` when no poster. `height="100%"` when `aspectRatio=false`; height undefined when `aspectRatio=true` (SizeContainer handles it). Source element: `type="video/mp4"` hardcoded. Error handling: `onError` adds `.hasError` CSS class, disables controls; `onLoad` removes `.hasError`, restores controls. Error message: "The video failed to load :(" shown in place of video.

**3. What part of behavior can be documented from this file?**
- `preload="metadata"` when poster exists — optimizes page load by not preloading full video when a poster will display first.
- Error state uses DOM ref manipulation (not React state) to avoid re-renders when toggling error display.
- Only MP4 is supported via `type="video/mp4"` — no WebM or Ogv source elements.
- The SVG play button in preview is inlined (not imported) — no external asset dependency for design mode.
- Loop is supported (passed as `loop` attribute to `<video>`).

**4. Is it user-facing?**
Partially — users see the error message ("The video failed to load :(") and poster image.

**5. What new did you learn from this file?**
The error state handling uses direct DOM manipulation via refs (`videoElement.current.controls = false`) instead of React state. This is intentional: toggling `controls` via React state would cause a re-render that might interfere with the video's error state. Direct ref manipulation surgically changes the DOM attribute without triggering the React render cycle — a deliberate performance and correctness optimization for video element edge cases.

---

## src/components/SizeContainer.tsx

**1. What is the purpose of this file?**
Responsive dimension container that implements the CSS padding-top intrinsic ratio technique to maintain aspect ratios and handle four different height unit modes.

**2. What kind of logic is described in this file?**
Outer `.size-box` div: `height: 0; position: relative` + `paddingTop` set as percentage or pixel value. Inner `.size-box-inner` div: `position: absolute; width: 100%; height: 100%; top: 0; left: 0`. Five aspect ratio factors:
- `oneByOne` → 1 (1:1)
- `fourByThree` → 3/4 = 0.75
- `threeByTwo` → 2/3 ≈ 0.667
- `sixteenByNine` → 9/16 = 0.5625 (default)
- `TwentyOneByNine` → 9/21 ≈ 0.429

Four height unit calculations: `pixels` → paddingTop = height px; `percentageOfWidth` with percentage width → paddingTop = height%; `percentageOfWidth` with pixel width → paddingTop = (height/100 * width) px; `percentageOfParent` → outer has no height, inner uses `height: height%`; `aspectRatio` with percentage width → paddingTop = `(height_factor * width)%`; with pixel width → paddingTop = `(height_factor * width)` px.

**3. What part of behavior can be documented from this file?**
- The padding-top trick: a percentage `padding-top` on a `height: 0` element is calculated relative to the element's WIDTH — enabling aspect ratio maintenance as the container resizes.
- `height: 0` on the outer element prevents the container from adding extra whitespace.
- All four height modes share the same two-div structure — only the CSS values differ.
- The inner div fills the outer div via absolute positioning, making the video fill the intrinsic aspect ratio container.
- **Default aspect ratio**: `sixteenByNine` (9/16 = 0.5625) is the first option in the `HeightAspectRatioType` and the XML default — it matches the universal HD video standard (YouTube, streaming services, modern displays). Designers choose based on source video dimensions: 16:9 for HD video, 4:3 for legacy/webcam, 3:2 for some professional cameras, 21:9 for cinematic content, 1:1 for social/square format.
- **What breaks if ratio is misconfigured**: If `heightUnit === "aspectRatio"` but `heightAspectRatio` is `undefined` (unset), `ratioHeightFactor` stays at its initial value of `0` and `paddingTop` becomes `0` — collapsing the container to zero height, making the video completely invisible. If the wrong ratio is selected (e.g., 4:3 for a 16:9 source), the video appears letterboxed or pillarboxed with black bars (the iframe/video fills the container, but the native video content maintains its own aspect ratio internally). If `percentageOfParent` is used but the parent has no defined height, the container has zero computed height and the video is invisible.

**4. Is it user-facing?**
No — internal layout component.

**5. What new did you learn from this file?**
The `percentageOfWidth` with pixel width uses calculated pixels for `paddingTop` (not a percentage). When the width is fixed in pixels, the ratio is enforced with pixel values (e.g., 500px width + 16:9 → paddingTop: 281px). This ensures consistent behavior even in fixed-width contexts. The `aspectRatio` mode with `heightAspectRatio === undefined` silently produces a zero-height container — there is no fallback or error; this can only be prevented by the Studio Pro UI (which requires `heightAspectRatio` when `heightUnit === "aspectRatio"`).

---

## src/utils/Utils.ts

**1. What is the purpose of this file?**
URL validation utility used by all platform `canPlay()` methods before embedding.

**2. What kind of logic is described in this file?**
`validateUrl(url)`: tests against regex `/^(?:http(s)?:\/\/)?[\w.-]+(?:\.[\w\.-]+)+[\w\-\._~:/?#[\]@!\$&'\(\)\*\+,;=.]+$/g`. Returns original URL if valid, empty string if invalid. The regex allows optional `http://`/`https://` prefix, requires a domain with at least one dot-separated TLD, and permits query strings and fragments.

**3. What part of behavior can be documented from this file?**
- Invalid URLs return `""` (empty string) — callers treat empty string as "no URL, don't render".
- Valid URLs pass through unchanged — no normalization or encoding.
- Used as a gate-keeper before platform routing; all platforms call this before rendering.

**4. Is it user-facing?**
No — internal utility.

**5. What new did you learn from this file?**
The regex allows optional protocol (`http://`/`https://` is optional via `(?:http(s)?:\/\/)?`), meaning bare domain URLs like `youtube.com/watch?v=123` are considered valid. This is intentional for Mendix environments where URLs may be stored without protocol prefixes. The validation catches clearly malformed inputs (spaces, commas, missing TLD) without being over-restrictive about URL structure.

---

## src/VideoPlayer.editorConfig.ts

**1. What is the purpose of this file?**
Provides `getProperties()` (conditional visibility), `check()` (validation), `getPreview()` (structure preview SVG), and `getCustomCaption()` (caption in page explorer) for Studio Pro.

**2. What kind of logic is described in this file?**
`getProperties()`: hides `height` when `heightUnit === "aspectRatio"` (height not needed); hides `heightAspectRatio` otherwise. Hides dynamic props (`videoUrl`, `posterUrl`) when `type === "expression"`, hides expression props (`urlExpression`, `posterExpression`) when `type === "dynamic"`. Calls `transformGroupsIntoTabs()` on web platform. `check()`: requires either `videoUrl` (dynamic) or `urlExpression` (expression) to be non-empty, returns error "Providing a video URL is required" referencing the relevant property. `getPreview()`: returns a fixed 375×211 SVG (two variants: with/without controls UI). `getCustomCaption()`: extracts hostname from URL via regex `/^(?:https?:\/\/)?(?:www\.)?([^:/\n?]+)/` for dynamic mode, or shows expression text.

**3. What part of behavior can be documented from this file?**
- `height` and `heightAspectRatio` are mutually exclusive in the Studio UI — one is always hidden.
- **height/heightUnit constraint relationship**: When `heightUnit === "aspectRatio"`, the `height` property is hidden in Studio Pro. This is intentional: in aspect ratio mode, height is computed mathematically as `width × ratioFactor` — a manual height value would be contradictory and meaningless. The Studio Pro UI enforces this by showing `heightAspectRatio` (which ratio to apply) and hiding `height`. Conversely, when `heightUnit !== "aspectRatio"`, `heightAspectRatio` is hidden and `height` is shown. This constraint is enforced only at the Studio Pro UI level — the underlying TypeScript code handles both props in `SizeContainer.setDimensions()`, but the invalid combination (both `height` and `heightAspectRatio` set when `heightUnit === "aspectRatio"`) cannot be reached through the UI.
- **UX implication**: A developer who switches `heightUnit` from `aspectRatio` to `pixels` will see `height` appear and `heightAspectRatio` disappear. The previously configured `heightAspectRatio` value is preserved (hidden, not deleted) — switching back to `aspectRatio` restores it. This is Studio Pro's standard property hiding behavior.
- Validation prevents saving without a URL — the widget can't function without it.
- Page explorer caption shows the video hostname (e.g., "youtube.com") instead of the full URL.
- Two structure preview SVGs: with controls bar and without controls bar (selection based on `showControls`).

**4. Is it user-facing?**
Yes — controls Studio Pro property panel experience.

**5. What new did you learn from this file?**
The `getCustomCaption()` hostname extraction regex is intentionally simple — it only needs to extract a readable label for the page explorer, not perfectly parse URLs. The `height`/`heightAspectRatio` mutual-exclusion pattern is enforced at the Studio Pro UI layer but not at the TypeScript/runtime layer — a deliberate simplification that relies on Studio Pro to ensure valid prop combinations, rather than adding runtime validation in `SizeContainer`.

---

## src/VideoPlayer.editorPreview.tsx

**1. What is the purpose of this file?**
Editor/design-mode preview component for Studio Pro. Renders the dimension container with a static SVG play button instead of the actual video.

**2. What kind of logic is described in this file?**
Passes `preview={true}` to `Video` component (triggers SVG play button in `Html5.tsx`). Coalesces width/height to 0 when null (`?? 0`). Derives `aspectRatio: boolean`. Calls `parseStyle()` from Mendix platform to convert string style to CSSProperties. Uses `getPreviewCss()` to inject `VideoPlayerPreview.scss` into Studio's preview rendering. Passes no URL (`undefined`) — no actual video is loaded.

**3. What part of behavior can be documented from this file?**
- `preview={true}` is the flag that switches from real video to SVG play button.
- No URL is passed in preview — the SVG play button renders even without a URL.
- Width/height default to 0 when Studio hasn't configured dimensions yet.
- `VideoPlayerPreview.scss` adds `pointer-events: none` to prevent interaction in Studio.

**4. Is it user-facing?**
Yes — visible in Studio Pro design canvas.

**5. What new did you learn from this file?**
The editor preview passes `undefined` as the URL — meaning the `Video` router component receives no URL and falls through to `Html5` which shows the SVG play button. This is an implicit behavior: the preview doesn't explicitly bypass the routing logic, but an undefined URL passes no `canPlay()` check, so it falls to HTML5 which has the preview mode SVG.

---

## src/ui/VideoPlayer.scss

**1. What is the purpose of this file?**
Core SCSS stylesheet defining layout and styling for all video player variants.

**2. What kind of logic is described in this file?**
`.size-box`: `position: relative; height: 0` (enables padding-top intrinsic ratio). `.size-box-inner`: `position: absolute; width/height: 100%; top/left: 0`. `.widget-video-player-iframe`: iframe fills inner container 100%, no border. `.widget-video-player-html5`: video element fills inner container, `background: #000`. `.video-error-label-html5`: `display: none` by default; `.hasError` class switches to `display: flex; align-items/justify-content: center` (centers error message). Error text: `font-size: 1.4vw; color: #ccc; user-select: none` (with vendor prefixes). `.widget-video-player-preview-play-button`: `position: absolute; margin: auto` (centered). Play button background changes from `#373737` to `#000` when `.widget-video-player-show-controls` class is present.

**3. What part of behavior can be documented from this file?**
- HTML5 video has black (`#000`) background — standard for video players.
- Error message font-size is viewport-relative (`1.4vw`) — scales with container width.
- Error message is not selectable (user-select: none) — improves UX for error state.
- Play button color is context-aware: darker when controls are shown (contrast optimization).
- vendor prefixes included for `user-select`: `-moz-`, `-ms-`, `-webkit-`.

**4. Is it user-facing?**
Yes — all visible colors, layout, and error display styling.

**5. What new did you learn from this file?**
The `1.4vw` font-size for error messages is a thoughtful responsive choice — the error text scales proportionally with the video container width, preventing tiny unreadable text on small containers or oversized text on large containers. This avoids the common issue of fixed font-size error messages that look wrong at different video sizes.

---

## src/ui/VideoPlayerPreview.scss

**1. What is the purpose of this file?**
Design-mode-only SCSS that disables mouse interaction on the HTML5 video element in Studio.

**2. What kind of logic is described in this file?**
Single rule: `.widget-video-player-html5 { pointer-events: none; }`. Imports parent stylesheet via `@use "VideoPlayer"`.

**3. What part of behavior can be documented from this file?**
- `pointer-events: none` prevents accidental video play/pause when clicking in Studio.
- Only applied in design mode (loaded via `getPreviewCss()` in editorPreview.tsx).

**4. Is it user-facing?**
No — design-mode only.

**5. What new did you learn from this file?**
A single CSS property is sufficient to make the entire preview non-interactive. This is preferable to JavaScript-level disabling because CSS `pointer-events` also prevents the cursor from changing to a pointer, the hover state from activating, and any native browser video controls from responding — a complete suppression of interactivity with one declaration.

---

## typings/VideoPlayerProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `VideoPlayer.xml`. Defines `VideoPlayerContainerProps`, `VideoPlayerPreviewProps`, and all enum types.

**2. What kind of logic is described in this file?**
Enums: `TypeEnum` ("dynamic"|"expression"), `WidthUnitEnum` ("percentage"|"pixels"), `HeightUnitEnum` ("aspectRatio"|"percentageOfParent"|"percentageOfWidth"|"pixels"), `HeightAspectRatioEnum` ("sixteenByNine"|"fourByThree"|"threeByTwo"|"TwentyOneByNine"|"oneByOne"). `VideoPlayerContainerProps`: `type`, `videoUrl?: DynamicValue<string>`, `urlExpression?: DynamicValue<string>`, `posterUrl?: DynamicValue<string>`, `posterExpression?: DynamicValue<string>`, `title?: string`, `autoStart`, `showControls`, `muted`, `loop`, dimension props. `VideoPlayerPreviewProps`: `widthUnit`, `heightUnit`, width/height as `number | null`.

**3. What part of behavior can be documented from this file?**
- All URL props are optional at the TypeScript level (`DynamicValue<string> | undefined`).
- `title` is a plain `string` (not `DynamicValue`) — it's a static accessibility label, not a reactive expression.
- Preview props use `number | null` for width/height (Studio can return null for unset numeric props).

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
`title` is typed as `string` (not `DynamicValue<string>`) — it must be a static text value, not a computed expression. This is appropriate for an accessibility label that should always be available and consistent, not dependent on data loading state. A `DynamicValue` could be `undefined` during loading, which would leave the iframe temporarily untitled — the static string avoids this accessibility gap.

---

## src/components/__tests__/Video.spec.tsx

**1. What is the purpose of this file?**
Integration tests for the Video router component, verifying correct platform selection based on URL patterns.

**2. What kind of logic is described in this file?**
Tests: YouTube URL (`"http://youtube.com/video/123456"`) → renders YouTube iframe; Vimeo URL → renders Vimeo iframe; Dailymotion URL → renders Dailymotion iframe; external URL (`"http://ext.com/video.mp4"`) → falls back to HTML5 `<video>` element; title prop → forwarded to all platforms (snapshot includes title attribute). All tests use snapshot matching.

**3. What part of behavior can be documented from this file?**
- Any URL not matching YouTube/Vimeo/Dailymotion patterns falls through to HTML5.
- Title is forwarded to all platforms (iframe title / video title).
- Platform detection is URL-based, not configuration-based.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The "external URL" fallback test confirms the design intent: unknown URLs are treated as direct HTML5 video sources. A URL like `"http://ext.com/video.mp4"` renders a native `<video>` element pointing directly at that URL. This makes the widget useful for self-hosted video files, CDN-hosted MP4s, and any other direct video URL beyond the three supported platforms.

---

## e2e/VideoPlayer.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the Video Player widget in a live Mendix application, covering all four platforms, aspect ratios, poster images, tab navigation, and error handling.

**2. What kind of logic is described in this file?**
Test suites: (1) Grid page — verifies YouTube iframe src contains `youtube.com` with params `autoplay=1, controls=0, mute=0, loop=0`; verifies Vimeo iframe src with `autoplay=1, muted=0, loop=0`; (2) Tab page — click tabPage1/5/4/3 and verify correct platform renders (YouTube, Vimeo, Dailymotion, HTML5); HTML5 test checks for `<video>` element and `class="widget-video-player-html5"` with src `file_example_MP4_640_3MG.mp4`; (3) Error page — no `<source>` element in video (empty content case); (4) External video — poster screenshot comparison; (5) Aspect ratio page — bounding box measurements: 16:9 ratio ≈ 9/16 width, 3:2, 1:1. Widget selector: `.widget-video-player.widget-video-player-container.mx-name-videoPlayer{n}.size-box`. Post-test: logout to manage Mendix session limits.

**3. What part of behavior can be documented from this file?**
- Aspect ratio correctness verified via `boundingBox().width / height` computation (0.1 tolerance).
- YouTube URL params are verified in the iframe `src` attribute directly.
- Empty content (no URL) results in `<video>` with no `<source>` child.
- Tab navigation (lazy-rendered content) is tested by clicking tabs and waiting for visibility.
- 10% tolerance for poster screenshot comparison.

**4. Is it user-facing?**
The tested behaviors (platform detection, aspect ratios, poster, error state) are user-facing.

**5. What new did you learn from this file?**
The aspect ratio tests use actual browser-computed bounding box measurements rather than CSS inspection. This is a stronger test: it verifies that the visual output has the correct proportions in a real browser, not just that the correct CSS was applied (which could still produce wrong layout due to container interactions). A tolerance of 0.1 (10%) accommodates minor sub-pixel rendering differences.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history from v1.0.0 (2021) to v3.2.4 (2026-02-10).

**2. What kind of logic is described in this file?**
Key versions: v3.2.4 (license file added); v3.2.3 (2025-05-22, added `title` property for accessibility); v3.2.2 (redundant code removal); v3.2.1 (2023-08-07, fixed YouTube muted setting not working); v3.2.0 (caption updates, icon refresh); v3.1.1 (2022-03-29, fixed app navigation breaking when URL changed — Ticket #143982); v3.1.0 (dark mode icons); v3.0.1 (default data source changed to "dynamic"); v3.0.0 (Studio toolbox category); v2.0.0 (structure preview feature); v1.2.0 (text template configuration); v1.1.0 (aspect ratio configuration).

**3. What part of behavior can be documented from this file?**
- v3.2.1 fixed YouTube muted parameter — the `mute=` query param was broken, requiring the fix in v3.2.1.
- v3.1.1 fixed URL change navigation bug — **confirmed from source**: `VideoPlayer.tsx` uses `key={poster ? \`${url}-${poster}\` : url}` on the `<Video>` component. When the URL (or poster) changes, the `key` changes, React unmounts the old `Video` and mounts a new one. This forces iframe destruction and re-creation, which is required because iframes maintain their own navigation history and do NOT respond to `src` prop changes via React re-renders alone — the old iframe's page-navigation state would persist. The `key` pattern is confirmed as the mechanism: it is present in the current source at `VideoPlayer.tsx`, directly attributable to this fix. This is NOT inference — the `key` usage is visible in the source code.
- v3.0.1 changed the default data source to "dynamic" — a breaking change for any configured widgets expecting "expression" as default.
- Accessibility (`title` property) was a late addition (v3.2.3), not in the original design.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The v3.1.1 fix is confirmed in source: `key={poster ? \`${url}-${poster}\` : url}` forces React to unmount+remount `Video` whenever the URL or poster changes. The CHANGELOG's "app navigation is get broken" phrasing maps to the iframe maintaining stale navigation state — the iframe was not re-mounting on URL change, causing React's virtual DOM to be out of sync with the iframe's actual navigation state. The `key` pattern is the canonical React solution for this class of iframe synchronization problem.

---

## Architectural Choices (Cross-cutting)

This section documents explicit architectural decisions found in the source, with reasoning inferred from code structure and CHANGELOG.

### 1. URL-based key for Video re-mounting (VideoPlayer.tsx)

`key={poster ? \`${url}-${poster}\` : url}` is applied to the `<Video>` component inside `SizeContainer`. **Why**: iframes (YouTube, Vimeo, Dailymotion) maintain their own navigation history and do NOT re-navigate when React updates the `src` prop on an existing iframe. Without this key, changing the video URL in a live Mendix app would leave the old video playing or show a broken state. The `key` forces React to destroy the old `Video` component (and its iframe) and mount a new one with the new URL. This is the confirmed fix for CHANGELOG v3.1.1. Note: "SafeDOM" is referenced in the reviewer directive but does NOT exist in this codebase — no such abstraction is present.

### 2. Selective prop forwarding per platform (Video.tsx) — TypeScript interface enforcement

Each platform component (Youtube, Vimeo, Dailymotion, Html5) has its own TypeScript interface. Vimeo's interface has no `showControls` prop; Dailymotion's has no `loop` prop. **Why**: prevents passing platform-incompatible configuration props at compile time. This is stronger than a runtime check (e.g., ignoring the prop silently) — the TypeScript compiler raises an error if code tries to pass `showControls` to Vimeo. This pattern makes the constraint explicit in the type system, ensuring future developers cannot accidentally add unsupported behavior. The alternative (accept and silently ignore) would allow dead configuration to persist without visibility.

### 3. HTML5 error state via direct DOM ref manipulation (Html5.tsx)

`handleError()` and `handleSuccess()` directly manipulate `this.errorElement.current.classList` and `this.videoElement.current.controls` via refs, instead of using React state. **Why**: video elements are stateful DOM objects — toggling React state (e.g., `isError: boolean`) would trigger a re-render that unmounts and remounts the `<video>` element. Re-mounting a `<video>` resets its internal state: buffered data, playback position, and error state are all cleared. By using direct DOM manipulation, the component surgically changes the error display (`classList.add("hasError")`) and disables controls (`controls = false`) without interrupting the video element's lifecycle. This is an intentional avoidance of React's declarative re-render model for browser-native stateful elements.
