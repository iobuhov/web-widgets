# Chart Playground (chart-playground-web)

## Purpose

The Chart Playground widget provides an in-browser developer tool for interactively editing a parent chart widget's Plotly configuration (layout, traces, and config) as raw JSON without changing the saved Studio Pro model. It is designed for preview-time experimentation only and must be placed inside a chart widget that supplies a playground context. The widget has no standalone configurable properties.

## User Scenarios

### [P1] Open the playground editor sidebar
**Given** a chart widget on a Mendix page with Chart Playground placed inside it  
**When** the developer clicks the "Toggle Editor" button (visible top-right of the chart)  
**Then** the playground sidebar opens, showing two code panels and a view selector  

#### Edge Cases
- The toggle button is always visible regardless of chart state or data availability
- If the widget is placed outside a chart context, an error alert is shown instead of the editor

### [P2] Edit chart configuration in the preview session
**Given** the playground sidebar is open  
**When** the developer edits JSON in the editable code panel  
**Then** the chart updates live to reflect the new configuration for the current browser session only  

#### Edge Cases
- Changes are preview-only; they are NOT persisted to the Studio Pro model
- A tooltip warning is always shown: "Changes are only for preview purposes. To persist changes copy value and paste it in Studio Pro."
- Invalid JSON is silently ignored; the chart is not updated until the JSON becomes valid

### [P3] Switch between configuration views
**Given** the playground sidebar is open  
**When** the developer selects a different view from the dropdown (Layout, individual traces, Configuration)  
**Then** the editable code panel displays the JSON for that view  

#### Edge Cases
- Trace views are listed by trace name; the number of views depends on the chart's series count
- Switching view resets the editable panel content to the current value for that view
- Keys `x`, `y`, `z`, `customSeriesOptions`, and `dataSourceItems` are filtered from the read-only modeler panel to reduce noise

### [P4] Read modeler settings panel
**Given** the playground sidebar is open  
**When** the developer views the second (read-only) code panel  
**Then** the panel shows the configuration as currently set in Studio Pro, excluding data-bound series keys  

#### Edge Cases
- The read-only panel reflects Studio Pro settings; it does not update when the editable panel changes
- For V2 charts, the read-only panel automatically syncs when the chart's underlying Mendix data refreshes (MobX reactivity)
- For V1 charts, the panel reflects a snapshot at the time the view was selected

### [P5] Navigate to Plotly documentation
**Given** the playground sidebar is open  
**When** the developer clicks the docs link  
**Then** the browser opens the Plotly reference documentation  

#### Edge Cases
- The docs link is always visible in the sidebar

## Functional Requirements

- FR-001: The widget MUST require a parent chart widget that provides a playground context via `usePlaygroundContext()`; it MUST render an error alert if no context is available.
- FR-002: The widget MUST have zero configurable properties in Studio Pro; all data is supplied at runtime via React context.
- FR-003: The widget MUST render a "Toggle Editor" button at `position: absolute; right: 10px; top: 10px; z-index: 50` over the parent chart.
- FR-004: When the sidebar is open, the widget MUST display a persistent tooltip warning that changes are preview-only and must be manually copied back to Studio Pro.
- FR-005: The editable code panel MUST accept raw JSON input and MUST silently discard invalid JSON (no error feedback to the user).
- FR-006: The widget MUST support two editor generations: V1 (plain React state, `PlaygroundDataV1`) and V2 (MobX-reactive, `PlaygroundDataV2`); the generation is determined at runtime from the context type discriminant `"editor.data.v2"`.
- FR-007: For V2 charts, the editable panel MUST automatically update when the parent chart's data refreshes externally (via MobX `reaction()`).
- FR-008: The view selector MUST offer Layout, one entry per chart trace (named by trace name), and a Configuration option.
- FR-009: The read-only modeler panel MUST exclude keys `x`, `y`, `z`, `customSeriesOptions`, and `dataSourceItems` from display.
- FR-010: Keyboard Tab focus MUST be trapped within the editor panels while the sidebar is open.
- FR-011: When the playground is open, the parent chart SVG MUST receive a `2px solid #172c89` border as a visual connection indicator.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| *(none)* | — | — | — | This widget has no configurable properties. All runtime data is injected via React context from the parent chart widget. |

## Changelog

**2.1.1** (2025-07-15) — Updated `@mendix/shared-charts` dependency.

**2.0.0** — Updated Plotly.js dependency to v3 (breaking change in Plotly major version).

**1.1.0** — Bundling change (shared-charts package extraction).

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Which specific chart widget types are compatible with Chart Playground (i.e., which charts expose a playground context)?
- [ ] Is there a documented procedure for applying playground-edited JSON back to Studio Pro (beyond the tooltip's copy-paste instruction)?
- [ ] What Plotly.js v2→v3 API changes affect existing playground configurations?
