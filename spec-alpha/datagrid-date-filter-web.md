# DatagridDateFilter

## Purpose

The DatagridDateFilter widget provides a date-based filter control for use inside a Data Grid 2 or Gallery widget. It supports nine filter operators (equal, not equal, greater, greater or equal, smaller, smaller or equal, between, empty, not empty) and two integration modes: "auto" (the parent DataGrid injects the filter store via React context) and "linked" (the widget manages its own filter store from explicitly configured DateTime attributes). It is used when end-users need to filter tabular or gallery data by date or date range.

## User Scenarios

### [P1] Filter by a single date value

**Given** the widget is placed inside a Data Grid 2 column header or Gallery header, with `adjustable = true` and a non-"between" default filter  
**When** the user types a date into the input field or picks one from the calendar  
**Then** the datagrid or gallery filters to rows matching the selected date and operator; the `onChange` action fires; the selected value is written to `valueAttribute` if configured

#### Edge Cases

- `strictParsing` is enabled: typed dates that do not match the locale date format are rejected and do not update the filter.
- The date format accepted by the input field is derived from the Mendix app's session locale. Single-digit format patterns (e.g., `d/M/yyyy`) are normalized to double-digit equivalents (e.g., `dd/MM/yyyy`); both the normalized and original formats are accepted so users can type with or without leading zeros.
- Custom date format patterns using uppercase `E` (day-of-week) are automatically converted to lowercase `e` for `date-fns` compatibility.

### [P2] Filter by a date range ("between")

**Given** the widget is configured with `defaultFilter = "between"`  
**When** the user selects start and end dates via the calendar picker  
**Then** the datagrid filters to rows where the date attribute falls within the selected range (inclusive); `startDateAttribute` and `endDateAttribute` are written if configured; the calendar stays open after the first date selection until the user selects the second date or dismisses

#### Edge Cases

- `allowSameDay = false`: start and end date cannot be the same day.
- Pressing Backspace in the input while in range mode clears both dates.
- Direct text input is blocked in range mode (`UNSAFE_handleChangeRaw`); users must use the calendar UI.
- A clear button appears in the date input only in range mode (`isClearable = true` for `selectsRange`).

### [P3] Change filter operator at runtime

**Given** `adjustable = true`  
**When** the user clicks the filter type selector button  
**Then** a dropdown shows the 9 available operators; selecting one immediately opens the calendar; the button is accessible via keyboard (Enter, Space, arrow keys)

#### Edge Cases

- When `adjustable = false`, the filter type selector button is entirely hidden; the filter type is fixed and cannot be changed by the end-user or by personalization settings.
- When the operator is "empty" or "notEmpty", the date input is disabled — no date is needed for these comparisons.

### [P4] Default value on page load

**Given** `defaultValue`, `defaultStartDate`, or `defaultEndDate` is configured  
**When** the page loads  
**Then** the filter is initialized with the configured default values; the widget defers rendering until all default value expressions are resolved (prevents flash of empty state)

#### Edge Cases

- **Breaking constraint (since v2.5.0):** The default value is used only as the initial value at mount. Subsequent changes to the `defaultValue` prop (e.g., from a microflow updating the expression) are ignored — the filter does NOT update reactively.
- The widget waits for all three default value expressions to resolve before rendering, even if only one is configured.

### [P5] Programmatic filter control via actions

**Given** the widget is on a page and the developer configures microflow/nanoflow actions  
**When** a `Reset_Filter` action is triggered (scoped to this widget's name or its `parentChannelName`)  
**Then** the filter is reset to its default values or cleared; when a `Set_Filter` action is triggered, the operator, start date, and end date are set atomically

#### Edge Cases

- When inside a filter group container, events are routed via `parentChannelName` rather than widget name — enabling a single "Reset All" button to reset all filters in the group simultaneously.

### [P6] Placement error

**Given** the widget is placed outside a Data Grid 2 column/header or Gallery header in "auto" mode  
**When** the page is rendered  
**Then** an alert is displayed: "The filter widget must be placed inside the column or header of the Data grid 2.0 or inside header of the Gallery widget."

### [P7] Locale-aware calendar

**Given** the Mendix app is configured for a specific locale (e.g., pt-BR, fi-FI, en-US)  
**When** the user opens the calendar picker  
**Then** day names and month/year navigation are displayed in the app's locale; the first day of the week follows the locale's `firstDayOfWeek` setting (0 = Sunday, 1 = Monday); the date input format matches the locale's date pattern

## Functional Requirements

- FR-001: The widget MUST support two integration modes: "auto" (filter store from parent DataGrid/Gallery context) and "linked" (filter store from configured DateTime attributes).
- FR-002: In "auto" mode, the widget MUST render an error alert if placed outside a Data Grid 2 column/header or Gallery header, or if the parent provides a store of the wrong type.
- FR-003: The widget MUST support nine filter operators: between, greater, greaterEqual, equal, notEqual, smaller, smallerEqual, empty, notEmpty.
- FR-004: When `adjustable = true`, the widget MUST render a filter type selector button that allows the end-user to change the operator at runtime.
- FR-005: When `adjustable = false`, the widget MUST hide the filter type selector button; the operator MUST be fixed and immune to personalization overrides.
- FR-006: When the selected operator is "empty" or "notEmpty", the date input MUST be disabled.
- FR-007: When the selected operator is "between", the widget MUST use a range date picker with `startDate`/`endDate`; `allowSameDay` MUST be false; direct text input MUST be blocked.
- FR-008: The widget MUST defer rendering until all default date expression values are resolved.
- FR-009: The default value MUST be used only as the initial value at mount; subsequent prop changes to `defaultValue`, `defaultStartDate`, or `defaultEndDate` MUST be ignored.
- FR-010: The widget MUST apply `strictParsing` to reject typed dates that do not match the locale date format.
- FR-011: The widget MUST normalize single-digit date format patterns (e.g., `d` → `dd`, `M` → `MM`) and accept both normalized and original formats from typed input.
- FR-012: The widget MUST convert uppercase `E` format tokens to lowercase `e` for date-fns compatibility.
- FR-013: The calendar picker MUST use a fixed-position portal to prevent clipping inside scrollable containers.
- FR-014: The calendar picker MUST show month and year dropdowns to allow rapid navigation.
- FR-015: The calendar toggle button MUST use `mousedown` (not `click`) to correctly race with the calendar's outside-click dismissal handler.
- FR-016: The widget MUST respond to `Reset_Filter` and `Set_Filter` external action events, scoped by widget name or `parentChannelName`.
- FR-017: The widget MUST write the selected date value to `valueAttribute`, and range dates to `startDateAttribute`/`endDateAttribute`, when those attributes are configured.
- FR-018: The widget MUST fire the `onChange` action when the filter value or operator changes.
- FR-019: The widget MUST support WCAG 2.1 AA accessibility; it MUST expose accessible labels for the filter type button, calendar toggle button, and date input via `screenReaderButtonCaption`, `screenReaderCalendarCaption`, and `screenReaderInputCaption`.
- FR-020: Multiple filter instances on the same page MUST have unique accessible label IDs.
- FR-021: The filter type selector MUST be keyboard operable (Enter, Space, arrow keys).
- FR-022: The widget MUST support offline apps (`offlineCapable = true`).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `attrChoice` | enum: `auto \| linked` | — | Attribute mode | "auto": parent DataGrid/Gallery provides the store; "linked": widget manages its own store from configured attributes |
| `attributes` | `AttributesType[]` | — | Attributes | List of DateTime attributes for linked mode; ignored in auto mode |
| `linkedDs` | `ListValue` | — | Data source | Linked datasource for attribute mode |
| `defaultFilter` | enum: `between \| greater \| greaterEqual \| equal \| notEqual \| smaller \| smallerEqual \| empty \| notEmpty` | `equal` | Default filter | Initial filter operator |
| `defaultValue` | `DynamicValue<Date>` | — | Default value | Initial date value (non-between filters); used at mount only |
| `defaultStartDate` | `DynamicValue<Date>` | — | Default start date | Initial start date for "between" filter; used at mount only |
| `defaultEndDate` | `DynamicValue<Date>` | — | Default end date | Initial end date for "between" filter; used at mount only |
| `placeholder` | `DynamicValue<string>` | — | Placeholder | Placeholder text shown in the date input |
| `adjustable` | `boolean` | `true` | Adjustable | Whether end-users can change the filter operator at runtime |
| `valueAttribute` | `EditableValue<Date>` | — | Value attribute | Stores the last selected single date for persistence |
| `startDateAttribute` | `EditableValue<Date>` | — | Start date attribute | Stores the last range start date for persistence |
| `endDateAttribute` | `EditableValue<Date>` | — | End date attribute | Stores the last range end date for persistence |
| `onChange` | `ActionValue` | — | On change | Action triggered when filter value or operator changes |
| `screenReaderButtonCaption` | `DynamicValue<string>` | `"Select filter type"` | Filter button label | ARIA label for the filter type selector button; hidden when `adjustable = false` |
| `screenReaderCalendarCaption` | `DynamicValue<string>` | `"Show calendar"` | Calendar button label | ARIA label for the calendar toggle button |
| `screenReaderInputCaption` | `DynamicValue<string>` | `"date filter"` | Input label | ARIA label for the date input |
| `tabIndex` | `number` | `0` | Tab index | Tab order for the widget |

## Changelog

**v3.10.0 (2026-05-06)**
- Fixed filter selector dropdown placement on small viewports.

**v3.8.1 (2026-02-19)**
- Filter selector dropdown now auto-selects best placement based on available space.

**v2.11.2 (2025-03-20)**
- Fixed: non-adjustable (`adjustable = false`) default filters were overridden by personalization settings. Behavioral constraint: if `adjustable = false`, the operator is now immutable by personalization.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] What is the exact behavior when `attrChoice = "linked"` and no `attributes` are configured — does the widget silently fail, show an error, or fall back to auto mode?
- [ ] Is the `parentChannelName` prop configurable by developers, or is it always injected by a parent container widget? The source shows it comes from `filterAPI` context but the Studio Pro configuration is not confirmed in the draft.
- [ ] The v2.10.0 CHANGELOG removed the "Group key" property added in v2.9.0 — is the current `parentChannelName` / filter group mechanism a direct replacement or a different design?
- [ ] Is the `filter-input` CSS class on the date input (present only when `adjustable = true`) documented as a stable public API for theme customization?
