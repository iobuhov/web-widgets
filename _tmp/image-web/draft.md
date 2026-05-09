# Draft: image-web

EX-033 — Worker first run, 2026-05-09.

---

## src/Image.xml

**1. What is the purpose of this file?**
Declares the widget contract for Mendix Studio/Studio Pro: ID, name, offline capability, all configurable properties grouped into "General" (Data source, Events, Accessibility, Conditional Visibility) and "Dimensions" tabs.

**2. What kind of logic is described in this file?**
Property definitions only — no runtime logic. Enumerations, default values, captions, and descriptions for every prop are defined here. The widget is `offlineCapable="true"` and does not require entity context (`needsEntityContext="false"`).

**3. What part of behavior can be documented from this file?**
Three image types are defined: `image` (from Mendix image object), `imageUrl` (external URL via textTemplate), and `icon` (Mendix WebIcon). Two on-click types: `action` (executes a Mendix action) and `enlarge` (opens lightbox). The `isBackgroundImage` toggle renders the widget as a CSS background with a content dropzone. Height units include `auto`, `pixels`, `percentage`, and `viewport` (vh). Min/max height are only available when heightUnit is `auto`. `responsive` constrains image to its natural size (never grows larger).

**4. Is it user-facing?**
Yes — the XML defines all Studio-visible properties that end users configure.

**5. What new did you learn from this file?**
The widget is in the "Images, videos & files" Studio Pro category. `defaultImageDynamic` is a fallback shown when no image is uploaded to the entity. `iconSize` defaults to 14px and is only visible for glyph/icon type. `displayAs` (fullImage/thumbnail) is only available for the `image` datasource.

---

## typings/ImageProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript interfaces derived from Image.xml; provides type-safe prop contracts for the container and preview components.

**2. What kind of logic is described in this file?**
Type declarations only. Two interfaces: `ImageContainerProps` (runtime, with `DynamicValue<WebImage>`, `DynamicValue<string>`, `DynamicValue<WebIcon>`, `ActionValue`) and `ImagePreviewProps` (design-time, with static/dynamic union types for imageObject/defaultImageDynamic and icon union types).

**3. What part of behavior can be documented from this file?**
`imageObject` and `defaultImageDynamic` are typed as `DynamicValue<WebImage>` — both support static and dynamic entity image references. `imageUrl` is `DynamicValue<string>` (textTemplate). `imageIcon` is `DynamicValue<WebIcon>` covering glyph, image, and icon sub-types. `children` is `ReactNode` — only visible when `isBackgroundImage` is true. `alternativeText` is `DynamicValue<string>`, optional.

**4. Is it user-facing?**
No — internal type contract; not directly user-facing.

**5. What new did you learn from this file?**
The `displayAs` prop is typed as `"fullImage" | "thumbnail"`. `HeightUnitEnum` includes `"viewport"` which maps to vh. `minHeightUnit` and `maxHeightUnit` both include `"none"` as an option (suppresses the dimension field).

---

## src/Image.tsx (main container)

**1. What is the purpose of this file?**
The Mendix pluggable widget entry point. Reads runtime prop values, resolves the correct image source, constructs style objects, and renders the inner Image component.

**2. What kind of logic is described in this file?**
`getImageProps` is a switch on `datasource`: for `"image"` it reads `imageObject.value.uri` when available, falls back to `defaultImageDynamic.value.uri` when imageObject is `ValueStatus.Unavailable`. For `"imageUrl"` reads the textTemplate value. For `"icon"` reads the WebIcon, distinguishing `"image"` (iconUrl) vs. glyph/icon (iconClass) sub-types. The container merges the Mendix-provided `style` with a computed `styleObject` only for the `"image"` type. `isBackgroundImage` is only active when datasource is not `"icon"`.

**3. What part of behavior can be documented from this file?**
Fallback chain: `imageObject` → `defaultImageDynamic` (only when imageObject is Unavailable, not just empty). Style computation (`constructStyleObject`) is skipped for icon types. `renderAsBackground` is set to `false` for icon datasource regardless of the `isBackgroundImage` prop value — icons cannot be background images.

**4. Is it user-facing?**
Indirectly — this is the runtime component users interact with.

**5. What new did you learn from this file?**
The fallback to `defaultImageDynamic` is specifically triggered by `ValueStatus.Unavailable` (not just `undefined`). The `onClick` callback is memoized via `useCallback` and guarded — passed as `undefined` if `props.onClick` is absent.

---

## src/components/Image/Image.tsx

**1. What is the purpose of this file?**
The core rendering component. Decides between regular image and background image rendering modes, handles the lightbox open/close lifecycle, and routes to the correct UI subcomponent (ContentImage vs. ContentIcon).

**2. What kind of logic is described in this file?**
`processImageLink` appends `?thumb=true` to the URL when `displayAs === "thumbnail"` using the URL API. In lightbox mode (`onClickType === "enlarge"`), the image is always shown at full resolution — the lightbox clones the content element and passes the original unprocessed `image` URL (without thumb param). Click handling: `onClickType === "action"` fires the action; `"enlarge"` opens the lightbox. Background rendering path passes `onClick` directly on the div without the lightbox mechanism. The wrapper becomes `hidden` when `hasImage` is false (image is undefined or empty string).

**3. What part of behavior can be documented from this file?**
When `displayAs === "thumbnail"`, a `thumb=true` query parameter is appended to the image URL for the thumbnail preview, but the lightbox always shows the full-resolution image. `tabIndex` defaults to `0` for clickable images. The `hidden` CSS class is applied when no image source is resolved. Background images do not support lightbox enlargement. `responsive` prop adds the `mx-image-viewer-responsive` class, constraining max dimensions.

**4. Is it user-facing?**
Yes — directly renders the image or icon visible to end users.

**5. What new did you learn from this file?**
Thumbnail mode uses URL query params (`?thumb=true`) — requires the image URL to be a valid absolute URL (parsed with `new URL()`). The lightbox uses `useLightboxState` hook. Background image has its own click handler that calls `onClick?.()` directly (no lightbox support). The `previewMode` prop disables lightbox and skips thumbnail URL processing.

---

## src/components/Image/ui.tsx

**1. What is the purpose of this file?**
Provides all pure presentational UI primitives: `Wrapper`, `ContentImage`, `ContentIcon`, `BackgroundImage`, and a shared `getImageContentOnClickProps` helper.

**2. What kind of logic is described in this file?**
`Wrapper` applies `mx-image-viewer`, `-responsive`, `-icon`, and custom `className` classes, plus `hidden` when no image. `ContentImage` renders an `<img>` with computed width/height styles (pixels or percentage) and alt text. `ContentIcon` renders a `<span>` with font-size for icon size; uses `aria-label` + `role="img"` for accessibility. `getImageContentOnClickProps` adds `role="button"`, `tabIndex`, and keyboard handler (`Enter`/`Space`) when onClick is set, or `role="img"` when not.

**3. What part of behavior can be documented from this file?**
Keyboard navigation: when `onClick` is provided, pressing `Enter` or `Space` fires the same handler as a mouse click. `role="button"` is set for clickable images/icons; `role="img"` for non-clickable. Alt text on images uses the HTML `alt` attribute; on icons, it uses `aria-label`. Glyphs get the `glyphicon` CSS class in addition to the icon class name. `BackgroundImage` applies `url('...')` as a CSS `backgroundImage` inline style and positions with `background-repeat: no-repeat; background-position: center` (from SCSS).

**4. Is it user-facing?**
Yes — these are the rendered DOM elements users see and interact with.

**5. What new did you learn from this file?**
`getStyle` returns empty string for `"auto"` heightUnit, deferring to browser/CSS defaults. Width does not support `"viewport"` unit (only pixels and percentage at the UI level). The `tabIndex` fallback inside `getImageContentOnClickProps` defaults to `0` if not provided.

---

## src/components/Lightbox.tsx

**1. What is the purpose of this file?**
Implements the full-screen lightbox overlay using `react-overlays/Modal`. Renders a backdrop with a close button and prevents event propagation from the backdrop to parent containers.

**2. What kind of logic is described in this file?**
`Lightbox` wraps `react-overlays/Modal` with `show`, `onHide`, `renderBackdrop`, and `onBackdropClick` props. `BackdropWithClose` renders the semi-transparent backdrop and a close button with an SVG icon. `onBackdropClick` calls `event.stopPropagation()` to prevent the backdrop click from bubbling to container onClick handlers. The close button uses an additional `onClose` prop because the library's `onHide` callback only fires on the actual backdrop element (not child elements) due to a `target !== currentTarget` check.

**3. What part of behavior can be documented from this file?**
The lightbox modal is positioned fixed, centered (50%/50%, transform translate -50%/-50%). The backdrop covers the full viewport at z-index 110 (above Atlas sidebar). The modal itself is at z-index 210 (above the backdrop). The close button shows a white SVG icon (ic24-close.svg) at 16×16px, positioned absolutely at top:30px, right:30px. Clicking the backdrop or the close button closes the lightbox.

**4. Is it user-facing?**
Yes — the lightbox is the full-screen image display shown when onClickType is "enlarge".

**5. What new did you learn from this file?**
The library (`react-overlays/Modal`) has a quirk: its `onHide` only fires on the true backdrop element due to target/currentTarget comparison. The close button therefore uses a separate `onClose` prop passed via `renderBackdrop`. Event propagation from the backdrop to outer containers is explicitly stopped.

---

## src/utils/helpers.ts

**1. What is the purpose of this file?**
Constructs the inline CSS style object for image dimensions from the widget's width/height/minHeight/maxHeight props.

**2. What kind of logic is described in this file?**
`constructStyleObject` maps each unit enum to a CSS value string: `pixels` → `${value}px`, `percentage` → `${value}%`, `viewport` → `${value}vh`. `auto` heightUnit sets `height: "auto"` and optionally sets `minHeight` and `maxHeight` (only when unit is not `"none"` and value > 0). When heightUnit is not `auto`, a fixed height is set and min/max height constraints are ignored. Width always outputs a value (pixels or percentage; auto is not handled here as it's excluded at the container level).

**3. What part of behavior can be documented from this file?**
`minHeight` and `maxHeight` are only applied when `heightUnit === "auto"`. Min/max height values of 0 are suppressed (treated as not set). The `viewport` unit maps to `vh` CSS unit. Width is always a numeric value in pixels or percentage (not empty string).

**4. Is it user-facing?**
No — internal utility.

**5. What new did you learn from this file?**
Width does not support the `"auto"` unit in `constructStyleObject` — the auto case is handled upstream (in `Image.tsx` which only calls this for `type === "image"`). The width field always emits a numeric style value.

---

## src/utils/lightboxState.tsx

**1. What is the purpose of this file?**
A simple React hook that manages the open/closed boolean state of the lightbox.

**2. What kind of logic is described in this file?**
`useLightboxState` uses `useState(false)` for `lightboxIsOpen`, with `useCallback`-memoized `openLightbox` and `closeLightbox` setters. Returns the state and two action functions.

**3. What part of behavior can be documented from this file?**
Lightbox starts closed. State is local to each Image widget instance — multiple image widgets on the same page each have independent lightbox state.

**4. Is it user-facing?**
No — internal hook.

**5. What new did you learn from this file?**
No persistence or global state — the lightbox is per-widget and resets to closed on re-mount.

---

## src/Image.editorConfig.ts

**1. What is the purpose of this file?**
Provides Studio Pro/Studio design-time behavior: conditional property visibility, structure preview rendering, validation, and custom captions in the page explorer.

**2. What kind of logic is described in this file?**
`getProperties` hides irrelevant data source fields based on the selected `datasource`. `isBackgroundImage` and `responsive` are hidden for icon datasource. `alternativeText` is hidden for background images (replaced by `children` dropzone). `defaultImageDynamic` is only shown when datasource is `"image"` and imageObject is a dynamic entity reference. Width value is hidden when widthUnit is `"auto"`. Min/max height properties are hidden unless heightUnit is `"auto"`. Icon size and dimension properties for glyph/icon icons are conditionally hidden. `onClick` action is hidden when `onClickType` is not `"action"`. `displayAs` is only shown for image datasource. `check` validates that a required source is configured for each datasource type. `getCustomCaption` returns the image URL/entity name/icon class as the page explorer label.

**3. What part of behavior can be documented from this file?**
The "Dimensions" tab is repositioned after "Data source" in Studio (web platform) via `reorderTabsForStudio`. For Mx 10.2+, the structure preview supports dynamic image natural size without explicit pixel dimensions. The structure preview for background images shows a dropzone layout with an "Image" header.

**4. Is it user-facing?**
Yes, in design-time (affects Studio experience).

**5. What new did you learn from this file?**
`defaultImageDynamic` is only shown when `imageObject.type === "dynamic"` (entity-bound). Version detection (`version?.[0] === 10 && version?.[1] >= 2`) is used to enable natural-size preview in Mx 10.2+. Glyph and icon types suppress dimension properties (iconSize is the only size control).

---

## src/Image.editorPreview.tsx

**1. What is the purpose of this file?**
Renders a live preview of the image widget inside Studio/Studio Pro's canvas using the `ImageComponent` in `previewMode`.

**2. What kind of logic is described in this file?**
`preview` function resolves the image source from preview props (static imageUrl or placeholder SVG) and renders `ImageComponent` with `previewMode=true` and `onClick=undefined`. `getPreviewCss` returns the SCSS stylesheet content for the preview iframe.

**3. What part of behavior can be documented from this file?**
In design mode, text template image URLs containing `{` and `}` (template expressions) are not rendered (falls back to placeholder). Static imageObject URLs and static defaultImageDynamic URLs are shown. Icons (glyph/image/icon) show the actual icon. Background image mode renders a dropzone for child widgets with "Place content here" placeholder.

**4. Is it user-facing?**
Yes, in design time (Studio canvas preview).

**5. What new did you learn from this file?**
Text template detection is a simple string check (`includes("{")` && `includes("}")`). Default width is 100px and height is 100px in preview mode when not configured. `iconSize` defaults to 14 in preview.

---

## src/ui/Image.scss

**1. What is the purpose of this file?**
Defines all CSS rules for the image widget: base sizing, responsive constraints, lightbox backdrop and modal positioning, background image styling.

**2. What kind of logic is described in this file?**
`.mx-image-viewer` takes 100% width/height by default; `-icon` variant overrides height to `unset`. `.mx-image-viewer-responsive img` constrains to `max-width: 100%; max-height: 100%; display: block`. Lightbox backdrop is `position: fixed; inset: 0px; background: rgba(0,0,0,0.85); z-index: 110`. Lightbox modal is `position: fixed; top/left: 50%; transform: translate(-50%, -50%); z-index: 210`. Background image variant adds `background-repeat: no-repeat; background-position: center`.

**3. What part of behavior can be documented from this file?**
The image wrapper always fills 100% of its container by default. Responsive images use `display: block` (removes inline image whitespace). Lightbox backdrop sits above Atlas sidebar (z-index 110). The lightbox modal is centered absolutely at z-index 210. Background images are centered and not repeated by default.

**4. Is it user-facing?**
Yes — defines the visual presentation.

**5. What new did you learn from this file?**
Icon wrapper height is explicitly `unset` to prevent stretching from the `height: 100%` default. The close button in the lightbox backdrop is positioned at `top: 30px; right: 30px`.

---

## src/components/__tests__/Image.spec.tsx

**1. What is the purpose of this file?**
Unit test suite for the Image component covering: rendering snapshots (image, icon, glyph, background), click behavior (action and enlarge types), keyboard/tabIndex/accessibility, thumbnail URL manipulation, and background image content rendering.

**2. What kind of logic is described in this file?**
`react-overlays/Modal` is mocked to render inline. Tests verify: `role="button"` when onClick is present, `role="img"` when not; tabIndex is respected; clicking image fires `onClick`; pressing image triggers lightbox open, close button closes it; clicking image does not bubble to parent container (`stopPropagation`); thumbnail URL includes `thumb=true` param; lightbox shows full-resolution URL (no thumb param); background image renders content and handles onClick.

**3. What part of behavior can be documented from this file?**
Alt text uses HTML `alt` attribute on `<img>` elements and `aria-label` on icon/glyph `<span>` elements. When no alt text is provided, no `alt` or `aria-label` attribute is set. Lightbox shows full-resolution image even when thumbnail mode is active (two images rendered — lightbox image does not contain `thumb=true`). Background image click fires `onClick` once; event does not propagate.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The mock setup uses `forwardRef` for the Modal. The close button selector is `.close-button`. The lightbox backdrop selector is `.mx-image-viewer-lightbox-backdrop`. `getAllByRole("img")` returns at least 2 images when the lightbox is open (thumbnail + lightbox full image). The `onKeyDown` for keyboard activation (`Enter`/`Space`) is confirmed via the `getImageContentOnClickProps` logic and covered in the unit context indirectly.

---

## e2e/dataTypes.spec.js

**1. What is the purpose of this file?**
E2e Playwright tests verifying that images load correctly from different data source types: dynamic URL, static image, and static URL.

**2. What kind of logic is described in this file?**
Tests navigate to specific pages and assert the `src` attribute of the rendered `<img>` element. Two tests are skipped (`test.skip`) for dynamic URL association and empty URL cases, with todo comments noting unresolved failures.

**3. What part of behavior can be documented from this file?**
E2e-confirmed data sources: (1) dynamic URL — `imageUrl` resolves to an absolute external URL via textTemplate; (2) static image — `imageObject` resolves to a Mendix file URL matching pattern `ImageViewer$Images$landscape_2.png`; (3) static URL — literal URL string `https://picsum.photos/200/300`. The dynamic URL association test is currently skipped (broken, cause unknown). Empty URL rendering is also skipped.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The static image path in the `src` attribute matches the Mendix file storage pattern (`ImageViewer$Images$landscape_2.png`). Session logout is forced after each test to stay within the 5-session license limit — confirms the widget runs in a licensed Mendix environment. Dynamic URL association via entity attribute is a known-broken scenario.

---

## e2e/differentViews.spec.js

**1. What is the purpose of this file?**
E2e tests verifying widget rendering in different Mendix layout containers: listening to a data grid, list view, template grid, and tab container.

**2. What kind of logic is described in this file?**
All tests are wrapped in `test.describe.skip(process.env.MODERN_CLIENT === true, ...)` — the entire suite is skipped when the Mendix modern React client is in use. Firefox is explicitly skipped for template grid and tab container tests.

**3. What part of behavior can be documented from this file?**
The image-web widget does NOT support the Mendix modern React client (MODERN_CLIENT=true). It targets the classic Dojo-based client only. The widget is e2e-confirmed to work inside: list view, template grid (Chromium only), and tab container (Chromium only). "Listen to data grid" interaction confirmed: clicking a grid row updates the displayed image to the static URL.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The `MODERN_CLIENT` environment variable controls the skip guard — this is a platform incompatibility constraint documented in tests. Firefox is specifically excluded for template grid and tab container scenarios, suggesting layout-specific rendering differences.

---

## e2e/onClick.spec.js

**1. What is the purpose of this file?**
E2e tests confirming all supported onClick action types work correctly: Microflow, Nanoflow, Open Page, and Enlarge (full-screen lightbox).

**2. What kind of logic is described in this file?**
Each test navigates to a specific page, clicks the image widget, and asserts the outcome. Microflow and Nanoflow open a modal dialog with specific text content. Open Page opens a modal window with a caption. Enlarge opens the lightbox overlay.

**3. What part of behavior can be documented from this file?**
E2e-confirmed onClick action types: Microflow (verified: shows "You clicked this image" dialog), Nanoflow (verified: shows the dynamic image URL in dialog), Open Page (verified: opens a modal window titled "GazaLand"), Enlarge/full-screen (verified: `.mx-image-viewer-lightbox img` is visible with the static image src). All four onClick types are e2e-confirmed working.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The nanoflow test confirms that the image's data context (the dynamic image URL) is passed as context to the nanoflow. The enlarge test confirms the lightbox selector is `.mx-image-viewer-lightbox` and the image inside it has the correct `src`. The `mx-name-imageRender1` and `mx-name-image1` selectors are distinct page configurations.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents all notable changes to image-web across versions following Keep a Changelog / SemVer conventions.

**2. What kind of logic is described in this file?**
Version history from v1.0.0 (2021-09-28) to v1.5.1 (2025-10-29). Key changes: v1.0.0 converted to pluggable widget; v1.1.0 fixed design properties/styles in Design mode, simplified properties; v1.2.0 added dark mode structure preview; v1.2.1 fixed CSP strict mode compatibility; v1.3.0 added custom icon collections; v1.4.0 updated captions/icons; v1.4.1 fixed tabIndex; v1.4.2 removed redundant code; v1.4.3 improved structure preview and fixed icon sizing and spacing outside containers; v1.5.0 added enhanced dimension properties (min/max height, viewport unit, percentage height); v1.5.1 fixed icon container scaling/alignment issue.

**3. What part of behavior can be documented from this file?**
v1.5.0 (2025-08-29): Added min/max height constraints and viewport (vh) unit — these are relatively new additions not present in v1.4.x. v1.4.3 (2023-10-05): Fixed icon size configurability for new icon collections (added in v1.3.0). v1.4.1 (2023-07-11): Fixed tabIndex being ignored — behavioral fix for keyboard navigation. v1.2.1 (2022-04-01): CSP strict mode compatible — image URLs must work without unsafe-inline. v1.1.0 (2021-12-03): Design mode styling fixed — widget properties are now applied in Studio.

**4. Is it user-facing?**
Yes — changelog is visible to marketplace users.

**5. What new did you learn from this file?**
The widget existed before being converted to a pluggable widget (v1.0.0). The `minHeight`/`maxHeight`/`viewport` dimension features are from v1.5.0 (August 2025). Icon container alignment bug was fixed in v1.5.1 (October 2025) — confirming there was a known disproportionate scaling issue with icon datasource.
