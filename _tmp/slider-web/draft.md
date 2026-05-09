# slider-web — Draft Spec

Widget: `slider-web`
Package: `packages/pluggableWidgets/slider-web/`
Agent: worker
Date: 2026-05-09

---

## src/Slider.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. Renders a `Container` (the stateful adapter) and a `ValidationAlert` for attribute validation messages.

**2. What kind of logic is described in this file?**
Minimal — wraps `Container` and `ValidationAlert` in a `Fragment`. All logic is delegated to `Container`.

**3. What part of behavior can be documented from this file?**
- `ValidationAlert` renders the string value of `props.valueAttribute.validation` as an error alert directly below the slider. It is visible only when `validation` is a non-empty string — i.e., when Studio Pro validation rules or a `$widget/SetValidation` microflow call set a validation error on the attribute. The alert disappears synchronously when the validation is cleared (no animation). Styling follows Mendix's standard validation pattern (Bootstrap `alert-danger`).
- SCSS (`./ui/Slider.scss`) is imported at this root level — not in a sub-component. This is the correct level for side-effect stylesheet imports in Mendix pluggable widgets.

**4. Is it user-facing?**
Partially — `ValidationAlert` is user-facing (visible error text below the slider); the `Container` wrapper is not.

**5. What new did you learn from this file?**
The Slider entry component is the thinnest in this widget set — only 6 lines of logic. The complexity is entirely in the sub-components and utility hooks. Notably, the SCSS import lives here (not in Container or Slider presentation component), making the root entry file responsible for both the alert and stylesheet side-effects.

---

## src/Slider.xml

**1. What is the purpose of this file?**
Mendix widget descriptor declaring widget properties across three tabs: General, Track, and Events.

**2. What kind of logic is described in this file?**
**General:**
- `valueAttribute` (attribute — Integer/Long/Decimal).
- `advanced` (boolean, default false) — gate for advanced options visibility.
- Min/Max/Step each have 3-way type enum (static/dynamic/expression) + corresponding static (decimal), dynamic (attribute), expression props.
- Tooltip: `showTooltip` (boolean, default true), `tooltipType` (enum: value/customText, default value), `tooltip` (textTemplate, optional), `tooltipAlwaysVisible` (boolean, default false).

**Track:**
- `noOfMarkers` (integer, default 2), `decimalPlaces` (integer, default 0).
- `orientation` (enum: horizontal/vertical, default horizontal).
- `heightUnit`/`height` — only relevant for vertical orientation.

**Events:**
- `onChange` (action, optional).

**3. What part of behavior can be documented from this file?**
- `needsEntityContext="true"`, `offlineCapable="true"`.
- Advanced options (markers, decimal places, orientation, tooltip details) are gated behind `advanced: boolean` toggle.
- Both min and max can be static (design-time decimal), dynamic (entity attribute), or expression (Mendix expression returning Decimal).
- Step size also has the same 3-way type, with the additional constraint: must be > 0, and `(max - min)` should be evenly divisible by step.
- `noOfMarkers = 0` hides markers (they are only visible when > 0).

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
The `advanced` toggle is unique in this set — no other widget uses a single boolean gate to hide a large group of properties. This is a progressive disclosure pattern: simple users only see the essential properties, power users enable "advanced" to see everything.

---

## typings/SliderProps.d.ts (inferred)

**1. What is the purpose of this file?**
Auto-generated TypeScript types. Key types:
- `valueAttribute: EditableValue<Big>`.
- `minValueType: "static" | "dynamic" | "expression"`, `staticMinimumValue?: Big`, `minAttribute?: EditableValue<Big>`, `expressionMinimumValue?: DynamicValue<Big>`.
- Same pattern for max and step.
- `tooltipType: "value" | "customText"`, `tooltip?: DynamicValue<string>` (textTemplate = `DynamicValue<string>`).
- `noOfMarkers: number`, `decimalPlaces: number`, `orientation: "horizontal" | "vertical"`.

**2. What kind of logic is described in this file?**
Type definitions only — provides type-safe access to all XML-declared props.

**3. What part of behavior can be documented from this file?**
- `tooltip` is a `DynamicValue<string>` (textTemplate) — can include entity attribute values.
- Static values (min/max/step) are `Big` (not number), consistent with Mendix's decimal type system.

**4. Is it user-facing?**
No — TypeScript types only.

**5. What new did you learn from this file?**
The three-way union type pattern (static = `Big`, dynamic = `EditableValue<Big>`, expression = `DynamicValue<Big>`) is resolved in `prop-utils.ts` before reaching the slider component.

---

## src/components/Container.tsx

**1. What is the purpose of this file?**
The main adapter component. Resolves dynamic/expression/static min/max/step values, handles loading state, and wires up the `Slider` presentation component with all resolved props.

**2. What kind of logic is described in this file?**
- `useNumber(minProp(props))`, `useNumber(maxProp(props))`, `useNumber(stepProp(props))` — each returns `{ loading: true }` or `{ loading: false, value: number | undefined }`.
- If any of min/max/step or `valueAttribute` is still loading, renders a disabled `<Slider>` as a placeholder.
- Once loaded, renders `InnerContainer` with resolved numeric values.
- `InnerContainer`: creates `handleRender` (tooltip), `onChange` (debounced), `marks` (via `useMarks`), reads `ariaLabelledByForHandle` from document (label lookup).
- `disabled` ← `props.valueAttribute.readOnly`.
- `value` ← `props.valueAttribute.value?.toNumber()`.

**3. What part of behavior can be documented from this file?**
- The slider renders disabled (greyed out, `cursor: not-allowed`) while values are still loading.
- Value is read from `valueAttribute.value?.toNumber()` — returns `undefined` when the attribute has no value. When `value` is `undefined`, `@rc-component/slider` defaults the handle position to `min`. This is confirmed by E2E test: without entity context, `left: 0%` when min=0. **Value initialization**: there is no explicit default or clamping — the slider inherits rc-slider's behavior of treating `undefined` as equivalent to `min`.
- Label association: `ariaLabelledByForHandle` is set to `${props.id}-label` only when a `<label for="${id}">` element exists in the DOM. Note: `ariaLabelledByForHandle` is always computed (line `const ariaLabelledByForHandle = \`${props.id}-label\``), but only passed to the slider when `hasLabel === true`.
- Tooltip is only created when `showTooltip === true`.

**4. Is it user-facing?**
No — internal adapter component.

**5. What new did you learn from this file?**
The loading-disabled pattern ensures the slider doesn't flash in a broken state (wrong min/max) before the dynamic values are fetched. The `useNumber` hook prevents this by tracking load state separately from the value.

---

## src/components/Slider.tsx

**1. What is the purpose of this file?**
The pure presentation component that wraps `@rc-component/slider` with Mendix-specific CSS classes.

**2. What kind of logic is described in this file?**
- Renders a `<div>` with `className="widget-slider"` + optional `widget-slider-vertical` when `vertical` is true.
- Wraps `RcSlider` from `@rc-component/slider` with all passed props.
- Uses `forwardRef` to expose the container `<div>` ref to parent (`Container` uses it for tooltip positioning).

**3. What part of behavior can be documented from this file?**
- The CSS class `widget-slider` is on the container `<div>`, not the `rc-slider` directly.
- `widget-slider-vertical` adds vertical layout CSS.
- All slider behavior (range, step, marks, dragging, keyboard) is provided by `@rc-component/slider`.

**4. Is it user-facing?**
Yes — this is the visible slider component.

**5. What new did you learn from this file?**
The ref is on the outer `<div>` (not the `rc-slider` element). This is used by `createHandleRender` to set `getTooltipContainer` — the tooltip is rendered inside the `widget-slider` div rather than `document.body`, which prevents tooltip clipping by scrollable containers.

---

## src/utils/prop-utils.ts

**1. What is the purpose of this file?**
Utility functions for prop resolution: `minProp`, `maxProp`, `stepProp` (type routing), `isVertical`, `getStyleProp`.

**2. What kind of logic is described in this file?**
- `minProp`/`maxProp`/`stepProp`: 3-way switch on type enum → returns the corresponding prop value (Big/EditableValue<Big>/DynamicValue<Big>).
- `isVertical`: `orientation === "vertical"`.
- `getStyleProp`: only sets `height` CSS when `orientation === "vertical"` (horizontal sliders don't need explicit height).

**3. What part of behavior can be documented from this file?**
- Horizontal sliders return `undefined` style prop (no inline height).
- Vertical slider height supports both pixels and percentage.

**4. Is it user-facing?**
No — internal utilities.

**5. What new did you learn from this file?**
The prop routing pattern (type enum → appropriate prop) cleanly isolates the "which source of truth to use" logic into one place. All three value types flow through `useNumber` uniformly after this routing.

---

## src/utils/useNumber.ts

**1. What is the purpose of this file?**
A hook that resolves a union type (Big | EditableValue<Big> | DynamicValue<Big> | undefined) into a stable `{ loading, value }` result with a "loaded once" latch.

**2. What kind of logic is described in this file?**
- `loading()`: returns `true` only when a Mendix value has `status === "loading"` (not for `Big` instances or `undefined`).
- `value()`: converts to `number` via `.toNumber()` when available; returns `undefined` otherwise.
- **Latch**: `useRef(false).current ||= !loading(prop)` — once loaded, `isLoaded` stays `true` even if the prop later becomes undefined or re-loading. This prevents the slider from reverting to loading state.

**3. What part of behavior can be documented from this file?**
- Static `Big` values never trigger loading state.
- Undefined props never trigger loading state (loading = false, value = undefined).
- The latch ensures that once a dynamic/expression value loads, subsequent re-renders don't re-enter the loading spinner.

**4. Is it user-facing?**
No — internal hook.

**5. What new did you learn from this file?**
The `useRef(false).current ||= !loading(prop)` idiom is an unconventional but compact way to implement a "set once" boolean ref. The `||=` operator: if `ref.current` is already `true`, stays `true`; if `false`, checks `!loading(prop)`. This avoids a full `useEffect` and `setState` pattern.

---

## src/utils/useOnChangeDebounced.ts

**1. What is the purpose of this file?**
A hook that debounces the Mendix `onChange` action by 500ms while immediately updating the attribute value on each slider drag step.

**2. What kind of logic is described in this file?**
- `onChange`: called on every drag step → immediately calls `valueAttribute.setValue(new Big(value))`, then schedules `onChangeEnd` (debounced 500ms).
- `onChangeEnd`: calls `executeAction(onChange)` after 500ms of inactivity.
- Cleanup: `abort` is called on unmount to cancel any pending debounce.

**3. What part of behavior can be documented from this file?**
- Attribute is updated in real-time during dragging (no debounce on `setValue`).
- The `onChange` Mendix action fires 500ms after the last drag step — so it fires once per drag gesture, not on every intermediate value.
- If the component unmounts during a drag (e.g., navigation), the pending `onChange` action is cancelled.

**4. Is it user-facing?**
No — internal hook.

**5. What new did you learn from this file?**
The separation between `setValue` (immediate) and `executeAction(onChange)` (debounced) is intentional — the attribute value is always up-to-date for reactive use, but the action fires only when the user has finished moving the slider. This matches the expected behavior for server-side `onChange` microflows.

---

## src/utils/marks.ts + src/utils/useMarks.ts

**1. What is the purpose of these files?**
`marks.ts` creates the marks record for the rc-slider; `useMarks.ts` memoizes it.

**2. What kind of logic is described in these files?**
- `createMarks`: generates `(numberOfMarks + 1)` evenly spaced marks from `min` to `max` (inclusive), each formatted to `decimalPlaces` decimal places. `interval = (max - min) / numberOfMarks`.
- Marks are keyed by numeric value and labeled with the value string.
- Returns `undefined` when `numberOfMarks <= 0` or `min >= max` (no marks to show).
- `useMarks`: wraps `createMarks` in `useMemo` with `[min, max, noOfMarkers, decimalPlaces]` deps.

**3. What part of behavior can be documented from this file?**
- `noOfMarkers = 2` (default) creates 3 marks: at min, midpoint, and max.
- `noOfMarkers = 0` → no marks rendered.
- `decimalPlaces = 0` → mark labels are integers; `decimalPlaces = 2` → "0.00", "50.00", "100.00".
- Marks are not valid when `min >= max` (which matches the validation check in `editorConfig.ts`).
- **Marks are independent of `step`**: mark positions are computed from `(max - min) / numberOfMarks` — they do not snap to step boundaries. When `(max - min)` is not evenly divisible by `step`, marks may appear at positions the user cannot actually reach (rc-slider snaps the handle to step-aligned values). Example: min=0, max=10, step=3, noOfMarkers=2 → marks at 0, 5, 10 but valid handle positions are only 0, 3, 6, 9. The mark at position 5 is a visual label that the handle cannot land on. `editorConfig.ts` validates that step > 0 but does NOT validate mark-step alignment.

**4. Is it user-facing?**
Partially — the mark labels are visible tick labels on the slider track.

**5. What new did you learn from these files?**
The `noOfMarkers` prop is the "number of intervals" (marks between), not the "number of marks" (labels). `createMarks` iterates `0..numberOfMarks` (inclusive) producing `numberOfMarks + 1` labels. So `noOfMarkers = 2` → 3 labels (min, mid, max). Mark positions are purely arithmetic (min + i × interval), making them independent of step — a design tradeoff that keeps mark logic simple at the cost of potential label/handle misalignment.

---

## src/utils/createHandleRender.tsx

**1. What is the purpose of this file?**
Creates a custom `handleRender` function for `@rc-component/slider` that wraps the slider handle in a `RcTooltip`.

**2. What kind of logic is described in this file?**
- `RcTooltip` from `@rc-component/tooltip` wraps each handle node.
- `overlay`: for `"customText"` type → `<div>{tooltip?.value ?? ""}</div>`; for `"value"` type → `restProps.value` (the numeric handle value).
- `visible={tooltipAlwaysVisible || dragging}` — tooltip visibility is **fully controlled** by this expression; `trigger` events do not override it.
- `trigger={["hover", "click", "focus"]}` — present but functionally overridden by the controlled `visible` prop; the tooltip does NOT independently show/hide on hover when `visible` is explicitly set.
- `getTooltipContainer={() => sliderRef.current ?? document.body}` — tooltip rendered inside the slider's container div.
- `mouseLeaveDelay={0}` — tooltip hides immediately when `visible` transitions to false (no animation linger).

**3. What part of behavior can be documented from this file?**
- **`dragging` prop origin**: `@rc-component/slider` passes `dragging: boolean` into the `handleRender` callback for each handle. It is `true` exactly while the user is actively dragging that handle, `false` otherwise. No React state in the Mendix component is needed to track drag state.
- **Tooltip visibility transitions**: When `tooltipAlwaysVisible = false` — tooltip appears immediately when drag starts (`dragging` → `true`) and disappears immediately when drag ends (`dragging` → `false`, `mouseLeaveDelay=0`). There is no fade or delay on either transition.
- When `tooltipAlwaysVisible = true`, tooltip is always visible regardless of drag or interaction state.
- Custom tooltip value comes from a `DynamicValue<string>` textTemplate — can reference entity attributes. When the textTemplate has no value yet (unavailable), overlay shows an empty string (`tooltip?.value ?? ""`).
- Tooltip is anchored to the slider container (not `document.body`), preventing misposition when the slider is inside a scrollable container. Falls back to `document.body` if `sliderRef.current` is null.

**4. Is it user-facing?**
Yes — the tooltip is the visible value label shown while dragging (or always, if configured).

**5. What new did you learn from this file?**
Tooltip visibility is a controlled boolean (`visible={tooltipAlwaysVisible || dragging}`) driven entirely by `rc-slider`'s `dragging` callback prop — no React state or event handlers needed. The `trigger` array and `defaultVisible=true` props are effectively inert: `trigger` is bypassed when `visible` is controlled, and `defaultVisible` is ignored once `visible` is provided. The `@rc-component/tooltip` bootstrap CSS is imported here as a side-effect.

---

## src/utils/helpers.ts

**1. What is the purpose of this file?**
Single utility: `getSliderLabel` — looks up the label element in the DOM by the Mendix system label convention.

**2. What kind of logic is described in this file?**
`getSliderLabel(sliderId)`: `document.querySelector(`label[for="${sliderId}"]`)`.

**3. What part of behavior can be documented from this file?**
- The Label system property generates `<label for="{widgetId}">`. This function checks if that label exists in DOM.
- When found, `ariaLabelledByForHandle` is set to `${widgetId}-label`, linking the slider handle to the label for accessibility.

**4. Is it user-facing?**
No — internal accessibility utility.

**5. What new did you learn from this file?**
The Mendix Label system property generates a `<label>` element with `for="{widgetId}"` in the DOM, not `for="{widgetId}-label"`. The `ariaLabelledByForHandle` is set to `${id}-label` which references the `<label id="{id}-label">` element that Mendix places alongside the label. The fix in v3.0.2 (CHANGELOG) made this work correctly.

---

## src/utils/getPreviewValues.ts

**1. What is the purpose of this file?**
Computes preview values (min, max, step, value) for the Studio Pro editor preview.

**2. What kind of logic is described in this file?**
- Uses static values when `*ValueType === "static"`; defaults to 0/100/1 for dynamic/expression types (values not available in Studio Pro).
- `value = max - (max - min) / 2` — always the midpoint, so the preview handle appears centered.

**3. What part of behavior can be documented from this file?**
- Studio Pro preview always shows the slider at 50% of the configured range.
- Dynamic and expression min/max/step show as 0/100/1 in preview.

**4. Is it user-facing?**
No — Studio Pro preview utility.

**5. What new did you learn from this file?**
The midpoint formula `max - (max - min) / 2` equals `(max + min) / 2` — mathematically identical but written to avoid potential floating-point issues when operating from the max end.

---

## src/Slider.editorConfig.ts

**1. What is the purpose of this file?**
Studio Pro property visibility rules (`getProperties`), validation checks (`check`), structure preview (`getPreview`), and custom caption (`getCustomCaption`).

**2. What kind of logic is described in this file?**
- `getProperties`:
  - Hides irrelevant min/max/step props per type (static hides dynamic+expression, etc.).
  - Hides tooltip/type when `showTooltip = false`; hides `tooltip` when `tooltipType = "value"`.
  - Hides height props when `orientation = "horizontal"`.
  - Web platform: hides all `advancedOptionKeys` when `!advanced`; calls `transformGroupsIntoTabs`.
  - Desktop platform: hides `advanced` prop.
- `check`: 6 validators — tooltip emptiness, min < max, missing attribute/expression for dynamic/expression types, step > 0, decimal places >= 0.
- `getPreview`: returns a static SVG image (300×28px, light/dark), not a live slider.
- `getCustomCaption`: returns `values.valueAttribute` or `"Slider"`.

**3. What part of behavior can be documented from this file?**
- `advanced` is a web-only concept — desktop Studio Pro users see all options unconditionally.
- `transformGroupsIntoTabs` converts property groups into tabs in the Studio Pro panel (web platform only).
- The structure preview is a static 28px-tall SVG (very compact), unlike other widgets.
- **min > max constraint interactions at runtime**: Studio Pro's `check()` validator enforces `min < max` for static values at design time, preventing invalid configuration in Studio Pro. However, when min/max are dynamic (attribute) or expression values, validation only happens at runtime. When `min >= max` at runtime: (1) `isParamsValidToCalcMarks` returns false → marks are not rendered (no crash); (2) `@rc-component/slider` receives an invalid range and renders in a visually broken state (the track may collapse or disappear); (3) no JavaScript error is thrown; (4) the slider value is not clamped (rc-slider does not guarantee behavior outside the min/max range when range itself is inverted). The design relies on the developer ensuring valid dynamic min/max values — there is no runtime guard beyond the marks suppression.

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
The `platform: "web" | "desktop"` parameter in `getProperties` reveals this widget has separate handling for Mendix Studio (web) vs Studio Pro (desktop). The `advanced` toggle only appears in the web platform UI. The constraint validation system (`check()`) provides strong design-time safety for static values but provides no runtime safety for dynamic/expression min/max values — the responsibility falls entirely on the Mendix developer to ensure valid ranges.

---

## src/Slider.editorPreview.tsx

**1. What is the purpose of this file?**
Renders a live `Slider` component (using `@rc-component/slider`) in the Studio Pro design-mode preview.

**2. What kind of logic is described in this file?**
- Calls `getPreviewValues` to compute min/max/value/step.
- Creates marks via `createMarks` with `noOfMarkers ?? 2`, `decimalPlaces ?? 2`.
- Passes `onChange={undefined}` (non-interactive preview), `disabled={props.readOnly}`.

**3. What part of behavior can be documented from this file?**
- The preview shows a real, rendered slider (not a static SVG) — differs from the structure preview in `getPreview()` (which returns a static SVG).
- **decimalPlaces discrepancy — behavioral impact**: `createMarks` is called with `props.decimalPlaces ?? 2`, but the XML default is `0`. When a developer does not configure `decimalPlaces` (accepting the default of 0), `props.decimalPlaces` in preview context is `undefined`, so `?? 2` applies. Result: the Studio Pro design preview shows mark labels as "0.00", "50.00", "100.00" while the live widget renders "0", "50", "100". This is a cosmetic-only discrepancy (no runtime behavior change), but it can mislead a developer into thinking the widget defaults to 2 decimal places, or cause confusion when verifying the widget appearance in Studio Pro. There is no impact on the live Mendix client rendering.

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
The `decimalPlaces ?? 2` fallback creates a documented discrepancy between Studio Pro preview and runtime behavior: the preview always shows at least 2 decimal places on mark labels, while the configured default of `decimalPlaces = 0` produces integer labels at runtime. This is a minor but confirmed inconsistency in the widget's preview fidelity.

---

## src/components/__tests__/Slider.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `Slider` presentation component — rendering and keyboard interaction.

**2. What kind of logic is described in this file?**
- Snapshot tests for horizontal and vertical layouts.
- `aria-valuenow` contains the correct value.
- Keyboard: ArrowUp → increases by step (−100 + 10 = −90); ArrowLeft → decreases (−90 − ... wait — ArrowLeft with value=−90 → −100); ArrowRight → increases back.
- Marks: renders labels for each mark value; snapshot of marks structure.

**3. What part of behavior can be documented from this file?**
- Confirmed: `role="slider"` on the handle, `aria-valuenow` = current value.
- Confirmed: ArrowUp/ArrowRight increase by step; ArrowLeft/ArrowDown decrease by step.
- Test setup: `min=-100, max=100, step=10`, `ariaLabelledByForHandle="test-slider"`.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The test verifies `aria-valuenow` via the handle's `getAttribute`, confirming that `@rc-component/slider` provides proper ARIA attributes on the handle element. This is the key accessibility attribute for sliders per ARIA spec.

---

## e2e/Slider.spec.js

**1. What is the purpose of this file?**
Playwright E2E tests verifying the slider renders correctly with and without context (entity object).

**2. What kind of logic is described in this file?**
- **With context**: verifies min/max marker text matches textbox values; verifies handle is at 50% (`left: 50%; transform: translateX(-50%)`); value input shows "10".
- **Without context** (`/p/no-context`): slider is disabled (`rc-slider-disabled` class); markers show 0/100 defaults; handle at 0% (`left: 0%`); cursor is `not-allowed`.

**3. What part of behavior can be documented from this file?**
- Confirmed: without entity context → slider is disabled with `cursor: not-allowed`.
- Confirmed: with context → handle position reflects the entity value via `left: X%` style.
- The test app uses `textBoxMinimumValue` / `textBoxMaximumValue` to dynamically configure min/max via the dynamic value binding.

**4. Is it user-facing?**
No — test file.

**5. What new did you learn from this file?**
The `left: 0%` handle position when no value is set (no-context) confirms that `undefined` value maps to the minimum position. This aligns with `valueAttribute.value?.toNumber()` returning `undefined` and `rc-slider` defaulting to `min` when value is undefined.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history since v2.0.1.

**2. What kind of logic is described in this file?**
- **v3.0.2 (2026-02-19)**: Fixed accessibility — slider handle not linked to label (fix contributed by external contributor @DiljohnSingh).
- **v3.0.1 (2026-02-10)**: Added license/readme for OSS deps.
- **v3.0.0 (2026-01-06)**: Updated rc-slider to support React 19.
- **v2.1.4 (2024-08-28)**: Changed `onChange` to not required.
- **v2.1.3 (2024-07-15)**: Fixed initial value being reset to min on load.
- **v2.1.2 (2024-06-25)**: Fixed tooltip position when page is scrolled.
- **v2.1.0 (2023-06-06)**: Updated caption to display datasource.

**3. What part of behavior can be documented from this file?**
- The accessibility fix (v3.0.2) added the label-to-handle ARIA link (`ariaLabelledByForHandle`).
- The v2.1.3 fix for "initial value reset to default min" corresponds to the `useNumber` latch mechanism.
- The tooltip scroll fix (v2.1.2) corresponds to the `getTooltipContainer` → `sliderRef.current` pattern.

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
The tooltip scroll fix (v2.1.2) and the label accessibility fix (v3.0.2) both have direct corresponding code implementations visible in the source. The CHANGELOG confirms these fixes are intentional, not accidental patterns.

---

## Summary of Key Findings

- **Core library**: Built on `@rc-component/slider` (the successor to `rc-slider`) — provides all drag/keyboard behavior, ARIA, and marks.
- **Three-way value sources**: Min, max, step all support static (decimal), dynamic (entity attribute), or expression (Mendix expression) modes.
- **Loading latch**: `useNumber` prevents re-entering loading state once values are resolved (fixes v2.1.3 reset bug).
- **Debounced onChange**: Attribute updated immediately on every drag step (500ms debounce only for the `onChange` Mendix action).
- **Value initialization**: When `valueAttribute` has no value (undefined), rc-slider positions the handle at `min`. Confirmed by E2E test (left: 0% when min=0, no entity context).
- **Marks**: Generated at `(numberOfMarks + 1)` evenly spaced points; `noOfMarkers = 0` disables marks. Marks are independent of `step` — mark positions may not align with step-snapped handle positions when `(max - min)` is not evenly divisible by step.
- **ValidationAlert**: Renders Mendix attribute validation text below the slider when `valueAttribute.validation` is set. Appears/disappears synchronously; styled as Bootstrap `alert-danger`.
- **Tooltip**: Fully controlled by `visible={tooltipAlwaysVisible || dragging}`. The `dragging` boolean comes from rc-slider's handleRender callback — no React state needed. Tooltip appears immediately on drag start, disappears immediately on drag end (`mouseLeaveDelay=0`). Anchored to slider container div to prevent scroll clipping.
- **Accessibility**: `role="slider"`, `aria-valuenow`, `ariaLabelledByForHandle` linked to Mendix Label system property.
- **Vertical mode**: Supported; requires explicit height on slider or parent.
- **Advanced gate**: Web platform only — hides advanced props behind a single boolean toggle.
- **offlineCapable**: `true`.
- **min > max at runtime**: Marks suppressed (no crash), but rc-slider renders in a visually broken state. No runtime guard beyond mark suppression. Design-time check in `editorConfig.ts` only covers static values.
- **Preview decimalPlaces discrepancy**: Studio Pro design preview always shows marks with ≥2 decimal places (`?? 2` fallback), while the runtime default is 0 decimal places. Cosmetic only — does not affect live widget behavior.
