# Timeline

## Purpose

The Timeline widget renders a vertically scrolling list of time-stamped events from a Mendix datasource, optionally grouped by date (day, month, or year). It supports two visualization modes: a basic mode using expression-bound text fields and a Mendix icon, and a custom mode where every visual element (icon, group header, title, date, description) is a developer-supplied widget slot per item. Events can be clickable with per-item Mendix actions. The widget is semantically a `<ul>` list and is accessible by default. It requires entity context and supports offline use.

## User Scenarios

### [P1] View grouped events
**Given** a Timeline widget with `groupEvents = true`, a `groupAttribute` pointing to a Date attribute, and `groupByKey = "day"`  
**When** data loads  
**Then** events are grouped under date headers formatted as the chosen day format (e.g., "Monday", "15 January", or "15 January 2024"), with all ungrouped items (no `groupAttribute` value) placed at the beginning or end as configured  

#### Edge Cases
- Events without a `groupAttribute` value are grouped under an empty key and rendered at beginning or end based on `ungroupedEventsPosition`.
- When `groupEvents = false`, no group headers are rendered.

### [P2] Click an event
**Given** a Timeline with `onClick` configured  
**When** the user clicks an event item  
**Then** the Mendix action executes in the context of that event's object item; events have CSS class `"clickable"` and a hover style  

### [P3] Custom visualization
**Given** a Timeline in custom mode (`customVisualization = true`)  
**When** data loads  
**Then** each event renders the per-item widget content from `customIcon`, `customTitle`, `customEventDateTime`, and `customDescription` slots; `customGroupHeader` renders as the group separator header per group  

#### Edge Cases
- Custom slots with no configured content produce no rendered `<li>` elements (guarded by `hasChildren()` check).
- Month grouping in custom mode uses `monthYear` format (e.g., "January 2024") rather than just "January" to prevent ambiguous labels across multiple years.

## Functional Requirements

- FR-001: System MUST support two visualization modes: `customVisualization = false` (basic) and `customVisualization = true` (custom). Basic and custom modes share the same `TimelineComponent` but inject different data shapes.
- FR-002: In basic mode, each event MUST render: a Mendix `Icon` (same icon for all items), `title` (per-item expression), `timeIndication` (per-item expression, optional), and `description` (per-item expression, optional).
- FR-003: In custom mode, all five slots MUST be per-item widget content: `customIcon`, `customGroupHeader`, `customTitle`, `customEventDateTime`, `customDescription`.
- FR-004: When `groupEvents = true`, the system MUST render a group header `<li>` element for each non-empty group key; events without a `groupAttribute` value MUST receive no group header.
- FR-005: Date group formatting MUST use locale-aware `DateTimeFormatter`. Supported formats: `dayName` ("EEEE"), `dayMonth` ("d MMMM"), `fullDate` ("d MMMM yyyy"), `month` ("MMMM"), `monthYear` ("MMMM yyyy"), `year` ("yyyy").
- FR-006: Custom mode MUST use `monthYear` format instead of `month` format to prevent ambiguous labels in multi-year datasets.
- FR-007: Events with an `onClick` action MUST receive CSS class `"clickable"` and the `onClick` action MUST execute when the event is activated.
- FR-008: The timeline MUST be rendered as a `<ul>` list with `<li>` items — accessible by screen readers as a list.
- FR-009: The widget MUST require entity context and support offline use.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `data` | `ListValue` | — | Data source | The Mendix entity datasource providing event items |
| `title` | `ListExpressionValue<string>` | — | Title | Per-item title expression (basic mode) |
| `description` | `ListExpressionValue<string>` (optional) | — | Description | Per-item description expression (basic mode) |
| `timeIndication` | `ListExpressionValue<string>` (optional) | — | Time indication | Per-item time display expression (basic mode) |
| `customVisualization` | boolean | `false` | Custom visualization | When true, switches to custom widget slot mode |
| `icon` | `ListAttributeValue<WebIcon>` (optional) | — | Icon | Icon used for all events in basic mode (same for all items) |
| `groupEvents` | boolean | — | Group events | When true, events are grouped under date headers |
| `groupAttribute` | `ListAttributeValue<Date>` (optional) | — | Group by | Date attribute used to compute group keys |
| `groupByKey` | `"day"` \| `"month"` \| `"year"` | — | Group by key | Granularity of grouping |
| `groupByDayOptions` | `"dayName"` \| `"dayMonth"` \| `"fullDate"` | — | Day format | Display format for day grouping |
| `groupByMonthOptions` | `"month"` \| `"monthYear"` | — | Month format | Display format for month grouping |
| `ungroupedEventsPosition` | `"beginning"` \| `"end"` | — | Ungrouped position | Where events without a group attribute value are placed |
| `onClick` | `ListActionValue` (optional) | — | On click | Per-item Mendix action |
| `customIcon` | `ListWidgetValue` (optional) | — | Custom icon | Per-item icon widget slot (custom mode) |
| `customGroupHeader` | `ListWidgetValue` (optional) | — | Custom group header | Per-item group header widget slot (custom mode) |
| `customTitle` | `ListWidgetValue` (optional) | — | Custom title | Per-item title widget slot (custom mode) |
| `customEventDateTime` | `ListWidgetValue` (optional) | — | Custom date/time | Per-item date/time widget slot (custom mode) |
| `customDescription` | `ListWidgetValue` (optional) | — | Custom description | Per-item description widget slot (custom mode) |

## Changelog

- **v3.2.3 (2026-02-10)**: License documentation added.
- **v3.2.2 (2024-06-26)**: Fixed data retrieval when items are accessed via Mendix association.
- **v3.1.1 (2022-12-12)**: Fixed month grouping in custom mode — introduced the `monthYear` format upgrade to prevent ambiguous labels in multi-year datasets.
- **v3.1.0 (2021-12-23)**: Added dark mode preview in Studio Pro.
- **v3.0.0 (2021-09-28)**: Initial major release.

## Open Questions

> Could not be determined from source code alone — requires human review
- [ ] In custom mode, which item's `customGroupHeader` is used to render the group header when a group has multiple items? (The implementation iterates over items, and the last item's group header content would win in the map if multiple items are in the same group.)
- [ ] Can the timeline "line" (CSS `border-left`) be removed for specific groups via the `.no-divider` CSS class, or is that only available globally?
