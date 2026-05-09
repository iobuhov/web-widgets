# Draft: rating-web

Widget package: `packages/pluggableWidgets/rating-web`

---

## src/StarRating.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, props schema, and Studio Pro categorization. Generates `StarRatingProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares: `rateAttribute` (Integer/Long/Decimal); `emptyIcon` and `icon` (image-type, optional custom icons); `maximumStars` (integer, default 5); `animation` (boolean, default true); `onChange` action. System properties: Name, Editability, Visibility, TabIndex. Widget is `needsEntityContext="true"`, `pluginWidget="true"`, `offlineCapable="true"`, categorized under "Display".

**3. What part of behavior can be documented from this file?**
- The rating attribute accepts Integer, Long, and Decimal types — supporting fractional ratings if needed.
- Custom icons can be provided for both the empty and filled states via Mendix image properties.
- `maximumStars` defaults to 5 — the number of stars is configurable.
- `animation` defaults to `true` — the stretch-bounce animation is on by default.
- `onChange` action fires after a rating is selected.
- Widget is offline capable and requires entity context.

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The widget accepts Decimal for the rating attribute, meaning fractional ratings (e.g., 3.5 stars) are supported at the data layer — though the UI renders only whole-star increments. The decimal capability is for storage precision, not fractional display.

---

## src/StarRating.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. Converts Mendix props (EditableValue, DynamicValue, ActionValue) into the format expected by the internal `Rating` component.

**2. What kind of logic is described in this file?**
Checks `isAvailable(props.icon)` and `isAvailable(props.emptyIcon)` to determine if custom icons are provided; falls back to `undefined` (triggering default star SVGs). Converts `rateAttribute.value` (Big) to a JavaScript number via `.toNumber()`. Clamps the displayed value to `maximumStars` to prevent out-of-range display. On user selection: calls `rateAttribute.setValue(new Big(value))` then `props.onChange?.execute()`. Passes `rateAttribute.readOnly` as the `disabled` prop.

**3. What part of behavior can be documented from this file?**
- Custom icons are used only when the Mendix `isAvailable()` check passes — unavailable (loading or not configured) icons fall back to default star SVGs.
- Rating values are stored as `Big` instances for decimal precision.
- Displayed value is clamped to `maximumStars` — a stored value higher than the configured maximum will display as the maximum.
- `onChange` is conditional — fires only when the `onChange` prop is defined.
- The `disabled` state is driven by `rateAttribute.readOnly`, not a separate prop.

**4. Is it user-facing?**
No — internal Mendix-to-component adapter.

**5. What new did you learn from this file?**
The widget writes `new Big(value)` back to the attribute — meaning it always stores full integer values (1, 2, 3, etc.) as Big numbers. The decimal support in the XML is for reading pre-existing decimal values, not for writing decimal ratings.

---

## src/components/Rating.tsx

**1. What is the purpose of this file?**
The core interactive presentation component implementing the star rating UI with full keyboard navigation and accessibility support.

**2. What kind of logic is described in this file?**
Renders a `role="radiogroup"` container with `maximumStars` child `<div>` elements, each acting as a radio button (`role="radio"`). Current value item gets `tabIndex="0"`, others get `tabIndex="-1"` (roving tabIndex pattern). Hover state: `onMouseEnter`/`onMouseLeave` track hovered index; renders a hover preview icon inside a `.rating-item-hover` div (but only when `animation` is enabled and not disabled). Click handler: if same star clicked twice, calls `onChange(0)` (clears rating); otherwise calls `onChange(index + 1)`. Keyboard: `ArrowLeft`/`ArrowRight`/`ArrowUp`/`ArrowDown` move focus; `Space`/`Enter` select. Icon rendering: `index + 1 <= currentValue` → full icon; otherwise → empty icon.

**3. What part of behavior can be documented from this file?**
- Values are 1-indexed: star positions map to values 1, 2, ..., maximumStars.
- Clicking the currently-selected star calls `onChange(0)` — clears the rating (toggle-to-clear behavior).
- Keyboard navigation uses the roving tabIndex pattern: only the current value item is in the tab order.
- Arrow keys move focus between stars (Left/Down decrease, Right/Up increase).
- Hover preview only renders when both `animation=true` AND not `disabled`.
- The hover preview div (`.rating-item-hover`) renders the full icon ABOVE the existing icon during hover.
- Disabled state blocks all pointer events and keyboard selection.

**4. Is it user-facing?**
Yes — this is the visible, interactive component users interact with directly.

**5. What new did you learn from this file?**
The hover preview renders a `.rating-item-hover` container with the full icon overlaid — this enables CSS animations (stretch-bounce) on the hover state without animating the background icon. The animation class is applied to the hover overlay, not the base icon, so the transition between hover and selected states is smooth.

---

## src/components/StarIcon.tsx

**1. What is the purpose of this file?**
Renders the default inline SVG star icon in either filled or empty (outline) variant.

**2. What kind of logic is described in this file?**
Single component with an `empty` boolean prop. Renders different SVG `<path>` data for filled vs. empty states, both within a `viewBox="0 0 32 32"` SVG. Both use `fill="currentColor"` for CSS color inheritance.

**3. What part of behavior can be documented from this file?**
- Both filled and empty stars use `currentColor` — the star color is controlled entirely by CSS (`color` property on the container).
- The SVG has no explicit width/height — sizing is controlled by CSS (set to 24×24 px in the SCSS).
- The empty star uses an outline/stroke-like path, visually distinct from the filled star.
- `viewBox="0 0 32 32"` is consistent for both variants, ensuring they are the same shape.

**4. Is it user-facing?**
No — internal icon component. Users see the rendered stars.

**5. What new did you learn from this file?**
The `currentColor` approach makes the widget's default star color fully controllable via CSS without any JavaScript. Changing the text color on the widget container changes all star colors automatically.

---

## src/components/Icon.tsx

**1. What is the purpose of this file?**
Polymorphic icon renderer — dispatches between the built-in `StarIcon` (default) and Mendix's `IconInternal` component (for custom glyphs or images).

**2. What kind of logic is described in this file?**
Checks `value?.type`: when `"icon"` renders `<StarIcon empty={empty}>` with appropriate CSS class; otherwise renders `<IconInternal value={value}>` for custom images/glyphs. Applies CSS classes based on: full/empty state (`rating-icon-full`/`rating-icon-empty` or `rating-image-full`/`rating-image-empty`), and `animate` boolean. When `IconInternal` has no value, renders an empty `<div>` placeholder.

**3. What part of behavior can be documented from this file?**
- The widget supports three icon types: built-in SVG star, Mendix glyph icon, or image.
- The `animate` class is applied for both built-in and custom icons — CSS animation applies universally.
- Empty placeholders (`<div>`) render when `IconInternal` has no value, preventing layout shifts.
- CSS class naming distinguishes icon type (`rating-icon-*` vs `rating-image-*`) and state (`-full` vs `-empty`).

**4. Is it user-facing?**
No — internal icon dispatching. Affects which visual element users see.

**5. What new did you learn from this file?**
The `animate` class is attached here (not in Rating.tsx), enabling the animation to apply to custom icons as well as the default star SVG. This means the stretch-bounce animation works equally for all icon types configured by the developer.

---

## src/ui/rating-main.scss

**1. What is the purpose of this file?**
Complete stylesheet for the rating widget — layout, icon colors, animation, disabled state, hover behavior, and accessibility focus styles.

**2. What kind of logic is described in this file?**
`.mx-name-{name}` container: flexbox row layout. `.rating-item`: inline flex, `cursor: pointer`. `.disabled` (on container): `opacity: 0.65`, `cursor: not-allowed`. Icon sizing: 24×24 px for SVGs. Colors: empty stars `#ccc`, full stars `#ffa611` (orange). Animation: `stretch-bounce` keyframe — scale 1 → 1.5 → 0.9 → 1.2 → 1 over 500 ms. `.rating-item-hover`: hover overlay, disables animation (`animation: none`). Focus: `outline: 1px solid #0595db` on `:focus-visible` only (keyboard-only). Image icons: 2px margin. Vendor-prefixed `@-webkit-keyframes` for older browser compatibility.

**3. What part of behavior can be documented from this file?**
- Default empty star color: `#ccc` (light gray).
- Default filled star color: `#ffa611` (amber/orange).
- Stars are 24×24 px.
- Animation (`stretch-bounce`): star scales up to 150%, contracts to 90%, springs to 120%, settles at 100% — 500 ms total.
- Hover state disables animation on the hover overlay to prevent visual conflict.
- Disabled state applies 65% opacity — not 100% invisible but visually muted.
- Focus outline only appears for keyboard users (`:focus-visible`), not on mouse click.

**4. Is it user-facing?**
Yes — all visual behavior is defined here.

**5. What new did you learn from this file?**
The `.rating-item-hover` overlay has `animation: none` explicitly — this prevents the hover animation from conflicting with the selection animation. The two animations are kept separate: hover uses CSS hover state (no animation class), selection uses the `animate` class on the permanent icon.

---

## src/StarRating.editorConfig.ts

**1. What is the purpose of this file?**
Provides `getPreview()` (structure preview), `check()` (IDE validation), and `getCustomCaption()` (page explorer label) for Studio Pro.

**2. What kind of logic is described in this file?**
`getPreview()`: Decodes SVG data URIs (dark/light mode variants) and assembles a `RowLayout` of filled-star and empty-star `Image` elements. Renders up to `maximumStars` (capped at 50 to prevent performance issues), showing `maximumStars - 1` filled stars and 1 empty star. `check()`: Validates `maximumStars > 0`. `getCustomCaption()`: Returns the `rateAttribute` name or the fallback string `"Rating"`.

**3. What part of behavior can be documented from this file?**
- Structure preview always shows the last star as empty (one unfilled star), matching a "rate-and-select" UI metaphor.
- Preview stars are capped at 50 to prevent Studio Pro from hanging on very large `maximumStars` values.
- IDE validation rejects `maximumStars = 0` at design time.
- Page explorer shows the attribute name (e.g., `[rating]`) for quick identification.
- Dark mode SVG variants are used when `isDarkMode=true`.

**4. Is it user-facing?**
Yes — visible to developers in Studio Pro structure preview.

**5. What new did you learn from this file?**
The 50-star cap in `getPreview()` is a performance guard — without it, a developer setting `maximumStars=1000` could freeze Studio Pro. This is a practical defensive limit applied only at design time.

---

## src/StarRating.editorPreview.tsx

**1. What is the purpose of this file?**
Provides the live React preview component for Studio Pro's design canvas.

**2. What kind of logic is described in this file?**
Uses `mapPreviewIconToWebIcon()` to convert preview-format icon props to web runtime format. Renders `Rating` component with `value = maximumStars - 1` (showing all-but-last star filled). Sets `disabled` from `props.readOnly`. Defaults `maximumStars` to 5 when null. Parses Mendix style string into CSS object.

**3. What part of behavior can be documented from this file?**
- Design canvas preview renders the actual `Rating` component with `maximumStars - 1` value — showing a real interactive star rating (with all-but-last filled).
- Preview is non-interactive only via `disabled` when `readOnly=true`; otherwise it is live in the canvas.
- Custom icons configured in Studio are shown in the preview.
- `maximumStars=null` in preview props defaults to 5.

**4. Is it user-facing?**
Yes — visible to developers in Studio Pro design canvas.

**5. What new did you learn from this file?**
The design canvas preview renders the real `Rating` React component, meaning developers see the actual star UI — not a placeholder. This is a higher-fidelity preview than widgets that show static placeholder text.

---

## typings/StarRatingProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `StarRating.xml`. Defines `StarRatingContainerProps` (runtime) and `StarRatingPreviewProps` (Studio design-mode).

**2. What kind of logic is described in this file?**
`StarRatingContainerProps`: `rateAttribute: EditableValue<Big>`, `emptyIcon?: DynamicValue<WebIcon>`, `icon?: DynamicValue<WebIcon>`, `maximumStars: number`, `animation: boolean`, `onChange?: ActionValue`. `StarRatingPreviewProps`: icons as union types (`{ type: "glyph"; value: string } | { type: "image"; ... } | { type: "icon"; ... }`), `maximumStars: number | null`, `readOnly: boolean`, `renderMode`, `translate`, `styleObject`.

**3. What part of behavior can be documented from this file?**
- `emptyIcon` and `icon` are `DynamicValue<WebIcon>` — they can change at runtime based on expressions (not just static configuration).
- `rateAttribute` is `EditableValue<Big>` — writable with decimal precision.
- Preview icon types cover all three Mendix icon sources: glyph (CSS class), image (URL), and icon (sprite).
- `maximumStars` is `number | null` in preview (null when not configured), `number` in runtime (always set).

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
Both icon properties are `DynamicValue<WebIcon>` — meaning the empty and full star icons can be expression-driven, potentially changing based on the current rating value or other entity attributes. This enables dynamic icon theming.

---

## src/components/__tests__/Rating.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `Rating` presentational component using Jest and React Testing Library.

**2. What kind of logic is described in this file?**
Tests: snapshot for 5-star default; snapshot for disabled state; snapshot for no animation; snapshot for custom className; click selects star (onChange called with 1-indexed value); click same star twice calls onChange(0); keyboard Space selects; keyboard Enter selects; disabled blocks click; disabled blocks Space; disabled blocks Enter; clamped value when value > maximumValue; all `maximumValue` stars rendered.

**3. What part of behavior can be documented from this file?**
- Stars values are 1-indexed: clicking star at index 0 → `onChange(1)`.
- Toggle-to-clear: clicking the currently selected star → `onChange(0)`.
- Both `" "` (space) and `"Enter"` select the focused star.
- Disabled state prevents all interactions: click, Space, Enter.
- When `value > maximumValue`, the displayed value is clamped — excess value does not create extra stars.
- The component always renders exactly `maximumValue` star items.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
Tests confirm that `onChange(0)` is the signal for "clear rating" — the parent component (StarRating.tsx) must handle 0 by calling `rateAttribute.setValue(new Big(0))`. The Rating component itself does not know about the clearing semantics — it simply passes 0 to the callback.

---

## src/components/__tests__/StarRating.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `StarRating` container component with Mendix integration using `@mendix/widget-plugin-test-utils`.

**2. What kind of logic is described in this file?**
Tests: `rateAttribute.setValue()` called with `Big(1)` when first star clicked; `onChange.execute()` called after selection; `setValue` NOT called when attribute is read-only; widget rendered as disabled when `rateAttribute.readOnly=true`.

**3. What part of behavior can be documented from this file?**
- User selection triggers `rateAttribute.setValue(new Big(value))` — confirms Big.js write-back.
- `onChange` action executes after attribute write.
- Read-only attributes: neither `setValue` nor `onChange.execute` is called — interaction is fully blocked at the container level.
- Read-only state renders the `Rating` component with `disabled=true`.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The tests confirm that read-only checking happens in the container (`StarRating.tsx`), not in the presentational `Rating` component. The Rating component sees only a `disabled` boolean — it does not know whether it is disabled due to Mendix read-only state or any other reason.

---

## e2e/Rating.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the Rating widget in a real Mendix application.

**2. What kind of logic is described in this file?**
One test: navigates to the home page, waits for network idle, locates `.mx-name-rating1` widget, takes a screenshot of `.mx-name-ratingContent`, compares to `ratingPageContent.png` baseline. Session logout after each test.

**3. What part of behavior can be documented from this file?**
- The widget is identified by Mendix's auto-generated `mx-name-{widgetName}` CSS class.
- Visual regression is the primary e2e test strategy — a screenshot baseline confirms the rendered appearance.
- Session logout is performed after each test (Mendix 5-session license limit).
- The test uses `waitForNetworkIdle` to ensure page data is fully loaded before screenshotting.

**4. Is it user-facing?**
The tested rendering is user-facing.

**5. What new did you learn from this file?**
The e2e test for rating-web is minimal — only a single screenshot comparison. There are no interaction tests (click, drag) in the e2e suite. Interaction behavior is tested exclusively at the unit test level.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the rating-web widget (published as "Star Rating" on the Mendix Marketplace).

**2. What kind of logic is described in this file?**
Key versions: 3.2.2 (2026-02-10, license docs), 3.2.1 (2023-09-27, bundle size reduction), 3.2.0 (2023-06-05, page explorer caption update, dark mode icons), 3.1.2 (2023-05-23, replaced glyphicons with internal icons), 3.1.1 (2022-04-01, CSP compliance fix), 3.1.0 (2021-12-23, dark mode structure preview), 3.0.0 (2021-09-28, toolbox category and tile), 2.0.0 (2021-05-10, major refactor: ARIA/keyboard accessibility, animation, custom icons, DOM restructure).

**3. What part of behavior can be documented from this file?**
- v2.0.0 was the major accessibility and animation refactor — ARIA roles, keyboard navigation, and the stretch-bounce animation were introduced in this version.
- v3.1.2 replaced glyphicons with internal SVG icons — the widget no longer depends on an external icon font (Mendix's built-in glyphicons), improving portability and CSP compliance.
- v3.1.1 fixed Content Security Policy compatibility — the widget previously used inline styles or scripts blocked by strict CSP headers.
- All post-v2.0.0 changes are IDE improvements, maintenance, and security updates; no runtime behavior changes.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The v3.1.2 switch from glyphicons to internal SVG icons explains why `StarIcon.tsx` exists as a self-contained SVG component rather than using CSS class-based icon fonts. This was a deliberate decoupling to remove the Mendix platform's built-in icon font dependency.
