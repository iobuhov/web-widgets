# Draft: color-picker-web

Extracted from `packages/pluggableWidgets/color-picker-web/` on 2026-05-08.

---

## src/ColorPicker.xml

**1. Purpose:** Declares widget metadata and all configurable properties. Authoritative source for prop names, types, defaults, and enumerations.

**2. Logic described:** Four property groups: "Data source" (colorAttribute — string attribute), "General" (advanced, mode, type, format, defaultColors list, Label system property, invalidFormatMessage), "Editability" (system Editability), "Events" (onChange action), "Visibility" (system Visibility). mode has three values: popover (button), input (input box), inline. type has 11 values: block, chrome, circle, compact, github, hue, material, sketch, slider, swatches, twitter. format has three values: hex, rgb, rgba.

**3. Documentable behavior:** `colorAttribute` accepts only String attribute types. Supported color formats in the attribute are hex, rgb, and rgba — "Non-color formats such as 'red' are not supported" (as stated in description). `defaultColors` is an optional list of preset color strings. `invalidFormatMessage` uses the `:colors:` token as a placeholder that gets replaced with an example of the correct format. `advanced=false` by default — hides type/defaultColors/format from basic users. `onChange` action is not required (changed in v2.1.2 to avoid Studio Pro warnings).

**4. User-facing:** Yes. All properties appear in Studio Pro's property panel.

**5. New learnings:** The widget uses `needsEntityContext="true"` and `offlineCapable="true"`. The "advanced" toggle gates three properties (type, defaultColors, format), making the widget simpler for basic use. The `:colors:` token in `invalidFormatMessage` is a custom templating convention specific to this widget (not a platform feature).

---

## typings/ColorPickerProps.d.ts

**1. Purpose:** Auto-generated TypeScript types from ColorPicker.xml. Provides compile-time safety for container (runtime) and preview (editor) components.

**2. Logic described:** Exports: `ModeEnum` ("popover"|"input"|"inline"), `TypeEnum` (11 picker types), `FormatEnum` ("hex"|"rgb"|"rgba"), `DefaultColorsType` (object with `color: string`), `ColorPickerContainerProps` (runtime props), `ColorPickerPreviewProps` (editor props). `colorAttribute` is `EditableValue<string>` — supports read/write + readOnly state. `invalidFormatMessage` is `DynamicValue<string>` (can be dynamic text template). `onChange` is optional `ActionValue`.

**3. Documentable behavior:** `colorAttribute` being `EditableValue<string>` means the widget respects Mendix's editability rules — if the attribute is read-only (from entity access rules or the Editability property), `colorAttribute.readOnly` will be `true` and the widget disables itself. `DynamicValue<string>` for `invalidFormatMessage` means it can include dynamic expressions in Studio Pro.

**4. User-facing:** No. Internal TypeScript contract.

**5. New learnings:** The `EditableValue<string>` type is more complex than a plain string — it carries value, readOnly, validation, and setValue capabilities. The container component passes `colorAttribute.readOnly` directly as the `disabled` prop to the UI component, bridging Mendix's editability system to the widget's disabled state.

---

## src/ColorPicker.tsx

**1. Purpose:** Container entry point bridging Mendix data layer to the pure UI ColorPicker component.

**2. Logic described:** Destructures all props, wraps `executeAction(onChange)` in `useCallback`. Creates `onColorChange` callback that calls `props.colorAttribute.setValue(value)` to write back to Mendix. Passes `colorAttribute.readOnly` as `disabled`. Passes `invalidFormatMessage?.value` (extracts value from `DynamicValue`).

**3. Documentable behavior:** Color selection directly updates the Mendix attribute via `setValue`. The `onChange` action fires separately (on color change completion, debounced). `disabled` state comes from `colorAttribute.readOnly` — not a separate prop. If the attribute value is `undefined` (no value set), `color={colorAttribute.value}` will be `undefined` — the UI component handles this with a default color.

**4. User-facing:** No. Internal orchestration layer.

**5. New learnings:** Two callbacks exist: `onColorChange` (synchronous attribute update, fires on every color change) and `onChange`/`onChangeFn` (the Mendix action, debounced and only fires on completion). This distinction is critical: the attribute is updated in real-time during color dragging, but the action fires after the user finishes selecting (with 500ms debounce).

---

## src/components/ColorPicker.tsx

**1. Purpose:** Core UI component implementing color picking logic. Manages display state, color validation, debouncing, and rendering of the appropriate picker type.

**2. Logic described:** Uses `react-color` library. `getColorPicker(type)` returns the appropriate picker component. `hidden` state controls popover visibility (starts hidden for non-inline modes). `currentColor` ref tracks the last submitted color. `onChange` action is debounced 500ms via `debounce` utility. `submitColor` updates the ref and calls `onColorChange` (immediate attribute update) + aborts any pending debounced action. `onChangeComplete` fires the debounced action only if `currentColor` matches parsed color. Validates color format on every `color` prop change (via `useEffect`). Renders differently per mode: inline shows picker always; popover shows button + picker overlay; input shows text input + button + picker overlay. `disableAlpha` is `true` when format is not "rgba".

**3. Documentable behavior:** For `github`, `block`, `twitter` picker types, the triangle decoration is hidden (`triangle: "hide"`). `defaultColors` are passed as `colors` prop to the picker (except for `swatches` type which uses its own color list). `defaultColors` only work for: block, sketch, circle, compact, twitter. A "cover" div (`widget-color-picker-cover`, fixed position covering the viewport) is rendered for non-inline modes — clicking it closes the picker. When disabled in inline mode, an overlay div blocks interaction. The `Alert` component from `@mendix/widget-plugin-component-kit` shows validation errors below the picker.

**4. User-facing:** Yes. This is the rendered component users interact with.

**5. New learnings:** The 500ms debounce on `onChange` action prevents excessive action firing during color dragging. The `abortCompleteColorChange()` call in `submitColor` means: if the user makes a new selection while the debounce is pending, the previous debounced call is cancelled and the new one starts fresh. The color picker supports 11 distinct visual styles via `react-color` library, each with different UX (gradient picker, swatches, slider, etc.).

---

## src/components/Button.tsx

**1. Purpose:** Renders the color preview button that triggers the picker popover or input mode picker.

**2. Logic described:** Simple presentational component. Shows a `<button>` with a `<div>` inside styled with `background: color`. Mode determines CSS class: "input" mode uses `widget-color-picker-input-inner` (15×15px with border); other modes use `widget-color-picker-inner` (36×14px). When mode is "inline", the button gets `hidden` class. When disabled, the button gets `disabled` class.

**3. Documentable behavior:** The color swatch preview in the button directly reflects the current color via inline `style={{ background: color }}`. In popover mode the button shows a 36×14px color rectangle. In input mode it shows a smaller 15×15px square. In inline mode the button is hidden (picker always visible). The button does not have an `aria-label` — this may be an accessibility gap.

**4. User-facing:** Yes. This is the visible color swatch button.

**5. New learnings:** The button intentionally has no `type="button"` attribute, relying on default behavior. The `disabled` class is added to the outer `<button>` element alongside the Bootstrap `btn` class — this is a CSS-based disabled state, not the HTML `disabled` attribute, meaning the button may still be clickable in some browsers.

---

## src/components/Input.tsx

**1. Purpose:** Renders the text input field for "input box" mode, allowing users to type color values directly.

**2. Logic described:** Simple presentational component. Renders a `<div>` containing an `<input type="text">` and the passed `children` (the Button component). `onKeyUp` is triggered on `ArrowDown` key — calls the parent's handler to show/toggle the color picker. `onChange` updates the color attribute directly.

**3. Documentable behavior:** In input mode, users can type a color value directly (e.g., "#4caf50", "rgb(42,94,210)"). Pressing `ArrowDown` opens the color picker dropdown. The `value` is controlled (uses the `color` prop), making it a controlled input. Uses Bootstrap's `form-control` class for styling. The child Button component renders inside the input container, to the right of the text input.

**4. User-facing:** Yes. Users type color values and press ArrowDown to open the picker.

**5. New learnings:** The `ArrowDown` key binding for opening the picker is a custom keyboard UX choice — not a standard HTML pattern. This keyboard interaction is specific to "input" mode only.

---

## src/utils/index.ts

**1. Purpose:** Utility functions for color picker type selection, color parsing, color format validation, and prop validation.

**2. Logic described:** `getColorPicker(type)` — switch statement returning the correct `react-color` Picker component; defaults to SketchPicker for unknown types. `parseColor(color, format)` — converts `ColorState` (react-color's internal format) to hex/rgb/rgba string. `validateColorFormat(color, colorFormat)` — regex validation for each format; returns example string if invalid, empty string if valid. `validateProps(props)` — validates all `defaultColors` entries against the widget's format; logs errors and returns concatenated error string.

**3. Documentable behavior:** HEX validation accepts 3- or 6-digit hex codes with optional `#`. RGB format must be exactly `rgb(R,G,B)` with no spaces and values 0-255. RGBA requires `rgba(R,G,B,A)` where A is 0-1 (decimal). The `:colors:` token in `invalidFormatMessage` is replaced by example strings like `'#0d0', '#d0d0d0'` (hex), `'rgb(115,159,159)'` (rgb), or `'rgba(195,226,226,1)'` (rgba). `validateProps` only checks colors for picker types that support defaultColors: block, sketch, circle, compact, twitter.

**4. User-facing:** No. Utility functions only.

**5. New learnings:** The HEX regex allows `#0d0` (3-digit shorthand). The RGBA alpha validation `(0?\.\d*|0|1(\.0)?)` accepts `0`, `1`, `1.0`, or any decimal like `0.49`. RGB strings must have no spaces — `rgb(42, 94, 210)` (with spaces) would fail validation. This strict format is important to document as a behavioral constraint.

---

## src/ColorPicker.editorConfig.ts

**1. Purpose:** Controls property visibility and editor preview rendering in Studio Pro.

**2. Logic described:** `getProperties`: hides type/defaultColors/format when `advanced=false`; hides `invalidFormatMessage` when `mode=inline`; hides `defaultColors` when type doesn't support it (not block/sketch/circle/compact/twitter). On web, groups become tabs. `getPreview`: returns different structure previews per mode — inline shows an SVG image (full picker visual), popover/input show a color rectangle button with/without a text input placeholder.

**3. Documentable behavior:** `invalidFormatMessage` is hidden in inline mode because inline pickers don't have an input for users to type invalid values. `defaultColors` is hidden for picker types that don't use custom colors (hue, chrome, github, material, swatches, slider). The inline mode preview uses a static SVG showing a full sketch picker. Popover preview shows a standalone button; input preview shows a text box + button together.

**4. User-facing:** No. Studio Pro editor only.

**5. New learnings:** The conditional hiding of `defaultColors` by type creates an important constraint: if the user switches type from sketch (supports defaultColors) to chrome (does not), the defaultColors values are still saved but hidden and ignored at runtime. The inline SVG preview gives a more accurate preview than the other modes.

---

## src/ColorPicker.editorPreview.tsx

**1. Purpose:** Renders a live preview of the ColorPicker inside the Studio Pro canvas.

**2. Logic described:** Passes a hardcoded `color="#3A65E5"` (a specific blue), `format="hex"`, and a no-op `onColorChange` and `onChange`. Passes `disabled={props.readOnly}` to show disabled state in read-only mode. Uses the actual `ColorPicker` UI component, so the preview accurately reflects all mode and type combinations.

**3. Documentable behavior:** The preview color is always `#3A65E5` (blue), regardless of actual attribute value. In readOnly mode in the editor, the picker shows as disabled. The preview renders all three modes accurately (popover button, input box, inline picker).

**4. User-facing:** Studio Pro canvas only.

**5. New learnings:** The preview uses the real component (not a static image), so all 11 picker types are rendered live in the Studio Pro canvas. This provides accurate visual feedback but also means `react-color` is a runtime dependency in the Studio Pro preview.

---

## src/ui/ColorPicker.scss

**1. Purpose:** Defines all visual styles for the color picker widget, including overrides for react-color picker components.

**2. Logic described:** Uses `font-family: inherit` on all picker subcomponents. Overrides specific picker widths: `material-picker` forced to 130px width. `block-picker` gets custom box-shadow. Sketch and compact pickers have `input { width: 100% }` override. Disabled state: `widget-color-picker-overlay` (semi-transparent white, z-index 50) covers the widget. Circle picker has custom padding and box-shadow. `widget-color-picker-cover` is fixed-position full-viewport overlay for closing the popover. Modal overrides: inside `.modal-content` and `.modal-body`, the popover switches to `position: fixed` to prevent clipping by modal scroll containers.

**3. Documentable behavior:** The modal override (`position: fixed` for popover) is critical behavior — without it, the popover would be clipped inside a scrolling modal dialog. The disabled overlay uses `z-index: 50` and 50% white opacity. Error state adds `border-color: #a94442` to the button. The `widget-color-picker-cover` uses `position: fixed` with full viewport coverage to enable "click outside to close" behavior.

**4. User-facing:** Yes. Visual appearance and layout behavior.

**5. New learnings:** The modal-specific CSS overrides reveal a known issue: color picker popovers in Mendix modal dialogs need `position: fixed` (not `absolute`) to render above the modal. This is a behavioral constraint — the widget's popover positioning changes inside modal contexts. The `material-picker` has a hardcoded width of 130px, which may cause layout issues in narrow containers.

---

## e2e/ColorPicker.spec.js

**1. Purpose:** End-to-end Playwright tests verifying all three modes render correctly and all color formats work.

**2. Logic described:** Three test groups: (1) mode rendering — button (popover) on `/p/modePage`, input box (shows `#4caf50`), inline (shows `.sketch-picker`); (2) picker type rendering — 11 types each verified by their CSS class (e.g., `.sketch-picker`, `.chrome-picker`); (3) color format — hex (fixme), rgb (shows `rgb(42,94,210)`), rgba (shows `rgba(39,255,238,0.49)`). Button test is browser-specific (`isFirefox` conditional for CSS assertion syntax differences). HEX format test is marked `test.fixme`.

**3. Documentable behavior:** All 11 picker types render in inline mode (the e2e test uses the inline page). Button mode shows a color swatch with the background matching the stored color. Input mode shows the raw color string in the text field. The HEX test is broken/skipped (`fixme`). RGB and RGBA values include decimal alpha (e.g., `0.49`). The e2e tests confirm no `test.skip(MODERN_CLIENT)` guard — color-picker works in both Mendix clients.

**4. User-facing:** No. Test infrastructure. Behaviors tested are user-facing.

**5. New learnings:** The `test.fixme` on the hex format test indicates a known open issue. The browser-specific CSS assertion for background color (Firefox vs Chromium format difference) is a documented test fragility. The inline mode shows all 11 picker types simultaneously on the test page, confirming they are all independent inline widgets.

---

## CHANGELOG.md

**1. Purpose:** Release history from v2.0.0 to v2.1.5 plus unreleased fixes.

**2. Logic described:** Unreleased: "fixed On change action not triggering in some cases" (active bug). v2.1.5 (2026-03-06): fixed context-switch bug in "listen to widget" setup. v2.1.4: license file. v2.1.3 (2025-03-21): fixed Color Picker not working in React client. v2.1.2: `onChange` changed from required to false. v2.1.1: removed redundant code. v2.0.1: reorganized configuration structure. v2.0.0: converted to pluggable widget.

**3. Documentable behavior:** The widget works in the Mendix React client (fixed in v2.1.3). The `onChange` action is optional (fixed in v2.1.2). The "listen to widget" context-switch bug (v2.1.5) is important: in a "listen to data view" setup where context switches, stale attribute values from the previous context are no longer applied. There is an unreleased fix for `onChange` not always triggering.

**4. User-facing:** No. Developer documentation.

**5. New learnings:** The unreleased `onChange` fix confirms a current (unfixed in latest release) behavioral issue: the `onChange` action may not fire in all scenarios. The v2.1.5 context-switch fix addresses a complex reactivity bug in Mendix's "listen to widget" data pattern. The React client fix in v2.1.3 means the widget is compatible with both Mendix Dojo (classic) and React clients.
