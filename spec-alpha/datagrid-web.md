# Datagrid (Data Grid 2)

## Purpose

Data Grid 2 (`datagrid-web`) is a full-featured, pluggable data grid widget for Mendix applications. It displays a list datasource in a scrollable, interactive table that supports column sorting, column hiding and reordering, per-column filter widgets, single and multi-row selection, Excel export, runtime personalization persistence (localStorage or attribute), and virtual/load-more/button-based pagination. The widget targets complex data-management screens where users must browse, filter, sort, and act on large lists of objects. Minimum Mendix version: 10.12.0.

---

## User Scenarios

### [P1] Browse and sort data

**Given** a Data Grid 2 with columns bound to object attributes  
**When** the page loads  
**Then** the grid renders rows from the datasource with correct column values; each sortable column header shows a bidirectional sort icon (`↕`); clicking a sortable header sorts ascending (`↑`) then descending (`↓`) on subsequent clicks

#### Edge Cases
- Sortable columns require an attribute binding; dynamic-text and custom-content columns cannot be sorted (consistency-check error in Studio Pro).
- Non-sortable columns show no sort icon and do not respond to clicks.
- On initial load, the grid shows a skeleton loader (number of rows = known item count or `pageSize`) before data arrives.

---

### [P2] Filter rows using per-column filter widgets

**Given** columns configured with filter widgets in the `filtersPlaceholder` dropzone (e.g., Datagrid Text Filter, Datagrid Dropdown Filter)  
**When** the user interacts with a filter widget  
**Then** the datasource re-queries with the active filter applied; only matching rows are displayed; the row count updates accordingly

#### Edge Cases
- Boolean filter "Yes" shows rows where the attribute is true; "No" shows false rows.
- Selecting the empty option in a dropdown filter restores all rows.
- Enum filters display enum captions as text in gridcells.
- Filter state is per-column and independent; multiple filters combine with AND logic.

---

### [P3] Select rows

**Given** `itemSelection` is set to `"Single"` or `"Multi"` with a configured selection method  
**When** the user clicks a row (row-click method) or a checkbox (checkbox method)  
**Then** the row is marked as selected (`aria-selected="true"`, `tr-selected` CSS class applied); selected state is available as data context for other widgets on the page

#### Edge Cases
- Multi-selection with row-click: plain click selects the first row; Shift+click extends selection to a range.
- Single-selection with row-click uses Shift+click as the trigger (to avoid conflicts with `onClick` actions on the same row).
- When `itemSelection = "None"`, `aria-selected` is not set on rows (per WAI-ARIA grid pattern).
- `keepSelection` preserves selected items across page navigation for single-selection mode.
- The "select all pages" flow uses a separate `SelectAllBar` and `SelectionProgressDialog` UI layer.

---

### [P4] Hide and reorder columns at runtime

**Given** one or more columns configured as `hidable`  
**When** the user opens the column selector (button in the header) and toggles a column  
**Then** the column is hidden from or shown in the grid; the column selector shows the updated checked state

#### Edge Cases
- The last visible column cannot be hidden; click, Enter, and Space on its menu item have no effect.
- Column order is changed by dragging column headers; drop indicators (before/after) appear on hover.
- Drop target logic uses ±10 px hysteresis from the midpoint to avoid jitter when hovering near the original column position.

---

### [P5] Export data to Excel

**Given** the widget has export configured  
**When** the user initiates an export  
**Then** a progress dialog with a progress bar appears; the grid fetches all datasource rows (bypassing pagination); an XLSX file is generated and downloaded; the dialog closes on completion or cancellation

#### Edge Cases
- Export columns support type hints (`exportType`): "number" (integer), "date" (locale-formatted as "M/D/YYYY"), "boolean" (string), "default" (string).
- Only columns with `exportValue`, attribute, or dynamic-text bindings are included; custom-content columns must provide `exportValue`.
- The export dialog freeze bug (pre-v3.8.1) is fixed; the dialog no longer locks the UI during large exports.

---

### [P6] Personalize column settings

**Given** `configurationStorageType` is set to `"localStorage"` or `"attribute"`  
**When** the user hides, reorders, or resizes columns  
**Then** the settings are persisted in the chosen storage; on the next page load, the grid restores the saved column configuration

#### Edge Cases
- Settings schema version 3 stores: column visibility, column order, column sizes, active sort order, and column filters.
- A `settingsHash` (CRC of all column IDs) invalidates stored settings when the column schema changes (columns added or removed).
- When `configurationStorageType = "attribute"`, an `onConfigurationChange` action fires when settings are modified, allowing Mendix logic to react.

---

### [P7] Virtual scrolling / load-more pagination

**Given** `pagingType = "virtualScrolling"` or `pagingType = "loadMore"`  
**When** the user scrolls to the bottom  
**Then** the next batch of rows loads and appends below existing rows without clearing them; a loading indicator appears below the existing rows during the fetch

#### Edge Cases
- During next-batch fetches, existing rows remain visible (no content flash).
- `Loaded rows` attribute (v3.9.0) exposes the current count of loaded rows.
- `Page`, `Page size`, and `Total count` attributes (v3.9.0) allow external Mendix logic to read and control pagination state.
- `refreshInterval` (seconds) triggers automatic datasource refresh when configured.

---

## Functional Requirements

- **FR-001:** The system MUST render a `<div role="grid">` structure with `role="rowgroup"` for header and body, `role="row"` for each row, and `role="gridcell"` for each data cell.
- **FR-002:** The system MUST support three column content modes: `"attribute"` (bound attribute value), `"dynamicText"` (expression result), and `"customContent"` (Mendix content dropzone).
- **FR-003:** Sortable columns MUST require an `attribute` binding; dynamic-text and custom-content columns MUST NOT be sortable (enforced by Studio Pro consistency check).
- **FR-004:** The system MUST support three `itemSelection` modes: `"None"`, `"Single"`, `"Multi"`.
- **FR-005:** The system MUST support two selection methods: `"checkbox"` (dedicated checkbox column) and `"rowClick"` (clicking a row selects it).
- **FR-006:** Multi-selection range extension MUST be triggered by Shift+click on the last row in the desired range.
- **FR-007:** The header checkbox MUST display an indeterminate (three-state) visual state when some but not all rows are selected.
- **FR-008:** When `itemSelection = "None"`, the system MUST NOT set `aria-selected` on row elements.
- **FR-009:** Selecting `"rowClick"` as the selection method AND `"single"` as the click trigger MUST be rejected as a consistency-check error in Studio Pro (the two handlers compete for the same DOM event).
- **FR-010:** The system MUST apply a configurable debounce before re-querying the datasource after filter changes.
- **FR-011:** The system MUST provide a column selector dropdown (`role="menu"`, items `role="menuitemcheckbox"`) for hiding/showing hidable columns at runtime.
- **FR-012:** The last visible column MUST NOT be hideable; the corresponding menu item MUST be disabled.
- **FR-013:** Column drag-and-drop reordering MUST use the HTML Drag-and-Drop API; drop indicators (before/after) MUST use ±10 px hysteresis to prevent jitter.
- **FR-014:** The system MUST support Excel export for all configured columns; the export MUST produce a valid XLSX file.
- **FR-015:** Export column types MUST be mapped: `"number"` → integer, `"date"` → locale-formatted date string, `"boolean"` → string, `"default"` → string.
- **FR-016:** The system MUST display an export progress dialog with a `@radix-ui/react-progress` progress bar during export; the dialog MUST provide a cancel action.
- **FR-017:** The system MUST support personalization persistence in two backends: `"localStorage"` (key = `{name}_{settingsHash}`) and `"attribute"` (JSON written to `EditableValue`).
- **FR-018:** Personalization schema MUST be version 3, storing: `name`, `schemaVersion`, `settingsHash`, `columns` (id + hidden + size), `columnFilters`, `customFilters`, `columnOrder`, `sortOrder`.
- **FR-019:** A structural change to columns (add/remove) MUST invalidate stored personalization via the `settingsHash`.
- **FR-020:** The system MUST support three pagination types: `"buttons"`, `"virtualScrolling"`, `"loadMore"`.
- **FR-021:** During virtual/load-more pagination next-batch fetches, existing rows MUST remain visible; the loading indicator MUST appear below existing rows.
- **FR-022:** The system MUST be operable in nested scenarios (Datagrid inside Datagrid); nested instances MUST use an isolated DI container (`isolated` flag on `ContainerProvider`) to prevent state inheritance.
- **FR-023:** The system MUST expose a JavaScript actions API via `useDataGridJSActions` for external programmatic control.
- **FR-024:** `refreshInterval` (seconds), when configured, MUST trigger automatic datasource refresh on that interval.
- **FR-025:** The system MUST apply the CSS class `widget-datagrid-selection-method-click` to the root element only when `itemSelectionMethod = "rowClick"`, enabling CSS-driven pointer cursor on rows.
- **FR-026:** The system MUST pass WCAG 2.1 AA accessibility requirements (zero axe-core violations) for both selection modes.
- **FR-027:** The system MUST render a skeleton loading state on first load; the number of skeleton rows MUST equal the known item count when available, otherwise `pageSize`.

---

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `datasource` | Datasource (list) | — | Data source | The list datasource providing the rows. |
| `columns` | List of column objects | — | Columns | Column definitions (see column props below). |
| `itemSelection` | `"None" \| "Single" \| "Multi"` | `"None"` | Selection | Row selection mode. |
| `itemSelectionMethod` | `"checkbox" \| "rowClick"` | — | Selection method | How rows are selected. Available when `itemSelection ≠ "None"`. |
| `itemSelectionMode` | `"toggle" \| "clear"` | — | Selection mode | `"toggle"`: Ctrl+click extends, Shift+click range. `"clear"`: each click replaces. |
| `autoSelectFirstRow` | Boolean | `false` | Auto-select first row | Automatically selects the first row when data loads. |
| `keepSelection` | Boolean | — | Keep selection | Preserves selected items across page changes (single-selection). |
| `selectAllMode` | — | — | Select all | Controls multi-page select-all behavior. |
| `pagingType` | `"buttons" \| "virtualScrolling" \| "loadMore"` | `"buttons"` | Pagination | Pagination type. |
| `pageSize` | Integer | — | Page size | Number of rows per page (buttons mode) or initial load size (virtual/load-more). |
| `dynamicPage` | `EditableValue<number> \| undefined` | — | Page attribute | Exposes and controls the current page number externally. |
| `dynamicPageSize` | `EditableValue<number> \| undefined` | — | Page size attribute | Exposes and controls page size externally. |
| `totalCountValue` | `EditableValue<number> \| undefined` | — | Total count attribute | Receives the total item count from the datasource. |
| `dynamicItemCount` | `EditableValue<number> \| undefined` | — | Loaded rows | Exposes the count of currently loaded rows (virtual/load-more only). |
| `refreshInterval` | Integer (seconds) | — | Refresh interval | When > 0, triggers automatic datasource refresh at this interval. |
| `columnsSortable` | Boolean | `false` | Sortable | Global gate: enables column sorting. Per-column `canSort` also required. |
| `columnsFilterable` | Boolean | `false` | Filterable | Global gate: enables column filter widgets. |
| `columnsHidable` | Boolean | `false` | Hidable | Global gate: enables column hiding via the column selector. |
| `columnsDraggable` | Boolean | `false` | Draggable | Global gate: enables column drag-and-drop reordering. |
| `columnsResizable` | Boolean | `false` | Resizable | Global gate: enables column width resizing. |
| `configurationStorageType` | `"localStorage" \| "attribute"` | `"localStorage"` | Personalization storage | Where to persist column personalization settings. |
| `configurationAttribute` | `EditableValue<string> \| undefined` | — | Personalization attribute | Attribute to store personalization JSON when `configurationStorageType = "attribute"`. |
| `onConfigurationChange` | `ActionValue \| undefined` | — | On configuration change | Fires when personalization settings change. |
| `pagingPosition` | `"top" \| "bottom" \| "both"` | `"bottom"` | Pagination position | Where pagination controls appear. |
| `customPagination` | Content dropzone | — | Custom pagination | Custom pagination content rendered in place of built-in pagination controls. |
| `filtersPlaceholder` | Content dropzone | — | Filters | Dropzone for global filter widgets placed above the grid. |
| `emptyPlaceholder` | Content dropzone | — | Empty placeholder | Content shown when the datasource returns no rows. |
| `onClick` | `ActionValue \| undefined` | — | On click | Action fired when a row is clicked (single-click trigger). |
| `onClickTrigger` | `"single" \| "double"` | — | Click trigger | Whether `onClick` fires on single-click or double-click. |
| `showEmptyPlaceholder` | Boolean | — | Show empty | Whether to render `emptyPlaceholder` when no rows exist. |

### Column Props

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `attribute` | Attribute | — | Attribute | Bound attribute for `"attribute"` content mode and for sorting. |
| `showContentAs` | `"attribute" \| "dynamicText" \| "customContent"` | `"attribute"` | Show as | Column content rendering mode. |
| `dynamicText` | Text template | — | Dynamic text | Expression for `"dynamicText"` mode. |
| `content` | Content dropzone | — | Custom content | Dropzone for `"customContent"` mode. |
| `header` | Text | — | Header | Column caption shown in header. Required for hidable columns (warning if absent). |
| `width` | `"auto" \| "fill" \| "manual"` | `"auto"` | Width | Column width mode. |
| `size` | Integer (px) | — | Size | Manual column width in pixels. Visible only when `width = "manual"`. |
| `alignment` | `"left" \| "center" \| "right"` | `"left"` | Alignment | Cell content alignment. |
| `canSort` | Boolean | `false` | Sortable | Enables sorting for this column (requires `columnsSortable = true` globally). |
| `canFilter` | Boolean | `false` | Filterable | Enables the filter widget slot for this column. |
| `hidable` | `"yes" \| "hidden" \| "no"` | `"no"` | Hidable | Column visibility control. `"yes"`: user can toggle. `"hidden"`: starts hidden. `"no"`: always visible. |
| `canDrag` | Boolean | `false` | Draggable | Enables drag-and-drop reordering for this column. |
| `canResize` | Boolean | `false` | Resizable | Enables width resizing for this column. |
| `exportType` | `"default" \| "number" \| "date" \| "boolean"` | `"default"` | Export type | Type hint for Excel export formatting. |
| `exportNumberFormat` | String | — | Number format | Excel number format string. Visible only when `exportType = "number"`. |
| `exportDateFormat` | String | — | Date format | Excel date format string. Visible only when `exportType = "date"`. |
| `exportValue` | Expression | — | Export value | Override value used during export for `"customContent"` columns. |
| `columnClass` | Expression | — | Column class | Dynamic CSS class applied to cells in this column. |

---

## Changelog

### 3.9.0 (2026-03-23)
- Accessibility: column selector changed to `role="menuitemcheckbox"`.
- Single-selection column header now has proper screen reader text.
- Added `Loaded rows` attribute (virtual/load-more only).
- Added `Page`, `Page size`, `Total count` attributes for virtual/load-more.
- Fixed pointer cursor incorrectly shown on rows in checkbox selection mode.
- Fixed `Page` attribute not syncing for button pagination.
- Fixed `Total count` for virtual/load-more modes.
- Fixed Android export crash.

### 3.8.1 (2026-02-19)
- Fixed export progress dialog freezing during large exports.

### 3.8.0 (2026-01-16)
- Added `keepSelection` for single-selection mode.
- Added `Page` and `Page size` attributes for button pagination.
- Added custom pagination dropzone.
- Added auto-select first row option.
- Added Excel export type hints (number/date format strings).
- Virtual scrolling + horizontal scroll bug fix.
- Footer spacing fix.
- Dutch translations added.

---

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] What is the exact behavior when `onClick` and `itemSelectionMethod = "rowClick"` both use `"single"` click trigger and the developer bypasses the Studio Pro consistency check (e.g., by deploying from CLI)?
- [ ] The `selectAllMode` prop signature and its exact effect on multi-page select-all behavior could not be fully traced from source alone; its values and precise semantics require documentation review.
- [ ] The `DerivedPropsGate` pattern bridges reactive MobX props into the DI container for runtime prop changes — which specific props are treated as reactive vs. frozen in `DatagridConfig` was not exhaustively enumerated.
- [ ] The `autoSelectFirstRow` feature (v3.8.0): is the first row selected after every datasource refresh, or only on the initial page load?
