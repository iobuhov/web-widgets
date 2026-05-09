# Draft: range-slider-web

Widget package: `packages/pluggableWidgets/range-slider-web`

---

## src/RangeSlider.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, props schema, and Studio Pro categorization. Generates `RangeSliderProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares two required data attributes (`lowerBoundAttribute`, `upperBoundAttribute`) for the two slider handles. Each of min, max, and step has a "value type" enumeration (static/dynamic/expression) plus three sub-properties for the three modes. Declares tooltip group: `showTooltip`, `tooltipTypeLower/Upper` (value or custom text), `tooltipAlwaysVisible`. Declares marker group: `showMarkers`, `noOfMarkers`, `decimalPlaces`. Declares advanced group: `stepSizeType`, `orientation`, `heightUnit`, `height`, `onChange`. System properties: editability, visibility.

**3. What part of behavior can be documented from this file?**
- The widget binds two Mendix attributes (lower and upper bound) — both are required (`required="true"`).
- Min, max, and step can each independently be sourced as a static literal, a data attribute (`EditableValue<Big>`), or an expression (`DynamicValue<Big>`).
- Tooltip can show the current numeric value or a custom text template, and can be configured to always be visible (not just on hover).
- Markers are evenly spaced across the track; `noOfMarkers` controls count, `decimalPlaces` controls label formatting.
- Orientation supports horizontal (default) and vertical layouts; vertical mode enables `heightUnit` (px or %) and `height` properties.
- Widget requires entity context (`needsEntityContext="true"`), is a pluggable widget, and is NOT marked offline capable (unlike progress-bar-web and progress-circle-web).
- `onChange` action fires after value changes.

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The range slider is NOT offline capable — unlike progress-bar-web and progress-circle-web which are both `offlineCapable="true"`. This is likely because the slider requires persistent attribute writes, which may not be fully supported in offline Mendix apps.

---

## src/RangeSlider.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. It wraps the internal `Container` component and handles Mendix validation alert rendering.

**2. What kind of logic is described in this file?**
Renders validation alerts for both `lowerBoundAttribute` and `upperBoundAttribute` using their `.validation` property. Delegates all other rendering to `Container`. Imports and applies `RangeSlider.scss`.

**3. What part of behavior can be documented from this file?**
- Validation errors for both the lower and upper bound attributes are displayed below the slider UI.
- The widget uses a three-layer pattern: `RangeSlider` (validation wrapper) → `Container` (logic) → `RangeSliderComponent` (presentation).
- There is no loading state handled at this level — that is handled inside `Container`.

**4. Is it user-facing?**
Yes — validation error messages are user-visible.

**5. What new did you learn from this file?**
The separation of validation alert rendering from the logic layer means validation messages appear consistently regardless of slider state, and the `Container` component does not need to know about Mendix validation APIs.

---

## src/components/Container.tsx

**1. What is the purpose of this file?**
The main logic/orchestration layer. Resolves value types (static/dynamic/expression) for min, max, and step; manages loading state; debounces onChange; and computes marks and slider values.

**2. What kind of logic is described in this file?**
Uses `useNumber(minProp(...))`, `useNumber(maxProp(...))`, `useNumber(stepProp(...))` hooks to resolve values from three possible source types. Shows a disabled slider while any of min, max, step, or bound attributes are loading. Converts `lowerBoundAttribute.value` and `upperBoundAttribute.value` (Big.js) to numbers. Computes `readOnly` from either attribute being read-only. Generates marks via `useMarks(noOfMarkers, decimalPlaces, min, max)`. Debounces onChange via `useOnChangeDebounced`. Passes `allowCross={false}` to prevent handles from overlapping.

**3. What part of behavior can be documented from this file?**
- The slider is disabled (non-interactive) while any of the min, max, step, or bound values are still loading from Mendix.
- `allowCross={false}` enforces that the lower handle cannot exceed the upper handle (and vice versa).
- Debounce delay is 500 ms — value changes during dragging are batched and emitted at most once per 500 ms.
- Both attribute values must be available before the slider becomes interactive.
- If bound attribute values are unavailable, the slider falls back to min (for lower) and max (for upper) as defaults.
- Tooltip rendering is handled per-handle via `HandleTooltip` component refs passed as `handleRender` callbacks.

**4. Is it user-facing?**
No — internal orchestration layer.

**5. What new did you learn from this file?**
The `handleRender` ref-callback pattern allows injecting `HandleTooltip` per handle while keeping the `RangeSliderComponent` unaware of tooltip logic. Each handle gets its own tooltip instance with independent value and type configuration.

---

## src/components/RangeSlider.tsx

**1. What is the purpose of this file?**
The presentation-layer React component wrapping `@rc-component/slider`'s `Slider` in range mode.

**2. What kind of logic is described in this file?**
Forwards a ref to the root `<div>`. Merges `"widget-range-slider"` base class (plus `"widget-range-slider-vertical"` when vertical) with any custom `className`. Applies a root style object (`rootStyle`) for vertical height. Passes all other props through to `@rc-component/slider`'s `Slider`.

**3. What part of behavior can be documented from this file?**
- Base CSS class: `"widget-range-slider"`.
- Vertical CSS class: `"widget-range-slider-vertical"`.
- The component is a minimal wrapper — it adds CSS context but does not modify slider behavior.
- All `@rc-component/slider` props (min, max, step, value, onChange, marks, disabled, allowCross, handleRender) are passed through via spread.

**4. Is it user-facing?**
No — internal DOM structure.

**5. What new did you learn from this file?**
The widget uses `@rc-component/slider` (v1.0.1) rather than the older `rc-slider` — this was a breaking change introduced in v3.0.0 (CHANGELOG). The scoped `@rc-component/` namespace indicates the newer, actively maintained fork.

---

## src/components/TooltipHandler.tsx

**1. What is the purpose of this file?**
Renders an `@rc-component/tooltip` wrapper around each slider handle, with conditional visibility based on dragging state and configuration.

**2. What kind of logic is described in this file?**
Returns `null` early when `showTooltip=false` or the slider ref is not yet set. Renders `<Tooltip>` portaled into the slider root div. Tooltip content: when `tooltipType="value"`, displays the numeric value; when `tooltipType="custom"`, displays the template text. Visibility: when `tooltipAlwaysVisible=true`, always shown; otherwise shown only when `dragging=true`. Triggers on hover, click, and focus. Mouse leave delay is 0 ms.

**3. What part of behavior can be documented from this file?**
- Tooltip visibility modes: always visible, or only while dragging (not on hover unless dragging).
- Tooltip renders into the slider's root div as a portal (preventing overflow clipping issues).
- Two tooltip content modes: automatic numeric value display, or custom text template.
- Tooltip hides immediately on mouse leave (0 ms delay).
- The `key` prop includes the current value — forcing tooltip re-render on value change to ensure correct positioning.

**4. Is it user-facing?**
Yes — tooltip text is visible to end users during interaction.

**5. What new did you learn from this file?**
The tooltip is conditionally shown during dragging only (not on hover) when `tooltipAlwaysVisible=false`. This is a deliberate UX choice — tooltips appear as feedback during dragging, not as persistent labels, reducing visual clutter.

---

## src/utils/useOnChangeDebounced.ts

**1. What is the purpose of this file?**
Custom hook providing a debounced onChange handler that writes new slider values to Mendix attributes and optionally fires the `onChange` action.

**2. What kind of logic is described in this file?**
Creates a 500 ms debounced function (via `@mendix/widget-plugin-platform/utils/debounce`). On each call: converts `[lower, upper]` numbers to `Big` instances, sets `lowerBoundAttribute.setValue(Big(lower))` and `upperBoundAttribute.setValue(Big(upper))`, then executes `onChange?.execute()`. Returns a memoized `useCallback` that depends on `onChange`.

**3. What part of behavior can be documented from this file?**
- Attribute values are updated with Big.js instances to maintain decimal precision.
- The `onChange` action is debounced — it fires 500 ms after the last drag event, not on every pixel move.
- Attribute value writes happen inside the debounced function (also debounced at 500 ms).
- The hook returns a stable callback reference that only changes when the `onChange` prop changes.

**4. Is it user-facing?**
No — internal data binding logic. Affects responsiveness of attribute updates.

**5. What new did you learn from this file?**
Both the attribute writes AND the action trigger are inside the debounced function — so attribute values are not updated on every frame during dragging, only 500 ms after dragging stops. This differs from a pattern where attribute writes would be immediate and only the action would be debounced.

---

## src/utils/useMarks.ts

**1. What is the purpose of this file?**
Custom hook memoizing slider mark generation.

**2. What kind of logic is described in this file?**
Wraps `createMarks(noOfMarkers, decimalPlaces, min, max)` in a `useMemo` keyed on all four parameters. Returns `undefined` if `createMarks` returns `undefined` (invalid configuration).

**3. What part of behavior can be documented from this file?**
- Marks are recalculated only when `noOfMarkers`, `decimalPlaces`, `min`, or `max` change.
- Default values: `min=0`, `max=100`.
- Returns `undefined` when markers cannot be generated (e.g., `noOfMarkers=0` or `min >= max`).

**4. Is it user-facing?**
No — internal memoization.

**5. What new did you learn from this file?**
The hook's default values (`min=0, max=100`) mean marks are always generated even before the actual min/max values load, using placeholder range. This could cause a brief flash of wrong marks on initial render.

---

## src/utils/useNumber.ts

**1. What is the purpose of this file?**
Custom hook for resolving a number from a static `Big`, `EditableValue<Big>`, or `DynamicValue<Big>`, with loading state tracking.

**2. What kind of logic is described in this file?**
Uses a `useRef<boolean>` to track whether the value has ever been loaded. Once the status transitions to "available" and the value is non-null, `loaded.current` is set to `true` — after which the hook never re-enters "loading" state. Returns `{ value: number | undefined, isLoading: boolean }`.

**3. What part of behavior can be documented from this file?**
- The loading gate is one-way: once loaded, the hook never reports loading again (even if the attribute becomes unavailable later).
- For `EditableValue`, value is extracted via `.value?.toNumber()`.
- For `DynamicValue`, value is extracted when `.status === "available"`.
- For raw `Big` instances, the value is immediately available and loading is never true.

**4. Is it user-facing?**
No — internal data type abstraction.

**5. What new did you learn from this file?**
The ref-based one-way loading gate means that if a dynamic min/max value becomes unavailable AFTER initial load (e.g., due to a page state change), the slider will still show the last known values rather than re-entering a disabled/loading state. This is a deliberate stability trade-off.

---

## src/utils/prop-utils.ts

**1. What is the purpose of this file?**
Utility functions that select the correct prop based on `MinValueTypeEnum`, `MaxValueTypeEnum`, `StepSizeTypeEnum`, and `OrientationEnum`.

**2. What kind of logic is described in this file?**
`minProp(props)`: switches on `props.minValueType` to return `props.staticMinimumValue`, `props.minAttribute`, or `props.expressionMinimumValue`. `maxProp(props)` and `stepProp(props)` follow the same pattern for their enums. `isVertical(props)`: returns `props.orientation === "vertical"`. `getStyleProp(props)`: returns `{ height: "${value}${unit}" }` only when vertical, otherwise `{}`.

**3. What part of behavior can be documented from this file?**
- Three source types per range parameter: static (literal), dynamic (entity attribute), expression (formula).
- Height style is only applied for vertical orientation — horizontal sliders have no explicit height set.
- The height unit can be `"px"` or `"%"` as configured in Studio.

**4. Is it user-facing?**
No — internal prop routing.

**5. What new did you learn from this file?**
This pattern of `minProp/maxProp/stepProp` selectors mirrors the value resolution pattern in progress-bar-web and progress-circle-web, but applied to three parameters each with three modes — 9 total combinations. The selector pattern keeps Container.tsx clean.

---

## src/utils/getPreviewValues.ts

**1. What is the purpose of this file?**
Extracts display values for the Studio design-mode preview, using deterministic static values.

**2. What kind of logic is described in this file?**
For static `minValueType`, reads `staticMinimumValue` (default 0). For dynamic/expression types, defaults to 0. Similarly for max (default 100) and step (default 1). Computes preview slider values as `[min + (max - min) * 0.25, min + (max - min) * 0.75]` — 25% and 75% of the range.

**3. What part of behavior can be documented from this file?**
- In Studio design mode, the slider shows handles at 25% and 75% of the configured range.
- Only static-type min/max/step values are used for the preview; dynamic and expression types default to 0/100/1.
- Preview is always deterministic — it does not wait for Mendix attribute values.

**4. Is it user-facing?**
Yes — visible to developers in Studio design canvas.

**5. What new did you learn from this file?**
The 25%/75% default positions are a deliberate design choice to show both handles in a visually balanced position, making the range slider concept immediately apparent to developers in the design canvas.

---

## src/utils/marks.ts

**1. What is the purpose of this file?**
Generates the `marks` object for the rc-slider component — an evenly-spaced set of tick marks with formatted labels.

**2. What kind of logic is described in this file?**
Validates `numberOfMarks > 0` and `min < max`. Computes interval as `(max - min) / numberOfMarks`. Iterates from `i=0` to `i=numberOfMarks` (inclusive), computing each mark value as `min + i * interval`, rounding to `decimalPlaces` via `parseFloat(value.toFixed(decimalPlaces))`. Returns a `Record<number, string>` mapping position to label.

**3. What part of behavior can be documented from this file?**
- Marks include both endpoints (min and max values).
- Total marks rendered: `numberOfMarks + 1` (endpoint inclusive).
- Decimal rounding uses `toFixed()` then `parseFloat()` to strip trailing zeros.
- Returns `undefined` when `numberOfMarks=0` or `min >= max` (invalid configuration).

**4. Is it user-facing?**
Yes — marks appear on the slider track as visible labels.

**5. What new did you learn from this file?**
The `parseFloat(value.toFixed(n))` pattern strips trailing zeros after decimal rounding — so `1.50` becomes `1.5` and `2.00` becomes `2`. This ensures clean label text without unnecessary padding.

---

## src/ui/RangeSlider.scss

**1. What is the purpose of this file?**
CSS styles for the range slider widget, importing and overriding `@rc-component/slider` and `@rc-component/tooltip` base styles.

**2. What kind of logic is described in this file?**
Imports `rc-slider/assets/index.css` and `@rc-component/tooltip/assets/bootstrap.css`. Widget container: 4px horizontal padding, 8px vertical padding, 16px bottom margin, `flex-grow: 1`. Disabled state: removes background color. Handle: forces `opacity: 1` (overrides rc-slider default fade). Vertical variant: `height: 100%`.

**3. What part of behavior can be documented from this file?**
- The slider container has default padding and bottom margin for spacing within Mendix layouts.
- Handle opacity is always 1 — the default rc-slider fade-in behavior on hover is disabled.
- Disabled state removes the track background color (visual indication of non-interactivity).
- Vertical slider stretches to 100% of its container height.
- v3.0.0 broke custom CSS due to the migration from `rc-slider` to `@rc-component/slider` (different CSS class names).

**4. Is it user-facing?**
Yes — directly affects visual appearance.

**5. What new did you learn from this file?**
The import of `@rc-component/tooltip/assets/bootstrap.css` means the tooltip uses Bootstrap's CSS as its base styling. This is inherited from the underlying library choice, not a deliberate Mendix design decision.

---

## typings/RangeSliderProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `RangeSlider.xml`. Defines `RangeSliderContainerProps` (runtime) and `RangeSliderPreviewProps` (Studio design-mode).

**2. What kind of logic is described in this file?**
Enumerations: `MinValueTypeEnum`, `MaxValueTypeEnum`, `StepSizeTypeEnum` (all "static"|"dynamic"|"expression"), `TooltipTypeLowerEnum`/`TooltipTypeUpperEnum` ("value"|"custom"), `OrientationEnum` ("horizontal"|"vertical"), `HeightUnitEnum` ("px"|"percentage"). Container props: `lowerBoundAttribute: EditableValue<Big>`, `upperBoundAttribute: EditableValue<Big>`, static values as `number`, dynamic as `EditableValue<Big>`, expression as `DynamicValue<Big>`, tooltip text as `DynamicValue<string>`, `onChange?: ActionValue`.

**3. What part of behavior can be documented from this file?**
- Both bound attributes use `EditableValue<Big>` — they are writable and use Big.js for decimal precision.
- Static min/max/step are typed as plain `number` (JavaScript number, not Big) — limited to standard float precision.
- Tooltip custom text is `DynamicValue<string>` — it can be an expression result.
- `onChange` is optional in the type.

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
Static values are plain `number` while dynamic/expression values are `Big` — meaning static values are subject to JavaScript floating-point limitations while dynamic and expression values have full decimal precision. Developers using very precise decimal ranges should prefer dynamic mode.

---

## src/components/__tests__/RangeSlider.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `RangeSliderComponent` presentation component using Jest and React Testing Library.

**2. What kind of logic is described in this file?**
Tests: horizontal and vertical snapshot baselines; `aria-valuenow` reflects slider state; step alignment (value=-21 with step=10 → displayed as -20); marks render with correct text; `onChange` mock is called on change. Uses `@testing-library/user-event` for interactions.

**3. What part of behavior can be documented from this file?**
- `aria-valuenow` attributes on handles reflect the current slider values — confirming ARIA accessibility support.
- rc-slider automatically snaps values to the nearest step increment.
- Marks are rendered as DOM text nodes with correct formatted values.
- Snapshot tests cover both horizontal and vertical orientations.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
Tests confirm that value snapping to step increments happens inside `@rc-component/slider` — the Container component does not need to snap values before passing them. The library handles alignment automatically.

---

## e2e/dataTypes.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the Range Slider widget in a real Mendix test application.

**2. What kind of logic is described in this file?**
Tests: slider track element is present with class `rc-slider-track-1` (confirming two-handle range mode); first marker label is "0" and second is "100"; the lower handle's bounding box X position is less than the upper handle's X position (confirming correct left-to-right ordering). Session logout after each test.

**3. What part of behavior can be documented from this file?**
- The slider renders in range mode (two handles) as confirmed by `rc-slider-track-1` class.
- Marker labels render with the default 0–100 range.
- Lower handle is always to the left of upper handle (correct ordering).
- `waitForNetworkIdle()` is used before assertions — confirming data-driven test setup.

**4. Is it user-facing?**
The tested behaviors (rendering, marker labels, handle ordering) are user-facing.

**5. What new did you learn from this file?**
The `rc-slider-track-1` class name (not `rc-slider-track-0`) is specific to range mode vs. single-handle mode in the rc-component library. This is a library implementation detail confirmed by the e2e test.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the range-slider-web widget.

**2. What kind of logic is described in this file?**
Key versions: 3.0.1 (2026-02-10, license docs), 3.0.0 (2023-12-22, migrated to `@rc-component/slider` — **breaking CSS change**), 2.1.3 (fix: initial value reset), 2.1.2 (fix: tooltip position on scroll), 2.1.1 (fix: handle icons, page explorer caption), 2.1.0 (feature: icons, page explorer caption), 2.0.0 (conversion to pluggable widget), 1.x (legacy Dojo widget).

**3. What part of behavior can be documented from this file?**
- v3.0.0 is a breaking change: custom CSS targeting old `rc-slider` class names will break after upgrade.
- v2.1.3 fixed an initial value reset bug — when the widget initialized, it would temporarily reset attribute values.
- v2.1.2 fixed tooltip misalignment when the page is scrolled.
- The widget was a Dojo widget before v2.0.0 and was converted to a React pluggable widget.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The v3.0.0 migration from `rc-slider` to `@rc-component/slider` was a deliberate breaking change, not a patch. The `@rc-component/` namespace represents the actively maintained successor project to the older `rc-*` libraries, which are in maintenance mode.
