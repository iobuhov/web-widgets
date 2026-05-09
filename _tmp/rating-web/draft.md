# rating-web — Draft Spec

Widget: `rating-web`
Package: `packages/pluggableWidgets/rating-web/`
Agent: worker
Date: 2026-05-09

---

## src/StarRating.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component (`StarRating`). It bridges Mendix framework props to the internal `Rating` component, resolves icons, sets up the onChange handler, and enforces the value ceiling against `maximumStars`.

**2. What kind of logic is described in this file?**
- `emptyIcon` and `fullIcon` are constructed from Mendix `DynamicValue<WebIcon>` props using `isAvailable()` guards; when unavailable or not set, a fallback `Icon` with empty `iconClass` is used.
- The `onChange` callback wraps `props.rateAttribute.setValue(new Big(value))` and then fires `executeAction(props.onChange)`, guarded by checking `rateAttribute.status === ValueStatus.Available`.
- The displayed `value` is capped: `value > props.maximumStars ? props.maximumStars : value`.

**3. What part of behavior can be documented from this file?**
- When `rateAttribute` is unavailable (e.g., loading), `value` defaults to `0`.
- Clicking a star calls `rateAttribute.setValue(new Big(value))` — always stores as `Big`, even though the attribute can be Integer/Long/Decimal.
- The `onChange` action fires after `setValue`; both are skipped if the attribute is not in `Available` status.
- Icon fallback: if no icon is configured or not yet available, an invisible star (empty `iconClass`) is used.

**4. Is it user-facing?**
No — internal Mendix-to-component adapter.

**5. What new did you learn from this file?**
Unlike progress-bar/progress-circle (which use a 3-way type enum), rating-web always binds to a single entity attribute (`EditableValue<Big>`). The value cap (`value > maximumStars ? maximumStars : value`) is enforced at the container level rather than inside the presentation component, but the unit test confirms the cap is also implicitly tested via rendering.

---

## src/StarRating.xml

**1. What is the purpose of this file?**
Mendix widget descriptor that declares the widget's identity, category, properties, and system property requirements for Studio Pro.

**2. What kind of logic is described in this file?**
Declares five user-configurable properties in two groups:
- **General**: `rateAttribute` (attribute — Integer/Long/Decimal), `emptyIcon` (icon, optional), `icon` (icon, optional), `maximumStars` (integer, default 5), `animation` (boolean, default true).
- **Events**: `onChange` (action, optional).
- **Common** (system properties): Name, Editability, Visibility, TabIndex.

**3. What part of behavior can be documented from this file?**
- `needsEntityContext="true"` — widget requires a Mendix entity context (data source object).
- `offlineCapable="true"` — works offline (PWA/Mendix native offline apps).
- `maximumStars` defaults to `5`, representing the number of star icons rendered.
- Both icon properties are `required="false"`, allowing the widget to display without custom icons.
- No `onClick` property — interaction is handled entirely by the attribute write-back pattern, not an explicit action button.

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
There is no `onClick` action property; user interaction maps directly to attribute mutation. This is different from progress-bar/circle which have an optional `onClick` for arbitrary actions. The rating widget's only event hook is `onChange`, which fires after the attribute is set.

---

## typings/StarRatingProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript type definitions derived from `StarRating.xml`, providing type-safe access to widget props.

**2. What kind of logic is described in this file?**
- `StarRatingContainerProps` (runtime): `rateAttribute: EditableValue<Big>`, `emptyIcon?: DynamicValue<WebIcon>`, `icon?: DynamicValue<WebIcon>`, `maximumStars: number`, `animation: boolean`, `onChange?: ActionValue`.
- `StarRatingPreviewProps` (Studio Pro preview): same shape but with icon types as union literals (`"glyph" | "image" | "icon"`), `maximumStars: number | null` (nullable for unset state), `onChange: {} | null`.

**3. What part of behavior can be documented from this file?**
- `rateAttribute` is always `EditableValue<Big>` — never a `DynamicValue` or plain number. The widget always binds to a mutable entity attribute.
- `maximumStars` is a plain `number` at runtime (never null), but `number | null` in preview (Studio Pro may leave it unset).
- `emptyIcon` and `icon` are both optional (`?`) — the widget handles missing icons gracefully with fallback.

**4. Is it user-facing?**
No — TypeScript types only.

**5. What new did you learn from this file?**
`maximumStars: number | null` in `StarRatingPreviewProps` explains why the editor config uses `props.maximumStars !== null` checks before building the preview array and why `editorPreview.tsx` uses `props.maximumStars ?? 5`.

---

## src/StarRating.editorConfig.ts

**1. What is the purpose of this file?**
Provides Studio Pro design-time behavior: structure preview (SVG-based star icons), validation errors, and custom caption.

**2. What kind of logic is described in this file?**
- `getPreview`: builds a `RowLayoutProps` with `padding: 8` and `columnSize: "fixed"`. When `maximumStars !== null`, renders `(capped at 50) - 1` filled star images and 1 empty star image (24×24 SVG each), giving visual preview of rating state. When null (unset), renders 1 filled star.
- `check`: validates `maximumStars > 0`; emits error on property key `"maximumValue"` (note: the property is named `maximumStars` in XML — this key mismatch may be a bug or intentional reference to a different validation slot).
- `getCustomCaption`: returns `values.rateAttribute` (the bound attribute name) or `"Rating"` as fallback.

**3. What part of behavior can be documented from this file?**
- Structure preview shows (maximumStars-1) filled stars + 1 empty star, capped at 50 icons maximum.
- Dark mode is supported: separate SVGs imported for light and dark modes.
- The SVG assets are embedded as `data:image/svg+xml,...` URIs, decoded before passing to the structure preview API.
- Error condition: `maximumStars <= 0` produces a validation error with message "Number of stars should be greater than zero (0)".

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
The cap of 50 stars in the preview (`props.maximumStars > 50 ? 50 : props.maximumStars`) exists to prevent rendering hundreds of SVG elements in the Studio Pro canvas. At runtime, `maximumStars` has no such cap — the widget renders exactly as many stars as configured.

---

## src/StarRating.editorPreview.tsx

**1. What is the purpose of this file?**
Provides a live React-rendered preview of the rating widget inside Studio Pro (design mode), using the actual `Rating` component.

**2. What kind of logic is described in this file?**
- Maps preview icon props using `mapPreviewIconToWebIcon()` for both `emptyIcon` and `icon`.
- Passes `value={(props.maximumStars ?? 5) - 1}` — shows one fewer filled star than the total, representing a "nearly complete" rating to give a realistic preview.
- Passes `disabled={readOnly ?? false}` — respects the Mendix editability setting in preview.
- `getPreviewCss()` returns the SCSS file content for injection into the Studio Pro preview iframe.

**3. What part of behavior can be documented from this file?**
- The preview value is always `maximumStars - 1` (e.g., 4 of 5 stars filled).
- Animation is passed through from props — the preview will show the configured animation setting.
- No `onChange` handler is passed to the preview `Rating` — preview is non-interactive.

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
Setting preview `value` to `maximumStars - 1` (rather than e.g. `Math.floor(maximumStars / 2)`) gives a more "full-looking" rating preview that visually communicates the widget's purpose clearly. This is a deliberate UX choice for the Studio Pro canvas.

---

## src/components/Rating.tsx

**1. What is the purpose of this file?**
The core presentation React component. Renders the star rating widget as a row of interactive `radio` items with hover, keyboard navigation, and click-to-clear support.

**2. What kind of logic is described in this file?**
- Maintains `hover` state (`useState<undefined | number>`) for hover highlighting.
- `onClickAction` is only defined when not disabled. Clicking the already-selected star sends `onChange(0)` (clearing the value); clicking any other star sends `onChange(currentIndex)`.
- `focusItem(direction)` reads `containerRef.current.querySelector(".rating-item:focus")` and calls `.focus()` on the adjacent sibling — enabling arrow-key keyboard navigation.
- The container uses `role="radiogroup"` with `onKeyDown` handling `ArrowLeft`/`ArrowRight` (plus Edge legacy `Left`/`Right`).
- Each star item: `role="radio"`, `aria-checked={currentIndex === props.value}`, `aria-label={currentIndex.toString()}`.
- `tabIndex` management: the currently selected star (or star at index 0 if value is 0) gets `tabIndex={0}`, all others get `tabIndex={-1}`.
- Render logic per star: if `currentIndex <= value` → `fullIcon`; else if hover exists and `currentIndex <= hover` → wrapped in `.rating-item-hover` div with `fullIcon`; otherwise → `emptyIcon`.

**3. What part of behavior can be documented from this file?**
- Clicking an already-selected star sets value to 0 (deselect/clear behavior).
- Hover highlighting only activates when `!disabled && animated`.
- Keyboard: Space or Enter activates a star; ArrowLeft/ArrowRight moves focus between stars.
- Accessibility: full ARIA radiogroup pattern with checked state per star.
- `tabIndex` follows the roving tabindex pattern — only one item is in the tab order at a time.

**4. Is it user-facing?**
Yes — this is the visible, interactive component users see and interact with.

**5. What new did you learn from this file?**
The click-to-deselect (clicking the current star → sends `onChange(0)`) is implemented at the `Rating` component level, not in the container. The container receives `onChange(0)` and writes `new Big(0)` to the attribute, effectively clearing the rating. This is the only way to set the rating to zero via click.

---

## src/components/Icon.tsx

**1. What is the purpose of this file?**
An icon resolver component that selects the appropriate rendering path based on the `WebIcon` value type (`"icon"` → `StarIcon`, `"image"` → `IconInternal`, other → `IconInternal` with fallback).

**2. What kind of logic is described in this file?**
- If `value?.type === "icon"` (glyphicon/internal icon class): renders `StarIcon` with CSS classes `rating-icon`, plus `rating-icon-empty` or `rating-icon-full`, plus `animate` when enabled.
- If `value?.type === "image"`: renders `IconInternal` with CSS classes `rating-image`, plus `rating-image-empty` or `rating-image-full`, plus `animate`.
- Fallback (glyph type or undefined): renders `IconInternal` with no className override.

**3. What part of behavior can be documented from this file?**
- The `animate` prop adds the CSS `animate` class that triggers the `stretch-bounce` keyframe animation on selection.
- The internal star SVG (`StarIcon`) is used whenever `type === "icon"` (the default empty `iconClass`). Custom images (`.jpg`, `.png`, etc.) go through `IconInternal`.
- `fallback={<div />}` on `IconInternal` renders an empty div when the icon source fails to load.

**4. Is it user-facing?**
No — internal rendering helper.

**5. What new did you learn from this file?**
The `type === "icon"` check is used to detect the default/no-icon case (empty `iconClass: ""`), routing to the custom SVG `StarIcon` rather than Mendix's `IconInternal`. This means the widget always shows custom SVG stars unless the user explicitly picks an image icon. When `type` is `undefined` (icon prop not set), the fallback path through `IconInternal` renders nothing visible (the `fallback={<div />}` is invisible).

---

## src/components/StarIcon.tsx

**1. What is the purpose of this file?**
Renders the default star SVG shape in either empty (outlined) or filled (solid polygon) form.

**2. What kind of logic is described in this file?**
- Simple conditional: `empty ? <outlined SVG> : <filled SVG>`.
- Both SVGs use `fill="currentColor"`, `viewBox="0 0 32 32"`.
- Empty star: an SVG `<path>` drawing a 5-pointed star outline using a path with two sub-paths.
- Filled star: an SVG `<polygon>` with pre-computed coordinates for a solid 5-pointed star.
- Wrapped in a `<span>` with the passed `className`.

**3. What part of behavior can be documented from this file?**
- Color is inherited from CSS (`fill="currentColor"`): `.rating-icon-empty { color: #ccc }` and `.rating-icon-full { color: #ffa611 }` in the stylesheet.
- Both stars are 24×24 px (set by CSS in SCSS, not in the SVG itself).
- The `full` prop is accepted but unused in the component body; only `empty` is checked.

**4. Is it user-facing?**
Yes — this is the visible default star shape users see when no custom icon is configured.

**5. What new did you learn from this file?**
The empty star uses an outlined `<path>` while the filled star uses a `<polygon>` — they are not the same star with different fill. This means if the Mendix theme overrides the star color, it affects both states uniformly (both use `currentColor`).

---

## src/ui/rating-main.scss

**1. What is the purpose of this file?**
Stylesheet for the rating widget, defining layout, icon appearance, hover state, disabled state, focus indicator, and the `stretch-bounce` animation.

**2. What kind of logic is described in this file?**
- `.widget-rating`: `display: flex; flex-direction: row` — stars laid out horizontally.
- `.rating-item`: `cursor: pointer`; `.disabled`: `cursor: not-allowed; opacity: 0.65`.
- Focus indicator: `:focus-visible` adds `outline: 1px solid #0595db` on `.rating-image` and `.rating-icon`.
- `.rating-icon-empty`: `color: #ccc` (light grey).
- `.rating-icon-full`: `color: #ffa611` (amber/gold); `&.animate` triggers `stretch-bounce` animation.
- `.rating-image-full`: `&.animate` also triggers `stretch-bounce`.
- `@-webkit-keyframes stretch-bounce`: scale 1 → 1.5 → 0.9 → 1.2 → 1 over 0.5s ease-in-out.
- Hover state (`.rating-item-hover`): disables animation on full icons within hover state (`.animate { animation: none }`).

**3. What part of behavior can be documented from this file?**
- Selected stars are amber/gold (`#ffa611`), empty stars are light grey (`#ccc`).
- On selection, the `stretch-bounce` animation plays (0.5s) when `animation: true`.
- Hover preview (non-selected stars shown as full while hovering) does NOT play the bounce animation.
- Disabled rating items have `opacity: 0.65` and `cursor: not-allowed`.
- Keyboard focus shows a blue outline (`#0595db`) using the modern `:focus-visible` selector (ignoring mouse clicks).

**4. Is it user-facing?**
Yes — entirely user-facing, controls all visual appearance.

**5. What new did you learn from this file?**
The `.rating-item-hover .rating-icon-full.animate { animation: none }` rule disables the bounce animation specifically during hover preview. This ensures the bounce only plays when the user actually selects a star, not while they're hovering. This is a deliberate UX decision to keep hover feedback subtle.

---

## src/components/__tests__/Rating.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `Rating` presentation component — rendering structure and interaction events.

**2. What kind of logic is described in this file?**
- **Rendering tests**: snapshot tests for default, disabled, no-animation, custom class, correct number of items, and value-capped-to-maximumValue scenarios.
- **Event tests**: click fires onChange with correct 1-based index; clicking current star calls `onChange(0)` on second click (re-render with updated value); Space/Enter key fires onChange; disabled state prevents all events.

**3. What part of behavior can be documented from this file?**
- Confirmed: `maximumValue: 2, value: 5` renders 2 radio items and both show `.full` icon (value capped at display, all items ≤ value).
- Confirmed: Click-to-deselect — clicking star at `value=2` a second time calls `onChange(0)`.
- Confirmed: Disabled component does not call `onChange` on click, Space, or Enter.
- Confirmed: Arrow key navigation is implemented via `focusItem` (tested implicitly via `onKeyDown` handlers in structural render).

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The `tabIndex` assignment on items — `index === Math.max(props.value - 1, 0) ? 0 : -1` — means when `value=0`, the first star gets `tabIndex=0`. When `value=5`, the fifth star gets `tabIndex=0`. This implements the roving tabindex pattern for ARIA radiogroup accessibility.

---

## src/components/__tests__/StarRating.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `StarRating` container component — rendering structure (with Mendix props) and attribute/action interaction.

**2. What kind of logic is described in this file?**
- **Rendering tests**: snapshot for default (value=5, maximumStars=5), without animation, and when `readOnly` (disabled state).
- **Event tests**:
  - Clicking a star calls `onChange.execute()`.
  - `rateAttribute.setValue` is called with `new Big(1)` when clicking the first star (index=1).
  - When attribute is read-only, `setValue` is NOT called.
  - When attribute is read-only, `onChange` action is NOT executed.

**3. What part of behavior can be documented from this file?**
- Confirmed: `rateAttribute.readOnly` maps directly to `disabled` in the `Rating` component.
- Confirmed: Both `setValue` and `executeAction` are guarded by `status === ValueStatus.Available` — read-only attributes have status Available but `readOnly: true`, so the guard is in `onChange` (status check), but readOnly check prevents `setValue` since the status is Available for readOnly (looking at the implementation, `onChange` is only called if `status === Available`; `setValue` would also be called... but the test says setValue is NOT called when readOnly. Looking at the source again: `if (props.rateAttribute.status === ValueStatus.Available) { props.rateAttribute.setValue(...); executeAction(props.onChange); }`. The EditableValueBuilder with `.isReadOnly()` may still have status Available but mock setValue to not work, OR the status is not Available for readOnly values in tests.)

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The `EditableValueBuilder.isReadOnly()` in test utils makes `rateAttribute.readOnly = true` while keeping status Available. The guard in `StarRating.tsx` only checks `status === ValueStatus.Available` — but since `readOnly` maps to `disabled` in the `Rating` component, the `onClickAction` callback is set to `undefined` when disabled, so `onChange` is never called and `setValue` never reaches the container. The test confirms: readOnly → Rating is disabled → click has no effect → setValue/onChange never called.

---

## e2e/Rating.spec.js

**1. What is the purpose of this file?**
Playwright end-to-end test that verifies the rating widget renders correctly in a running Mendix app.

**2. What kind of logic is described in this file?**
- Single test: navigates to the root page, waits for `networkidle`, checks `.mx-name-rating1` is visible, and takes a screenshot of `.mx-name-ratingContent` compared against a baseline `ratingPageContent.png`.
- Session logout after each test to stay within Mendix's 5-session license limit.

**3. What part of behavior can be documented from this file?**
- The widget's DOM root gets a Mendix-generated class `mx-name-rating1` (based on widget Name property).
- The test uses screenshot comparison — verifies visual appearance rather than specific DOM structure.
- Only one test scenario is covered: a static page screenshot. No interaction (click, keyboard) is tested at the E2E level.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The E2E test suite for rating-web is minimal (one screenshot test), unlike progress-bar-web which has multiple scenario groups (differentViews, displayText, errors, onClick). The screenshot baseline is stored at `e2e/Rating.spec.js-snapshots/ratingPageContent-chromium-linux.png`.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history documenting changes since v2.0.0.

**2. What kind of logic is described in this file?**
Key version milestones:
- **v2.0.0 (2021-05-10)**: Major rewrite — added Size design property, full ARIA (radiogroup/radio/aria-label/aria-checked), keyboard events, animation on selection; DOM structure changed for custom icon support; Editability moved to system property; removed inline styles.
- **v3.0.0 (2021-09-28)**: Added toolbox category and tile image for Studio/Studio Pro.
- **v3.1.0 (2021-12-23)**: Added dark mode to structure mode preview and dark icons.
- **v3.1.1 (2022-04-01)**: Fixed CSP strict mode compatibility.
- **v3.1.2 (2023-05-23)**: Replaced glyphicons with internal icons.
- **v3.2.0 (2023-06-05)**: Updated page explorer caption to display datasource; updated icons/tiles.
- **v3.2.1 (2023-09-27)**: Removed redundant code (performance improvement).
- **v3.2.2 (2026-02-10)**: Added license file and README for OSS dependencies.

**3. What part of behavior can be documented from this file?**
- The ARIA radiogroup pattern was added in v2.0.0 — this was a significant accessibility overhaul.
- Keyboard navigation (arrow keys, space, enter) was introduced in v2.0.0.
- Animation on selection was introduced in v2.0.0.
- Custom icon support (the DOM restructuring) was v2.0.0.
- CSP compatibility fix in v3.1.1 — widget had inline style issues before.

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
The rating widget underwent a major rewrite in v2.0.0, fundamentally changing the DOM structure to support custom icons. All the accessibility features (ARIA, keyboard) were added at once in that release. The current implementation is the v2.0.0+ architecture.

---

## Summary of Key Findings

- **Architecture**: Single `EditableValue<Big>` attribute binding (no multi-mode like progress widgets). The value is always stored as `Big` regardless of the attribute type (Integer/Long/Decimal).
- **Click-to-clear**: Clicking the currently selected star sends `onChange(0)`, clearing the rating. This is implemented in `Rating.tsx`.
- **Custom icons**: Supported via Mendix `WebIcon` (`DynamicValue<WebIcon>`) with three rendering paths: custom SVG stars (default, `type: "icon"`), Mendix `IconInternal` for images, fallback to `IconInternal` for glyphs.
- **Animation**: `stretch-bounce` CSS keyframe (0.5s) plays on selection but is suppressed during hover. Controlled by the `animation: boolean` prop.
- **Accessibility**: Full ARIA radiogroup pattern with roving tabindex. Arrow keys navigate between stars; Space/Enter activates. Fully keyboard-operable.
- **Disabled state**: Maps to `cursor: not-allowed; opacity: 0.65` visually; blocks all onChange/setValue calls functionally.
- **Value cap**: Values above `maximumStars` are silently capped to `maximumStars` in the container.
- **offlineCapable**: `true` — works in Mendix offline apps.
- **E2E coverage**: Minimal — only one visual screenshot test; no interaction testing at E2E level.
- **No onClick**: Unlike progress widgets, rating has no general `onClick` action — user interaction is purely attribute write-back + `onChange` event.
