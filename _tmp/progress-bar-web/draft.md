# Draft: progress-bar-web

Widget package: `packages/pluggableWidgets/progress-bar-web`

---

## src/ProgressBar.tsx

**1. What is the purpose of this file?**
Root Mendix widget component. Converts raw `ProgressBarContainerProps` (which contain three type-variants of the three value props) into normalized `currentValue/minValue/maxValue` numbers, computes the label content, and renders the `ProgressBar` display component.

**2. What kind of logic is described in this file?**
- `getProgressBarValues()`: switch on `props.type` ("static", "dynamic", "expression") — each variant reads from a different prop set. Dynamic uses `EditableValue<Big>`, expression uses `DynamicValue<Big>`, static uses plain `number`. Fallbacks: `dynamicCurrentValue?.value ?? 0`, `dynamicMinValue?.value ?? defaultValues.minValue` (0), `dynamicMaxValue?.value ?? defaultValues.maxValue` (100).
- Label computation: if `showLabel` is false → `null`. If `labelType === "percentage"` → `${calculatePercentage(currentValue, minValue, maxValue)}%`. If `labelType === "custom"` → `props.customLabel` (a ReactNode widget slot). If `labelType === "text"` → `props.labelText?.value`.
- `onClick` is only passed when `props.onClick` is configured; otherwise `undefined`.

**3. What part of behavior can be documented from this file?**
- Three value source modes: static (design-time integer constants), dynamic (Mendix entity attributes: Decimal/Integer/Long), expression (computed Mendix expressions returning Decimal).
- Default values for dynamic/expression modes: currentValue defaults to 0, minValue to 0, maxValue to 100.
- The label can be: absent (null), percentage string, arbitrary text template string, or arbitrary widget(s) via custom widget slot.
- When no `onClick` is configured, the handler is not passed at all (not set to a no-op) — this signals to the display component to not add click styling.

**4. Is it user-facing?**
Indirectly — orchestrates data before rendering.

**5. What new did you learn from this file?**
The three type modes (static/dynamic/expression) share the same visual rendering — the type distinction is purely about where the values come from. A consumer gets the same bar regardless of value source.

---

## typings/ProgressBarProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `ProgressBar.xml`. Defines `ProgressBarContainerProps` (runtime) and `ProgressBarPreviewProps` (design-mode).

**2. What kind of logic is described in this file?**
`TypeEnum`: "static" | "dynamic" | "expression". `LabelTypeEnum`: "text" | "percentage" | "custom". Three parallel triplets of props for current/min/max value: `staticCurrentValue: number`, `dynamicCurrentValue?: EditableValue<Big>`, `expressionCurrentValue?: DynamicValue<Big>` (same pattern for min and max). `customLabel?: ReactNode` — widget slot for custom label.

**3. What part of behavior can be documented from this file?**
- Static value props are non-optional integers (defaults come from XML). Dynamic and expression value props are optional.
- `dynamicCurrentValue` is `EditableValue<Big>` — writable attribute, but the widget only reads it (no write-back).
- `expressionCurrentValue` is `DynamicValue<Big>` — expression-computed, always read-only.
- `customLabel` is a `ReactNode` widget slot — it can contain arbitrary Mendix widgets placed in Studio.
- `labelText` is `DynamicValue<string>` (text template) — supports attribute references and expressions.

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
Both "dynamic" (attribute) and "expression" modes use `Big` — meaning Decimal precision is preserved. The `Number()` conversion in `ProgressBar.tsx` loses decimal precision for display, but the source attribute maintains full Mendix Decimal precision.

---

## src/ProgressBar.xml

**1. What is the purpose of this file?**
Widget descriptor XML. Defines all props, defaults, and Studio categorization.

**2. What kind of logic is described in this file?**
Three enumeration values for `type` (static default). Three property groups for current/min/max, each with static (integer), dynamic (attribute: Decimal/Integer/Long), and expression (Decimal) variants. `showLabel` boolean (default false). `labelType` enum (text/percentage/custom, default text). `labelText` text template. `customLabel` widgets slot. Optional `onClick` action. System `Visibility` property. Widget is `offlineCapable="true"`.

**3. What part of behavior can be documented from this file?**
- Static current value defaults to 50, min to 0, max to 100 — so a freshly dropped widget shows 50% progress.
- `showLabel` is false by default — no label shown unless explicitly enabled.
- The `labelType` property description notes: "If the Size of the progress bar is set to 'Small' in the Appearance tab, then text and percentage labels will be shown in a tooltip and custom labels will be ignored." — this is a behavioral constraint related to the "small" CSS class.
- Widget is in "Display" category for both Studio and Studio Pro.
- `offlineCapable="true"` — can be used in offline apps.
- `onClick` is not required (added in v3.2.2 as `required="false"` to avoid unnecessary Studio Pro warnings).

**4. Is it user-facing?**
Defines the developer-facing configuration interface.

**5. What new did you learn from this file?**
The "small size" behavior (labels shown as tooltip, custom labels ignored) is a documented behavioral constraint in the XML, driven by an Atlas UI CSS class (`progress-bar-small`) — not by a widget prop. The widget responds to its CSS class at runtime.

---

## src/components/ProgressBar.tsx

**1. What is the purpose of this file?**
Pure display component for the progress bar. Accepts normalized numeric values and renders the Bootstrap-based progress bar DOM structure. Also handles error state (invalid value ranges).

**2. What kind of logic is described in this file?**
- `getValuesErrorMessage`: returns error messages for three invalid states: maxValue < minValue, currentValue < minValue, currentValue > maxValue.
- Calculates `percentage` via `calculatePercentage`.
- DOM: `<div className="widget-progress-bar progress-bar-medium progress-bar-primary [class]">` wrapping `<div className="progress [widget-progress-bar-alert if maxValue < 1] [widget-progress-bar-clickable if onClick]">` wrapping `<div className="progress-bar" style={{ width: `${percentage}%` }}>`.
- `title` attribute on `.progress-bar` is set only when `label` is a string — provides tooltip for small-size mode.
- When `errorMessage` exists, renders an `Alert` component with danger style below the bar.

**3. What part of behavior can be documented from this file?**
- The bar width is set via inline `style.width` (percentage) — not via CSS variable.
- When values are invalid (out of range), the bar still renders (clamped to 0% or 100% by `calculatePercentage`), AND an error alert appears below the bar.
- `.widget-progress-bar-alert` CSS class is added when `maxValue < 1` — a zero or negative max value triggers an alert class (visually signals a configuration issue).
- `.widget-progress-bar-clickable` is added when `onClick` is provided — enabling cursor and styling for interactive bars.
- Default CSS classes are hardcoded: `progress-bar-medium` (size) and `progress-bar-primary` (color) — these are Atlas UI Bootstrap classes.
- Label renders inside `.progress-bar` — labels appear inside the colored portion of the bar.

**4. Is it user-facing?**
Yes — produces the visible progress bar DOM.

**5. What new did you learn from this file?**
Error messages are displayed at runtime inside the widget (not only in Studio validation) — invalid value ranges (e.g., current > max from dynamic data) produce a visible error alert below the bar in the running application. This means runtime data errors are surfaced to end users.

---

## src/progressBarValues.ts

**1. What is the purpose of this file?**
Defines the `ProgressBarValues` interface (currentValue, minValue, maxValue: all numbers) and `defaultValues` (currentValue: 50, minValue: 0, maxValue: 100).

**2. What kind of logic is described in this file?**
Constant object with three default numeric values. Shared between the runtime component and the editor preview.

**3. What part of behavior can be documented from this file?**
Default values for dynamic/expression mode when attributes/expressions are not yet loaded or return undefined: current=50 (50%), min=0, max=100. These also serve as the design-mode preview defaults.

**4. Is it user-facing?**
Indirectly — defaults appear in design-mode preview and as fallbacks for unresolved values.

**5. What new did you learn from this file?**
The default current value of 50 (halfway) is used both for design preview and as runtime fallback when dynamic/expression values are unavailable — developers see a 50% bar by default in Studio.

---

## src/util.ts

**1. What is the purpose of this file?**
Single utility function `calculatePercentage(currentValue, minValue, maxValue): number`. Converts a value in [minValue, maxValue] to an integer percentage.

**2. What kind of logic is described in this file?**
- Clamps: if `currentValue < minValue` returns 0; if `currentValue > maxValue` returns 100.
- Otherwise: `Math.round(((currentValue - minValue) / (maxValue - minValue)) * 100)`.
- Returns `Math.abs(percentage)` — takes absolute value of the rounded result.
- Does NOT guard against `maxValue === minValue` (division by zero would give `NaN`/`Infinity`).

**3. What part of behavior can be documented from this file?**
- Percentage is always an integer (rounded to nearest 1%).
- Values below min clamp to 0%; values above max clamp to 100%.
- The `Math.abs()` call is unusual — it converts negative percentages (possible if range is zero/negative) to positive. This partially masks misconfiguration.
- Out-of-range values produce clamped bar widths AND error alerts (per `components/ProgressBar.tsx`).

**4. Is it user-facing?**
Indirectly — determines the visual bar width and percentage label.

**5. What new did you learn from this file?**
The `Math.abs()` on the final percentage result means that even if `calculatePercentage` somehow produces a negative result (e.g., due to zero-range edge cases), the bar width is still non-negative. The real protection against negative widths is the upstream value clamping.

---

## src/ProgressBar.editorConfig.ts

**1. What is the purpose of this file?**
Provides `getProperties` (prop visibility), `check` (validation), `getPreview` (structure preview SVG), and `getCustomCaption` (Studio page explorer label) for the Studio/Studio Pro editor.

**2. What kind of logic is described in this file?**
- `getProperties`: Shows only the props relevant to the selected `type` — hides the other 6 value props (2 sets × 3 props each). If `showLabel` is false, hides all label props; if true, hides non-selected label sub-props.
- `check`: For dynamic/expression type, validates all three value props are set. For static type, validates current ≥ min and current ≤ max.
- `getPreview`: Returns `null` for custom label type (cannot preview custom widgets in SVG); otherwise returns the SVG structure preview (dark/light mode).
- `getCustomCaption`: Returns `[type, currentValue]` format (e.g., `[dynamic, someAttribute]`) or "Progress Bar".

**3. What part of behavior can be documented from this file?**
- The Studio editor shows only the 3 value props relevant to the selected type — clean, no confusing unused fields.
- Static mode validates value ranges at design time; dynamic/expression modes only validate that props are set (not value relationships, since values are runtime).
- Custom label mode disables the structure preview in Studio (returns null) — SVG preview cannot show widget content.
- Studio page explorer caption format: `[dynamic, CurrentValueAttribute]` — helpful for identifying which attribute is driving the bar.

**4. Is it user-facing?**
Yes — visible to developers in Studio/Studio Pro.

**5. What new did you learn from this file?**
The `check` function validates static value range constraints at design time (current ≥ min, current ≤ max), but this same check is NOT performed for dynamic/expression modes — those errors appear only at runtime via the alert in the display component.

---

## src/ProgressBar.editorPreview.tsx

**1. What is the purpose of this file?**
Design-mode canvas preview using the actual `ProgressBar` display component with placeholder values.

**2. What kind of logic is described in this file?**
Uses `getProgressBarValues` (same logic as runtime but with string-to-number parsing for expression/dynamic values, falling back to defaults). Renders the real `ProgressBar` component with `onClick={undefined}`. For custom labels, renders the widgets via `props.customLabel.renderer`.

**3. What part of behavior can be documented from this file?**
- Design-mode preview renders the actual progress bar (not a static SVG) — developers see a live-ish bar with current config values.
- For dynamic/expression modes, the preview uses attribute names as strings — `asNumber(value, default)` parses them as numbers or falls back to default (50% for current value).
- `onClick` is always undefined in design mode — the bar is never clickable in Studio.
- Custom label content (widgets) is rendered in design mode via the widget renderer.

**4. Is it user-facing?**
Yes — visible to developers in Studio design canvas.

**5. What new did you learn from this file?**
The `asNumber` helper gracefully handles non-numeric strings (attribute names, expressions) — if the string isn't a valid number, it falls back to the default value. This means in design mode, dynamic/expression bars show 50% by default unless the attribute name happens to parse as a number.

---

## src/components/__tests__/ProgressBar.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `ProgressBar` display component covering: structure, percentage calculation, click behavior, range handling, clamping, error messages, and label types.

**2. What kind of logic is described in this file?**
Tests confirm:
- Bar width is `23%` for `currentValue=23, min=0, max=100`.
- Bar width is `30%` for `currentValue=23, min=20, max=30` — confirming range-relative calculation.
- Values below min clamp to `0%`; values above max clamp to `100%`.
- Clicking a clickable bar calls onClick.
- `.widget-progress-bar-clickable` absent when no onClick provided.
- Error alerts appear for all three error conditions (current < min, current > max, max < min).
- Label accepts string, ReactNode component, or null (no text).
- String label gets `title` attribute (tooltip) when class is `progress-bar-small`.
- Component label (ReactNode) does NOT get `title` attribute even when class is `progress-bar-small` (custom label ignored per XML description).

**3. What part of behavior can be documented from this file?**
- Clicking the `.progress` div (not the `.progress-bar` fill) triggers onClick — the full bar container is the clickable area.
- The small-size tooltip behavior is driven by `label` type (string vs ReactNode) — if `label` is a string, `title` is set; if it's a component, no `title` is set.
- Error alerts coexist with a rendered (clamped) bar — both appear simultaneously.
- Range of `min=20, max=30, current=23` → 30% confirmed — range is properly offset, not just current/max.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The "small size tooltip" behavior is not CSS-driven at the component level — it's controlled by passing a string label. When `labelType === "custom"` in the container, `label` is a ReactNode and therefore no `title` tooltip is set. This is why the XML documentation says "custom labels will be ignored" in small size — no tooltip is possible for ReactNode labels.

---

## e2e/differentViews.spec.js

**1. What is the purpose of this file?**
E2E tests confirming the progress bar renders correctly in different Mendix page container types.

**2. What kind of logic is described in this file?**
Five tests: group box, listens-to-data-grid (selecting a row updates the bar), list view, template grid, and tab container (on tab 2). All tests verify that the progress bar text matches a reference text box on the same page.

**3. What part of behavior can be documented from this file?**
- The widget works in all common Mendix layout containers: group box, list view, template grid, tab container.
- The widget responds to "listen to grid" pattern — selecting a data grid row updates the progress bar via data context.
- In tab container, the bar correctly renders when its tab is activated.

**4. Is it user-facing?**
Tested behaviors are user-facing.

**5. What new did you learn from this file?**
The "listen to grid" pattern is e2e-confirmed — clicking a grid row updates the progress bar context, and the bar reflects the new data. This confirms dynamic attribute binding works with grid selection contexts.

---

## e2e/displayText.spec.js

**1. What is the purpose of this file?**
E2E tests for label display modes: attribute text, no text, static text, percentage, and value (raw number).

**2. What kind of logic is described in this file?**
- Attribute text test: verifies bar text matches an input field value (dynamic binding confirmed).
- No text: three bars all show empty text (showLabel=false confirmed).
- Static text: three bars show hardcoded strings "Static text 1/2/3".
- Percentage: bars show "45%", "67%", "0%".
- Value: bars show raw numbers "45", "67", "0" (the `displayValue` page presumably uses a text template showing the raw value, not percentage).

**3. What part of behavior can be documented from this file?**
- All three label types (text, percentage, custom/value) are e2e-confirmed.
- "0%" is a valid e2e-confirmed state for a bar at 0% progress.
- Attribute text binding to the label is e2e-confirmed: the label reflects the bound attribute value.

**4. Is it user-facing?**
Tested behaviors are user-facing.

**5. What new did you learn from this file?**
The "displayValue" page shows raw numbers (45, 67, 0) in the label — this appears to use the text template label type with an attribute reference, not the "percentage" type. Percentage labels always append "%"; value labels show the raw number.

---

## e2e/errors.spec.js

**1. What is the purpose of this file?**
E2E test for the "no context" error state.

**2. What kind of logic is described in this file?**
One test: when the widget has no data context, it renders a bar with "0%" label.

**3. What part of behavior can be documented from this file?**
- Without data context (dynamic attributes unavailable), the widget falls back to showing 0% progress.
- This confirms the default fallback `dynamicCurrentValue?.value ?? 0` in the container component.
- The widget does not crash or show an error when context is missing — it gracefully falls back.

**4. Is it user-facing?**
Tested behavior is user-facing.

**5. What new did you learn from this file?**
The "no context" state is explicitly tested — the widget is designed to work gracefully without entity context, showing 0% as the fallback. This is a deliberate UX decision, not just a code side-effect.

---

## e2e/onClick.spec.js

**1. What is the purpose of this file?**
E2E tests confirming the onClick action handler fires for all supported Mendix action types.

**2. What kind of logic is described in this file?**
Five action types: Microflow (opens modal dialog with bar label text), Nanoflow (updates a text box), Open Full Page (navigates to new page), Open Popup Page (opens popup), Open Blocking Popup Page (opens blocking popup). Each test clicks the progress bar and verifies the expected result.

**3. What part of behavior can be documented from this file?**
- All five Mendix action types for onClick are e2e-confirmed: microflow, nanoflow, open full page, open popup page, open blocking popup page.
- Clicking the bar (not just the fill portion) triggers the action — the clickable area is the full `.progress` container.
- Microflow result is displayed inside a modal dialog containing the progress bar label text — confirming data context is passed to the microflow.

**4. Is it user-facing?**
Tested behaviors are user-facing.

**5. What new did you learn from this file?**
All five standard Mendix action types are explicitly e2e-confirmed for the progress bar onClick. The widget does not restrict action types — any Mendix action can be used.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history from v2.1.0 to v3.2.3.

**2. What kind of logic is described in this file?**
No logic — version history entries.

**3. What part of behavior can be documented from this file?**
Key changes:
- v3.2.3 (2026-02-10): Added license file and open source dependency docs.
- v3.2.2 (2024-08-28): `onClick` made `required="false"` to remove unnecessary Studio Pro warnings.
- v3.2.0 (2023-06-05): Page explorer caption updated to `[type, currentValue]` format.
- v3.1.0 (2021-12-23): Dark mode support for structure preview; dark icons added.
- v3.0.1 (2021-12-03): Fixed design properties/styles not applied in Design mode & Studio.
- v3.0.0 (2021-09-28): Added toolbox category and tile image for Studio & Studio Pro.
- v2.1.0 (2021-07-07): Added structure preview.

**4. Is it user-facing?**
Publicly visible on Mendix Marketplace.

**5. What new did you learn from this file?**
The structure preview was only added in v2.1.0 — earlier versions had no visual representation in Studio's structure mode. The `onClick` being required was a bug (causing spurious Studio Pro warnings) fixed in v3.2.2.
