# Calendar

## Purpose

The Calendar widget renders a full-featured interactive calendar in a Mendix web application, powered by react-big-calendar. It displays a list of events from a Mendix data source across day, week, month, work week, and agenda views. Users can navigate between dates, create new events by selecting time slots, edit events by clicking them, and drag/drop or resize events to reschedule them. The widget is intended for scheduling, planning, and time-management use cases where a visual calendar representation of Mendix entity data is required.

## User Scenarios

### [P1] Display events from a Mendix data source on a calendar

**Given** `databaseDataSource` is bound to a list of objects with title, start, end, and optional color attributes  
**When** the page loads and the data source is available  
**Then** events are rendered on the calendar in the correct date/time positions, using their configured color for the event block background

#### Edge Cases

- When `databaseDataSource` is not configured, the calendar renders without events (empty calendar).
- When an event's `eventColor` is a hex color (`#RGB` or `#RRGGBB`), selection lightens the color by 25%. For all other color formats (named, rgb, rgba), selection applies a CSS `brightness(0.85)` filter instead.
- When `allDay` is true for an event, it renders in the all-day row at the top of day/week views.
- When `allDay` is not configured but the event duration is an exact multiple of 24 hours, the event is treated as all-day.

---

### [P2] Create a new event by selecting a time slot

**Given** `editable = true` and `onCreateEvent` is configured  
**When** the user clicks or selects a time range in an empty calendar slot  
**Then** `onCreateEvent` fires with action variables `startDate`, `endDate`, and `allDay` populated from the selected slot

#### Edge Cases

- When an event is currently selected, clicking an empty slot does NOT create a new event — the user must deselect the event first.
- When `editable = false`, slot selection is disabled and no create action fires.
- When `onCreateEvent` is not configured, slot selection is silently ignored.

---

### [P3] Edit an event by clicking it

**Given** `onEditEvent` is configured  
**When** the user single-clicks an event that is already selected, or double-clicks any event  
**Then** `onEditEvent` fires with the event's Mendix `ObjectItem` as context

#### Edge Cases

- A single click on an unselected event selects it but does NOT trigger `onEditEvent`; a second single click on the same (now selected) event triggers `onEditEvent` after a 250ms deduplication window.
- Double-clicking immediately triggers `onEditEvent` (the single-click timer is cancelled).
- `onEditEvent` is guarded by `ListActionValue.canExecute` — the action does not fire if access rights prevent it.

---

### [P4] Drag, drop, and resize events to reschedule

**Given** `editable = true` and `onDragDropResize` is configured  
**When** the user drags an event to a new position or resizes it  
**Then** `onDragDropResize` fires with action variables `oldStart`, `oldEnd`, `newStart`, `newEnd` populated from the drag operation

#### Edge Cases

- The `onDragDropResize` action provides both old and new start/end times, allowing the developer to update the Mendix entity accordingly.
- `onDragDropResize` fires with the event's `ObjectItem` as context for ListActionValue resolution.

---

### [P5] Navigate to a specific date via a Mendix attribute

**Given** `startDateAttribute` is bound to a writable DateTime attribute  
**When** `startDateAttribute.status = "loading"`, a loading bar is shown; when it becomes `"available"`, the calendar displays the date from the attribute value  
**Then** the calendar initializes at the date stored in the attribute; the loading bar dismisses automatically

#### Edge Cases

- When `startDateAttribute.status` is `"unavailable"` or the attribute is not configured, the calendar renders with today's date as the default.
- The loading bar is an indeterminate animated progress element — it does not reflect actual load progress.

---

### [P6] Monitor view range changes

**Given** `onViewRangeChange` is configured  
**When** the user navigates to a different date range or switches views  
**Then** `onViewRangeChange` fires with action variables `rangeStart`, `rangeEnd`, and `currentView`

#### Edge Cases

- `rangeStart`/`rangeEnd` for month/agenda views are passed as a `{start, end}` object; for day/week/custom views they are passed as a date array — the action variable binding must match the expected type.
- `currentView` is tracked via an internal ref synchronized on navigation, because RBC does not pass the view name in the `onRangeChange` callback for month and agenda views.

---

## Functional Requirements

- FR-001: The system MUST render events from `databaseDataSource` on the correct calendar date and time positions.
- FR-002: The system MUST apply the event color from `eventColor` as `backgroundColor` on the event block when provided.
- FR-003: The system MUST treat events with exact 24-hour multiples as all-day events even when `allDay` attribute is not configured.
- FR-004: The system MUST render an indeterminate loading bar (`<progress>`) when `startDateAttribute.status === "loading"`.
- FR-005: The system MUST initialize the calendar at the date from `startDateAttribute.value` when available.
- FR-006: The system MUST support two view modes: `standard` (day/week/month) and `custom` (day/week/month/work_week/agenda with configurable toolbar).
- FR-007: The system MUST clamp `step` to [1, 60] and log a console warning if out of range.
- FR-008: The system MUST clamp `timeslots` to [1, 4] and log a console warning if out of range.
- FR-009: When the configured `defaultView` is not in the active views set, the system MUST fall back to the first enabled view to prevent a runtime crash.
- FR-010: The system MUST block event creation (`onCreateEvent`) when an event is currently selected.
- FR-011: The system MUST use a 250ms deduplication timer to distinguish single-click (select) from double-click (edit) interactions.
- FR-012: The system MUST execute `onEditEvent` only when `ListActionValue.canExecute` is true for the event's entity instance.
- FR-013: The system MUST fire `onDragDropResize` with `oldStart`, `oldEnd`, `newStart`, `newEnd` variables after a drag or resize operation.
- FR-014: The system MUST fire `onCreateEvent` with `startDate`, `endDate`, `allDay` variables when a slot is selected and `editable = true`.
- FR-015: The system MUST fire `onViewRangeChange` with `rangeStart`, `rangeEnd`, `currentView` when the visible date range changes.
- FR-016: The calendar container MUST have a minimum height of 350px enforced via CSS — no configuration can make it shorter.
- FR-017: The calendar MUST use the Mendix session locale for weekday names, month names, date/time format patterns, and week start day.
- FR-018: The widget MUST NOT require an entity context — it can be placed anywhere on a page.
- FR-019: The widget MUST function in offline-enabled Mendix applications (`offlineCapable = true`).
- FR-020: The system MUST support custom toolbar items in `custom` view mode with configurable position (left/center/right), render mode (button/link), button style, caption, and tooltip.
- FR-021: In custom view mode, the work_week view MUST navigate in full weekly increments and render only the configured visible days.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `databaseDataSource` | ListValue | _(none)_ | Data source | Optional event list. Calendar renders without events when not configured. |
| `titleAttribute` | ListAttributeValue\<string\> | — | Title attribute | Event title from an entity attribute. |
| `titleExpression` | ListExpressionValue\<string\> | — | Title expression | Event title from an expression. Mutually exclusive with title attribute. |
| `eventColor` | ListAttributeValue\<string\> | — | Color | HTML color string (named, hex, rgb, rgba) for the event background. Hex colors support selection lightening. |
| `startDateAttribute` | ListAttributeValue\<Date\> | — | Start date | Event start datetime. |
| `endDateAttribute` | ListAttributeValue\<Date\> | — | End date | Event end datetime. |
| `allDay` | ListAttributeValue\<boolean\> | — | All day | When true, event renders in the all-day row. Also inferred from exact 24h duration when not set. |
| `editable` | DynamicValue\<boolean\> | — | Editable | Enables slot selection (create) and drag/drop/resize. |
| `viewMode` | enum | `"standard"` | View mode | `"standard"` (day/week/month toolbar) or `"custom"` (configurable toolbar). |
| `defaultViewStandard` | enum | — | Default view | Initial view for standard mode: day/week/month. |
| `defaultViewCustom` | enum | — | Default view | Initial view for custom mode. Must match a view enabled in toolbar items; falls back to first enabled view if not. |
| `timeFormat` | string | — | Time format | date-fns format pattern for time display (standard mode only). Falls back to locale-aware `"p"` on invalid pattern. |
| `topBarDateFormat` | string | — | Top bar date format | date-fns format pattern for the toolbar date label (standard mode only). |
| `showEventDate` | DynamicValue\<boolean\> | — | Show event date | When false, suppresses time range text in event blocks. |
| `minHour` | integer | — | Minimum hour | Earliest hour displayed in time-grid views (0–23). |
| `maxHour` | integer | — | Maximum hour | Latest hour displayed in time-grid views (0–23). |
| `showAllEvents` | boolean | — | Show all events | When true, shows all events in month cells without truncation. |
| `step` | integer | — | Step | Time slot duration in minutes. Clamped to [1, 60]. |
| `timeslots` | integer | — | Timeslots | Number of time slots per `step` interval. Clamped to [1, 4]. |
| `startDateAttribute` | EditableValue\<Date\> | — | Start date attribute | Non-list DateTime attribute controlling the calendar's initial date. When loading, shows progress bar. |
| `toolbarItems` | list | — | Toolbar items | Custom toolbar items (custom view mode only). Each item has: type (9 types), position (left/center/right), caption, renderMode (button/link), buttonStyle, tooltip, and per-view date format strings. |
| `visibleDays.sunday` … `visibleDays.saturday` | boolean | true | Visible days | Which weekdays appear in the work_week custom view. |
| `onEditEvent` | ListActionValue | — | On edit event | Action fired when an event is edited (single-click selected, double-click, or keyboard Enter). Receives event's ObjectItem as context. |
| `onCreateEvent` | ActionValue | — | On create event | Action fired on slot selection. Variables: `startDate`, `endDate`, `allDay`. Blocked when an event is selected. |
| `onDragDropResize` | ListActionValue | — | On drag/drop/resize | Action fired after drag, drop, or resize. Variables: `oldStart`, `oldEnd`, `newStart`, `newEnd`. |
| `onViewRangeChange` | ActionValue | — | On view range change | Action fired when the visible date range changes. Variables: `rangeStart`, `rangeEnd`, `currentView`. |
| `widthUnit` | enum | `"percentage"` | Width unit | `"percentage"` or `"pixels"`. |
| `width` | integer | — | Width | Widget width in selected unit. |
| `heightUnit` | enum | `"percentageOfWidth"` | Height unit | `"percentageOfWidth"` (auto/content height), `"pixels"`, `"percentageOfParent"`, `"percentageOfView"`. |
| `height` | integer | — | Height | Widget height. Ignored when `heightUnit = "percentageOfWidth"` (auto mode). |
| `minHeightUnit` | enum | — | Min height unit | Unit for minimum height. `"none"` disables min height. |
| `minHeight` | integer | — | Min height | Minimum height value. Only applies in auto height mode. |
| `maxHeightUnit` | enum | — | Max height unit | Unit for maximum height. `"none"` disables max height. |
| `maxHeight` | integer | — | Max height | Maximum height value. When set in auto height mode, also enables `overflowY`. |
| `overflowY` | enum | — | Overflow Y | Scroll behavior when content exceeds max height. Only active when max height is configured. |

## Changelog

### [2.4.0] - 2026-03-20
- Fixed: `startDateAttribute` handling for correct calendar initialization.

### [2.3.0] - 2026-02-17
- Added: `step` and `timeslots` properties for configurable time slot granularity.
- Fixed: `onViewRangeChange` not triggering when switching from Day/Week to Month view.

### [2.2.0] - 2025-11-11
- **Breaking change**: custom view captions are now configured inside `toolbarItems` (not separate caption props). Apps using v2.1.x custom view captions require reconfiguration.
- Added: Configurable toolbar items with captions, tooltips, link-style appearance, and consistent cross-view title formatting.
- Fixed: Localization (dates, weekday/month names), custom format patterns, abbreviated month names, default view error when view not in toolbar, title expression rendering.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] Is the `startDateAttribute` writable at runtime (does the widget ever call `setValue()` on it to persist navigation)? The type is `EditableValue<Date>` which supports writing, but no write call was found in the source files reviewed.
- [ ] What is the exact `currentView` string passed in `onViewRangeChange` variables — does it use RBC's internal view name strings (e.g. `"work_week"`) or the enum values from the XML?
- [ ] The custom week view always navigates in weekly increments even when only a subset of days is shown. Is this the intended UX, or should navigation skip to the next occurrence of a visible day?
- [ ] The `lightColor` function only processes `#RGB` and `#RRGGBB` hex formats. Are there plans to support RGB/RGBA/named-color lightening for event selection?
- [ ] What is the behavior when `allDay` attribute returns `false` but the event spans exactly 24h? The draft shows the `(duration_ms / 86400000) % 1 === 0` check applies regardless of the `allDay` attribute — does the attribute value take precedence?
