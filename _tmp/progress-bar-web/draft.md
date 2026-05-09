# progress-bar-web — Draft Spec

Widget: `progress-bar-web`  
Package: `packages/pluggableWidgets/progress-bar-web/`  
Agent: worker  
Date: 2026-05-09

---

## src/ProgressBar.tsx

**1. What is the purpose of this file?**
The top-level Mendix pluggable widget entry component. It resolves the widget's `type` prop (static, dynamic, or expression) into numeric `currentValue`, `minValue`, and `maxValue`, then delegates rendering to the inner `ProgressBar` component.

**2. What kind of logic is described in this file?**
A switch on `props.type` to extract values from three different source types: static integers (from widget config), dynamic attribute values (`EditableValue<Big>`), and expression values (`DynamicValue<Big>`). All values are coerced to `Number()`. The label is computed inline based on `showLabel`, `labelType`, and the computed percentage.

**3. What part of behavior can be documented from this file?**
- Three value source types: `static` (hardcoded integers), `dynamic` (entity attribute — Decimal, Integer, or Long), `expression` (Mendix expression returning Decimal).
- For dynamic and expression types, `null`/`undefined` values fall back to `defaultValues` (currentValue=50, minValue=0, maxValue=100).
- The `onClick` prop is wrapped in `useCallback` — it only fires if `props.onClick` is defined; otherwise `undefined` is passed to the inner component (no click handler).
- Label computation: `showLabel=false` → `null`; `labelType="percentage"` → `"${calculatePercentage(...)}%"`; `labelType="text"` → `props.labelText?.value`; `labelType="custom"` → `props.customLabel` (ReactNode widget content).

**4. Is it user-facing?**
Not directly — it is the wiring layer. The rendered output comes from `src/components/ProgressBar.tsx`.

**5. What new did you learn from this file?**
The `type` switch determines which set of three props (current/min/max) is used — the other two sets are completely ignored at runtime. Default values for dynamic/expression types come from `defaultValues` (not from the XML), meaning if a dynamic attribute is unset, the bar shows at 50% by default.

---

## src/ProgressBar.xml

**1. What is the purpose of this file?**
Widget definition file declaring all configurable properties and their groupings for Studio and Studio Pro.

**2. What kind of logic is described in this file?**
Property schema: `type` enum (static/dynamic/expression) controls which value props are relevant. Three parallel sets of value props (staticCurrentValue, dynamicCurrentValue, expressionCurrentValue; same for min/max). Label group with `showLabel`, `labelType`, `labelText`, and `customLabel`. Events group with `onClick`.

**3. What part of behavior can be documented from this file?**
- **Type "static":** `staticCurrentValue`, `staticMinValue`, `staticMaxValue` are integers with defaults 50, 0, 100.
- **Type "dynamic":** `dynamicCurrentValue`, `dynamicMinValue`, `dynamicMaxValue` accept Decimal, Integer, or Long entity attributes (all optional).
- **Type "expression":** `expressionCurrentValue`, `expressionMinValue`, `expressionMaxValue` are expressions returning Decimal (all optional).
- **Label types:** `text` (text template), `percentage` (computed), `custom` (widget/ReactNode). The XML note says: "If the Size is set to 'Small', text and percentage labels will be shown in a tooltip and custom labels will be ignored."
- **`onClick`** is optional — the bar becomes clickable only when it is set.
- The widget is `offlineCapable="true"`.
- `studioCategory` and `studioProCategory` are both "Display".

**4. Is it user-facing?**
Yes — defines the configuration interface for Mendix developers in Studio/Studio Pro.

**5. What new did you learn from this file?**
When the progress bar size is "Small" (a Mendix styling concept, not a widget prop), text and percentage labels are shown in a tooltip (`title` attribute) rather than inline, and custom labels are ignored entirely. This behavioral constraint is documented only in the XML description field — it is not enforced by code logic in the widget itself but handled via CSS.

---

## typings/ProgressBarProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript types from ProgressBar.xml. Defines `ProgressBarContainerProps` (runtime) and `ProgressBarPreviewProps` (Studio design mode).

**2. What kind of logic is described in this file?**
Type declarations only. Key types: `TypeEnum` ("static" | "dynamic" | "expression"), `LabelTypeEnum` ("text" | "percentage" | "custom"). Dynamic values are `EditableValue<Big>` (supports write-back); expression values are `DynamicValue<Big>` (read-only).

**3. What part of behavior can be documented from this file?**
- Dynamic value props use `EditableValue<Big>` — this means dynamic values could theoretically be written back to the entity, though the widget only reads from them.
- Expression value props use `DynamicValue<Big>` — read-only computed values.
- `customLabel` is `ReactNode` in container props (a widget slot at runtime).
- `labelText` is `DynamicValue<string>` — a text template resolved at runtime.
- Static value props are plain `number` (integers).

**4. Is it user-facing?**
No — internal TypeScript contract.

**5. What new did you learn from this file?**
Dynamic values are `EditableValue<Big>` — they are entity attributes that the widget can potentially write, unlike expression values which are strictly read-only. However, the widget only reads these values; it does not implement two-way data binding.

---

## src/ProgressBar.editorConfig.ts

**1. What is the purpose of this file?**
Controls Studio/Studio Pro editor behavior: property visibility per type, validation errors, structure preview image, and custom caption.

**2. What kind of logic is described in this file?**
`getProperties`: hides the two irrelevant value prop sets based on `values.type`. Also hides label-related props when `showLabel` is false, and hides `customLabel`/`labelText` based on `labelType`. `check`: validates that dynamic/expression current/min/max props are set (not null). For static type, validates that currentValue is within [minValue, maxValue]. `getPreview`: returns an SVG image (light or dark mode variant) unless `labelType === "custom"` (returns null to let the widget render live). `getCustomCaption`: generates a Studio page explorer caption like `[static, 50]`.

**3. What part of behavior can be documented from this file?**
- When `type` is "dynamic" or "expression": all three value props (current, min, max) are required — missing any generates a validation error.
- When `type` is "static": current value must be ≥ minValue and ≤ maxValue — validation enforced at design time.
- When `showLabel` is false: `labelType`, `labelText`, and `customLabel` props are hidden from the editor.
- When `labelType` is "custom": the structure preview returns `null`, meaning the live preview renders the actual widget instead of an SVG placeholder.
- The custom caption format `[type, currentValue]` appears in Studio's page explorer for easy identification.
- Both light and dark SVG previews exist for the structure preview.

**4. Is it user-facing?**
Yes — visible to developers in Studio/Studio Pro as property panel behavior and validation messages.

**5. What new did you learn from this file?**
When `labelType` is "custom", the editor returns `null` from `getPreview` — this is intentional to allow the custom widget content to render live in design mode. For all other label types, a static SVG is shown. The `getCustomCaption` function makes the widget identifiable in the page explorer by type and current value.

---

## src/ProgressBar.editorPreview.tsx

**1. What is the purpose of this file?**
Renders the live widget preview in Studio's design canvas using the same `ProgressBar` component as the runtime.

**2. What kind of logic is described in this file?**
Mirrors `src/ProgressBar.tsx` but for preview props (all values come as strings in design mode). Uses `asNumber()` helper to parse string values with fallback to `defaultValues`. The `customLabel` renders via `props.customLabel.renderer` (a component type provided by the Mendix framework for widget slots in preview mode).

**3. What part of behavior can be documented from this file?**
- Design mode preview uses the same `ProgressBar` component as runtime — the preview is visually accurate.
- String prop values from design mode are converted to numbers via `asNumber()`; NaN or empty string falls back to default values.
- `onClick` is always `undefined` in preview mode — no click interaction in the design canvas.
- The custom label slot renders via a `renderer` component wrapper — the actual custom widget content is provided by the Mendix Studio framework.

**4. Is it user-facing?**
Visible to Mendix developers in Studio design canvas only.

**5. What new did you learn from this file?**
The preview renders a live `ProgressBar` component (not just a static image) — developers see a real progress bar in the canvas. The `asNumber` helper accepts empty string and NaN gracefully, always falling back to sensible defaults, so the preview never crashes on incomplete configuration.

---

## src/components/ProgressBar.tsx

**1. What is the purpose of this file?**
The pure UI component for the progress bar. Renders the HTML structure, computes percentage width, handles click, shows error alerts for invalid value combinations, and manages the label display.

**2. What kind of logic is described in this file?**
`getValuesErrorMessage`: checks three error conditions — maxValue < minValue, currentValue < minValue, currentValue > maxValue — and returns the first matching error message. The render: outer `.widget-progress-bar.progress-bar-medium.progress-bar-primary` div, inner `.progress` div (clickable if onClick, alert-styled if maxValue < 1), innermost `.progress-bar` div with `width: {percentage}%`. Label: if string, also set as `title` attribute (tooltip for small size). Error: rendered as an `Alert` component with `bootstrapStyle="danger"`.

**3. What part of behavior can be documented from this file?**
- **CSS classes:** `.widget-progress-bar`, `.progress-bar-medium`, `.progress-bar-primary` are always applied. `.widget-progress-bar-clickable` added when onClick is defined. `.widget-progress-bar-alert` added when `maxValue < 1`.
- **Error display:** All three invalid value states (max < min, current < min, current > max) display a danger `Alert` below the bar — the bar still renders (with clamped width) alongside the error.
- **Percentage clamping:** Handled by `calculatePercentage` — values outside [min, max] render as 0% or 100% respectively.
- **Tooltip behavior:** When `label` is a string, it is set as the `title` attribute on the `.progress-bar` div — this provides a tooltip for small-size bars where the label may not be visible inline.
- **Custom label:** When `label` is a ReactNode (non-string), no `title` attribute is set — it renders directly inside the `.progress-bar` div.
- **Non-clickable state:** When `onClick` is undefined, the `.progress` div has no click handler and no `.widget-progress-bar-clickable` class.

**4. Is it user-facing?**
Yes — this is the visible progress bar rendered to end users.

**5. What new did you learn from this file?**
The `.widget-progress-bar-alert` class is applied when `maxValue < 1` (not just when there's an error). This is a distinct visual state for edge-case configurations. The error Alert component renders alongside the bar (not instead of it) — so a misconfigured progress bar shows both a (likely 0% or 100%) bar and a red error message below it.

---

## src/progressBarValues.ts

**1. What is the purpose of this file?**
Defines the `ProgressBarValues` interface and `defaultValues` constant shared between the container and preview components.

**2. What kind of logic is described in this file?**
A simple interface `{currentValue: number, minValue: number, maxValue: number}` and a constant `defaultValues = {currentValue: 50, minValue: 0, maxValue: 100}`.

**3. What part of behavior can be documented from this file?**
Default values when dynamic/expression props are not set: current=50, min=0, max=100 — the bar shows at 50% progress by default with an empty or unavailable attribute. These defaults apply at runtime and in design mode preview.

**4. Is it user-facing?**
Indirectly — determines the default visual state when values are unavailable.

**5. What new did you learn from this file?**
The defaults (50/0/100) are shared between runtime and preview code from a single source of truth. The default currentValue of 50 means an unconfigured dynamic/expression bar shows at 50% rather than 0% — this is intentional for demo/preview purposes.

---

## src/util.ts

**1. What is the purpose of this file?**
Provides `calculatePercentage(currentValue, minValue, maxValue): number` — the core calculation used in both the container, component, and editor preview.

**2. What kind of logic is described in this file?**
Clamps current to [min, max] first: returns 0 if current < min, 100 if current > max. Otherwise: `Math.round(((current - min) / (max - min)) * 100)` wrapped in `Math.abs()`.

**3. What part of behavior can be documented from this file?**
- Percentage is always rounded to the nearest integer — no fractional percentages displayed.
- Values outside the range are clamped to 0 or 100 (no negative percentages, no > 100%).
- `Math.abs()` on the final result means even if the range calculation produces a negative intermediate (e.g., due to negative range), the absolute value is returned — though this case is also caught by the error display.
- The same function is used for percentage label display and for bar width calculation.

**4. Is it user-facing?**
Indirectly — determines the displayed percentage and bar fill width.

**5. What new did you learn from this file?**
The `Math.abs()` wrapper on the percentage result is a defensive measure — it ensures the CSS `width` is never negative even if input validation misses an edge case. Percentage is always an integer (Math.round), so the bar width steps in 1% increments.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history for the progress-bar-web widget, following Keep a Changelog / Semantic Versioning.

**2. What kind of logic is described in this file?**
Version entries from v2.1.0 to v3.2.3.

**3. What part of behavior can be documented from this file?**
- **v3.2.3 (2026-02-10):** Added license file and open-source dependency documentation. No behavioral change.
- **v3.2.2 (2024-08-28):** Changed `action` (onClick) to `required="false"` — prevents unnecessary warnings in Studio Pro when onClick is not configured.
- **v3.2.1 (2023-09-27):** Removed redundant code to improve browser load time. No behavioral change.
- **v3.2.0 (2023-06-05):** Updated page explorer caption to display `[type, current value]`. Updated light/dark icons and tiles.
- **v3.1.0 (2021-12-23):** Added dark mode to structure preview. Added dark icons.
- **v3.0.1 (2021-12-03):** Fixed design properties and styles not applying in Design mode/Studio.
- **v3.0.0 (2021-09-28):** Added toolbox category and tile image.
- **v2.1.0 (2021-07-07):** Added structure preview.

**4. Is it user-facing?**
No — developer/maintainer documentation.

**5. What new did you learn from this file?**
The `onClick` action was changed to `required="false"` in v3.2.2 to avoid spurious Studio Pro warnings — confirming that the clickable behavior is strictly optional and by-design. The custom caption `[type, value]` was added in v3.2.0, introduced alongside icon updates.

---

## e2e/differentViews.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests verifying the progress bar renders correctly in different Mendix container types: group box, data grid listener, list view, template grid, and tab container.

**2. What kind of logic is described in this file?**
Each test navigates to a different page, reads a text box value, and verifies the `.progress-bar` element has matching text content. The tests confirm the widget receives data context from various Mendix containers.

**3. What part of behavior can be documented from this file?**
- Progress bar correctly renders within: group box, data grid (listen-to-grid mode), list view, template grid, and tab container.
- In listen-to-grid mode: clicking a grid row updates the progress bar label — confirming reactive data binding when the widget listens to a selection.
- In tab container: the progress bar in a non-active tab renders correctly after clicking the tab.
- The `.widget-progress-bar.mx-name-progressBar1 .progress-bar` selector confirms the CSS class structure.

**4. Is it user-facing?**
The tested behaviors are user-facing. The test file itself is not.

**5. What new did you learn from this file?**
The "listen to data grid" pattern is explicitly tested — the progress bar can be placed outside a grid and react to grid row selection via Mendix's listen-to-grid data source. The tab container test confirms the widget renders after its parent tab becomes visible (no lazy initialization issue).

---

## e2e/displayText.spec.js

**1. What is the purpose of this file?**
End-to-end tests covering all label type combinations: attribute text, no text, static text, percentage, and raw value.

**2. What kind of logic is described in this file?**
Tests for five display modes: attribute text (dynamic binding), no label (empty text), static text label (text template), percentage label, and value label. Verifies the text content of the progress bar element.

**3. What part of behavior can be documented from this file?**
- **Attribute text:** Dynamic attribute value is displayed as label text — e2e-confirmed binding.
- **No label:** `showLabel=false` results in empty text content.
- **Static text label:** Supports multiple bars with different text template values (`"Static text 1"`, `"Static text 2"`, `"Static text 3"`).
- **Percentage:** Displayed as `"45%"`, `"67%"`, `"0%"` — confirms integer rounding and `%` suffix.
- **Value:** Displayed as `"45"`, `"67"`, `"0"` — confirms raw value display (no `%`).
- Three bars are tested simultaneously in most scenarios, covering different configurations.

**4. Is it user-facing?**
The tested behaviors (label display) are user-facing.

**5. What new did you learn from this file?**
The "display value" label type (showing raw integer, not percentage) is confirmed by e2e tests — this corresponds to `labelType="text"` with `labelText` bound to the current value attribute directly. The test confirms `0%` (not `NaN%` or empty) when progress is at minimum.

---

## e2e/errors.spec.js

**1. What is the purpose of this file?**
End-to-end test covering the "no context" scenario — when the progress bar has a dynamic value source but no data context is available.

**2. What kind of logic is described in this file?**
A single test: navigates to the `p/noContext` page, checks that the progress bar renders with text `"0%"`.

**3. What part of behavior can be documented from this file?**
- When a dynamic progress bar has no data context (e.g., not inside a data view with data), it renders at `0%` rather than crashing or showing an error.
- The widget gracefully handles missing context by falling back to default values.
- The `noContext` scenario is covered by a dedicated test page, confirming this is a known and tested edge case.

**4. Is it user-facing?**
Yes — the no-context fallback behavior is user-facing.

**5. What new did you learn from this file?**
The `0%` fallback in the no-context case suggests that when `dynamicCurrentValue` is undefined/unavailable, the fallback `0` (from `props.dynamicCurrentValue?.value ?? 0` in the container) takes effect, overriding the `defaultValues.currentValue` of 50. The `?? 0` default in `getProgressBarValues` takes precedence over `defaultValues` for the current value.

---

## e2e/onClick.spec.js

**1. What is the purpose of this file?**
End-to-end tests for the onClick action on the progress bar, covering all supported Mendix action types.

**2. What kind of logic is described in this file?**
Five tests, each clicking a progress bar and verifying the triggered action: Microflow (opens modal dialog), Nanoflow (updates a text box), Open Full Page, Open Popup Page, Open Blocking Popup Page. Each test verifies both that the action fired and that the resulting state matches the original progress bar text.

**3. What part of behavior can be documented from this file?**
- **Supported action types (e2e-confirmed):** Microflow, Nanoflow, Open Full Page, Open Popup Page, Open Blocking Popup Page.
- Clicking the progress bar executes the configured action — confirmed for all five action types.
- After a Microflow action: a modal dialog appears containing a progress bar with matching text.
- After a Nanoflow action: a text box is updated with the clicked progress bar's text value.
- The test pattern verifies action execution by checking a resulting UI state change — not just that no error occurred.

**4. Is it user-facing?**
The tested behaviors (click → action) are user-facing.

**5. What new did you learn from this file?**
All five standard Mendix action types (microflow, nanoflow, open full page, open popup, open blocking popup) are e2e-tested for the onClick handler. This is a more comprehensive action coverage than many other widgets. The Nanoflow test uses a `NewEditTextBox` as the result indicator — confirming nanoflows can interact with page state.

---

## src/components/__tests__/ProgressBar.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `ProgressBar` UI component using Jest and React Testing Library. Covers rendering, click behavior, value clamping, range handling, error messages, and label variations.

**2. What kind of logic is described in this file?**
Tests: snapshot for structure, CSS width verification, click handler firing, different ranges (non-zero min), clamping below min (0%), clamping above max (100%), non-clickable state (no onClick). Error Alert tests for all three invalid value conditions. Label tests: static text, ReactNode component, null (no label), tooltip for small-size string labels, no tooltip for small-size custom labels.

**3. What part of behavior can be documented from this file?**
- Progress width CSS is set as `style: { width: "23%" }` on `.progress-bar` element.
- `.widget-progress-bar-clickable` class is present only when `onClick` is provided; absent when `onClick` is undefined.
- Value 23 on range [20, 30] = 30% — confirms relative range calculation.
- Out-of-range values clamp to 0% or 100% AND trigger a danger Alert simultaneously.
- All three error Alert messages are tested: "The maximum value is lower than the minimum value", "The progress value is lower than the minimum value", "The progress value is higher than the maximum value".
- String labels get `title` attribute on `.progress-bar` for tooltip; ReactNode labels do not.
- Null label renders empty content (no text).
- The class `progress-bar-small` is used in tests to simulate "small size" — confirming CSS class is the mechanism for size, not a widget prop.

**4. Is it user-facing?**
No — unit tests only.

**5. What new did you learn from this file?**
The "small size" behavior (label as tooltip) is entirely CSS-class-driven: when `class="progress-bar-small"` is set on the outer div, the label string becomes a `title` attribute on the inner progress bar div. The widget itself applies the title attribute when `label` is a string — the size class is applied externally via Mendix's Design Properties or CSS. Custom label (ReactNode) is ignored in small size because the widget doesn't set `title` for non-string labels.

---

## Summary of Key Behavioral Findings

### Value Sources
- Three mutually exclusive types: `static` (integers from config), `dynamic` (Decimal/Integer/Long entity attributes), `expression` (Decimal expression).
- Default values (50/0/100) apply for dynamic/expression when attributes are unset. But `currentValue` defaults to `0` (not 50) when context is missing — `?? 0` in container vs. `defaultValues` fallback.

### Label Types
- `text`: text template (`DynamicValue<string>`) rendered inline in the bar.
- `percentage`: computed `Math.round((current-min)/(max-min)*100)%`, clamped to [0,100].
- `custom`: arbitrary ReactNode (widget slot) rendered inside the bar.
- `showLabel=false`: no label at all.
- Small size behavior: string labels (text/percentage) become `title` tooltips; custom labels are ignored.

### Error States
- Three runtime error conditions show a danger Alert below the bar: max < min, current < min, current > max.
- Design-time validation: same checks for static type. For dynamic/expression: required fields checked.
- `.widget-progress-bar-alert` class on `.progress` div when `maxValue < 1`.

### Click Interaction
- Optional — requires `onClick` action to be configured.
- Adds `.widget-progress-bar-clickable` CSS class when active.
- Supported action types (e2e-confirmed): Microflow, Nanoflow, Open Full Page, Open Popup Page, Open Blocking Popup Page.

### Layout Contexts (e2e-confirmed)
- Group box, data grid (listen-to-grid), list view, template grid, tab container — all work correctly.

### Percentage Calculation
- Always rounded integer (Math.round).
- Clamped to [0, 100] — no negative or >100% widths.

### Changelog Key Events
- v3.2.2: `onClick` made non-required (no Studio Pro warnings).
- v3.2.0: Custom caption `[type, value]` in page explorer.
- v3.0.1: Design properties fix for Design mode/Studio.
