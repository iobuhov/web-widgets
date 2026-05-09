# Range Slider

## Purpose

The Range Slider widget renders an interactive dual-handle slider on a Mendix page, enabling users to select a range of values by dragging two handles — a lower bound and an upper bound. Both bound values are stored in writable Mendix entity attributes. The minimum, maximum, and step size can each be independently sourced as static literals, dynamic entity attributes, or computed Mendix expressions. The widget supports horizontal and vertical orientations, configurable tooltip display per handle, evenly-spaced tick marks with formatted labels, and an optional onChange action. The widget is NOT offline capable and requires entity context.

## User Scenarios

### [P1] Select a numeric range by dragging handles
**Given** a page with a Range Slider widget bound to `lowerBoundAttribute` and `upperBoundAttribute`  
**When** the user drags either handle  
**Then** the dragged handle moves along the track, the opposing handle remains in place, and attribute values are written 500 ms after the drag ends

#### Edge Cases
- The lower handle cannot cross or surpass the upper handle: `allowCross={false}` is always enforced.
- During dragging, the slider tracks the mouse position visually but does not write attribute values on every pixel move.
- The debounce delay is 500 ms — attribute writes and the `onChange` action are emitted together once per drag gesture.
- If attribute values are unavailable on render, the slider falls back to min for the lower handle and max for the upper handle.

### [P2] Display tooltips on each handle
**Given** `showTooltip` is true  
**When** the user interacts with the slider  
**Then** a tooltip appears above the active handle showing either the current numeric value or a custom text string

#### Edge Cases
- When `tooltipAlwaysVisible` is true, both tooltips are permanently visible regardless of interaction state.
- When `tooltipAlwaysVisible` is false, tooltips appear only while a handle is being dragged (not on hover).
- Each handle has an independent tooltip type (`tooltipTypeLower`, `tooltipTypeUpper`) — one can show the numeric value while the other shows a custom text.
- Tooltips are rendered as portals into the slider root div to prevent overflow clipping.
- Tooltip hides immediately on mouse leave (0 ms delay).
- The tooltip `key` includes the current value, forcing re-render and correct positioning on value change.

### [P3] Show tick marks along the track
**Given** `showMarkers` is true and `noOfMarkers` is configured  
**When** the slider renders  
**Then** `noOfMarkers + 1` tick marks appear at evenly-spaced positions from min to max (endpoints inclusive), with labels formatted to `decimalPlaces` decimal places

#### Edge Cases
- Returns no marks when `noOfMarkers = 0` or `min >= max`.
- Labels strip trailing zeros: `1.50` displays as `1.5`, `2.00` displays as `2`.
- Mark positions are recalculated only when `noOfMarkers`, `decimalPlaces`, `min`, or `max` change.
- Default min/max (0/100) are used for mark calculation until actual values load, which may briefly show incorrect mark spacing.

### [P4] Use vertical orientation
**Given** `orientation === "vertical"` and `heightUnit`/`height` are configured  
**When** the slider renders  
**Then** the slider track runs vertically and the widget container applies the configured height

#### Edge Cases
- Vertical orientation applies the CSS class `"widget-range-slider-vertical"` to the root element.
- Height is applied as `{ height: "{value}{unit}" }` (e.g., `"200px"` or `"50%"`).
- Horizontal sliders have no explicit height style applied.

### [P5] Slider is disabled while values are loading
**Given** any of min, max, step, lowerBoundAttribute, or upperBoundAttribute is still loading  
**When** the slider renders  
**Then** the slider is non-interactive (disabled state) until all values are available

#### Edge Cases
- The loading gate is one-way: once a min/max/step value loads, the hook never re-enters loading state even if the attribute becomes unavailable later.
- Disabled state removes the track background color visually.
- Both bound attributes must report an available value before the slider becomes interactive.

## Functional Requirements

- FR-001: The widget MUST bind two writable Mendix entity attributes — `lowerBoundAttribute` and `upperBoundAttribute` — both of which are required.
- FR-002: Min, max, and step MUST each independently support three source types: static (`Big` literal), dynamic (`EditableValue<Big>`), or expression (`DynamicValue<Big>`).
- FR-003: The slider MUST enforce `allowCross={false}`: the lower handle MUST NOT exceed the upper handle.
- FR-004: Attribute writes MUST be debounced at 500 ms. Both attribute writes and the `onChange` action MUST execute inside the debounced function.
- FR-005: Attribute values MUST be written as `Big` instances (`Big(lower)`, `Big(upper)`) to maintain decimal precision.
- FR-006: The slider MUST be disabled while any of min, max, step, or bound attribute values are in a loading state.
- FR-007: If bound attribute values are unavailable, the slider MUST default to: lowerBoundAttribute → min, upperBoundAttribute → max.
- FR-008: Tick mark count MUST be `noOfMarkers + 1` (both endpoints included). Mark labels MUST be formatted to `decimalPlaces` using `parseFloat(value.toFixed(decimalPlaces))` to strip trailing zeros.
- FR-009: Marks MUST NOT be generated when `noOfMarkers === 0` or `min >= max`.
- FR-010: Each handle MUST have an independent tooltip with its own type configuration (`tooltipTypeLower`, `tooltipTypeUpper`).
- FR-011: Tooltips MUST be portaled into the slider root div to prevent overflow clipping.
- FR-012: In vertical orientation, the container MUST apply a height style from `heightUnit` and `height` props.
- FR-013: The widget is NOT offline capable.
- FR-014: Validation alerts for both `lowerBoundAttribute` and `upperBoundAttribute` MUST be rendered below the slider UI.
- FR-015: The Studio design canvas MUST show handles at 25% and 75% of the configured static range; dynamic/expression types default to range 0–100 for preview purposes.
- FR-016: Custom CSS targeting `rc-slider` class names from versions prior to v3.0.0 will be broken — the widget uses `@rc-component/slider` (v3.0.0+) with different class names.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `lowerBoundAttribute` | `EditableValue<Big>` | — | Lower bound attribute | Required writable attribute for the lower handle value. |
| `upperBoundAttribute` | `EditableValue<Big>` | — | Upper bound attribute | Required writable attribute for the upper handle value. |
| `minValueType` | `"static" \| "dynamic" \| "expression"` | `"static"` | Minimum value type | Source type for the minimum boundary. |
| `staticMinimumValue` | `number` | `0` | Minimum value (static) | Static literal minimum value. |
| `minAttribute` | `EditableValue<Big>?` | — | Minimum attribute | Entity attribute for minimum (dynamic mode). |
| `expressionMinimumValue` | `DynamicValue<Big>?` | — | Minimum expression | Computed expression for minimum (expression mode). |
| `maxValueType` | `"static" \| "dynamic" \| "expression"` | `"static"` | Maximum value type | Source type for the maximum boundary. |
| `staticMaximumValue` | `number` | `100` | Maximum value (static) | Static literal maximum value. |
| `maxAttribute` | `EditableValue<Big>?` | — | Maximum attribute | Entity attribute for maximum (dynamic mode). |
| `expressionMaximumValue` | `DynamicValue<Big>?` | — | Maximum expression | Computed expression for maximum (expression mode). |
| `stepSizeType` | `"static" \| "dynamic" \| "expression"` | `"static"` | Step size type | Source type for the step size. |
| `staticStepSize` | `number` | `1` | Step size (static) | Static literal step size. |
| `stepAttribute` | `EditableValue<Big>?` | — | Step attribute | Entity attribute for step size (dynamic mode). |
| `expressionStepSize` | `DynamicValue<Big>?` | — | Step expression | Computed expression for step size (expression mode). |
| `showTooltip` | `boolean` | `true` | Show tooltip | Enables tooltip display on handles. |
| `tooltipTypeLower` | `"value" \| "custom"` | `"value"` | Lower tooltip type | Display numeric value or custom text on the lower handle tooltip. |
| `tooltipTypeUpper` | `"value" \| "custom"` | `"value"` | Upper tooltip type | Display numeric value or custom text on the upper handle tooltip. |
| `tooltipAlwaysVisible` | `boolean` | `false` | Always show tooltip | When true, both tooltips are permanently visible. |
| `showMarkers` | `boolean` | `false` | Show markers | Enables tick marks along the track. |
| `noOfMarkers` | `number` | `0` | Number of markers | Count of intervals; renders `noOfMarkers + 1` marks. |
| `decimalPlaces` | `number` | `0` | Decimal places | Decimal places for marker labels. |
| `orientation` | `"horizontal" \| "vertical"` | `"horizontal"` | Orientation | Slider orientation. |
| `heightUnit` | `"px" \| "percentage"` | `"px"` | Height unit | Unit for vertical slider height. Active only when `orientation === "vertical"`. |
| `height` | `number` | `100` | Height | Numeric height value for vertical slider. |
| `onChange` | `ActionValue?` | — | On change | Optional Mendix action fired 500 ms after the last handle drag event. |

## Changelog

**v3.0.1 (2026-02-10):** License documentation update only.

**v3.0.0 (2023-12-22):** Migrated from `rc-slider` to `@rc-component/slider`. **Breaking change**: custom CSS targeting old `rc-slider` class names will break after upgrade.

**v2.1.3:** Fixed initial value reset bug where the widget would temporarily reset attribute values on initialization.

**v2.1.2:** Fixed tooltip misalignment when the page is scrolled.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] What is the allowed type for `lowerBoundAttribute` and `upperBoundAttribute` in Studio Pro — Integer, Decimal, Long, or all three?
- [ ] Is there a visual indicator when the slider is in loading/disabled state (besides removing the track background color)?
- [ ] Does the `onLeave`-style onChange behavior exist, or is onChange always debounced at 500 ms on every change?
