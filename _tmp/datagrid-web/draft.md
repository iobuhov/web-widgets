# Draft: datagrid-web

Widget package: `@mendix/datagrid-web` (Datagrid / Data Grid 2), v3.10.0. Minimum Mendix version: 10.12.0. Entry: `src/Datagrid.tsx`. License: Apache-2.0.

---

## package.json

1. **Purpose**: Declares the widget package identity, build scripts, and runtime dependencies.
2. **Logic**: Defines npm dependencies (brandi, brandi-react, mobx, mobx-react-lite, @floating-ui/react, @radix-ui/react-progress, nanoevents, classnames) and workspace packages (@mendix/widget-plugin-filtering, @mendix/widget-plugin-grid, @mendix/widget-plugin-platform, @mendix/widget-plugin-mobx-kit, @mendix/widget-plugin-hooks, @mendix/widget-plugin-external-events, @mendix/widget-plugin-component-kit). Build tooling: pluggable-widgets-tools.
3. **Behavior documented**: The widget uses brandi v5 for dependency injection, MobX 6.12.3 for reactivity, and floating-ui for the column selector dropdown. Local workspace packages provide filtering, grid keyboard navigation, and platform abstractions.
4. **User-facing**: No — internal configuration only.
5. **New learning**: The widget depends on `@radix-ui/react-progress` for the export progress bar. `nanoevents` is used for lightweight event emission (likely in the export controller). `@mendix/widget-plugin-external-events` suggests the widget exposes a JS API for external triggers.

---

## typings/DatagridProps.d.ts

1. **Purpose**: Defines the complete TypeScript props interface (`DatagridContainerProps`) consumed by the widget root and all its subsystems.
2. **Logic**: Declares every configurable property: datasource, columns array (with per-column display, width, sort, filter, hide, drag, export settings), row selection (type, method, mode, auto-select, keep-selection, select-all), pagination (buttons/virtualScrolling/loadMore, dynamic page/pageSize attributes), personalization storage (localStorage vs attribute), UI text labels, and click actions.
3. **Behavior documented**: `showContentAs` drives column rendering mode ("attribute", "dynamicText", "customContent"). `itemSelection` is typed as "None"/"Single"/"Multi" — a string enum in the props. `configurationStorageType` switches storage between localStorage and an EditableValue attribute. Dynamic pagination attributes (`dynamicPage`, `dynamicPageSize`, `totalCountValue`, `dynamicItemCount`) allow external control of grid state. `exportType` per column supports "default", "number", "date", "boolean" with optional format strings.
4. **User-facing**: Partially — Mendix Studio Pro reads this to generate the property panel.
5. **New learning**: `onConfigurationChange` is a separate ActionValue triggered when personalization settings change, allowing Mendix logic to react to column hide/sort/resize events. `refreshInterval` in seconds triggers automatic datasource refresh.

---

## src/Datagrid.tsx

1. **Purpose**: The root widget component exported as the default export. Sets up the brandi dependency injection container and renders the `DatagridRoot` observer.
2. **Logic**: Calls `useDatagridContainer(props)` to build the DI container from props, then wraps everything in `<ContainerProvider container={container} isolated>`. `DatagridRoot` is an `observer` that reads `columnsStore` and `exportProgress` from the container, wires up `useDataExport`, `useDataGridJSActions`, and renders `<Widget />`.
3. **Behavior documented**: The `isolated` flag on `ContainerProvider` (brandi-react v5) prevents binding inheritance from parent containers — critical for nested Datagrid scenarios (Datagrid inside Datagrid). `useDataGridJSActions` registers an external JS API for the widget. Export cancellation is passed down as `onExportCancel` to `Widget`.
4. **User-facing**: No — infrastructure wiring only.
5. **New learning**: The `isolated` prop on `ContainerProvider` is explicitly required for the nested datagrid use case. Without it, nested grids would incorrectly inherit DI bindings from parent grids, causing state corruption.

---

## src/Datagrid.editorConfig.ts

1. **Purpose**: Configures Studio Pro's property panel — hides/shows properties conditionally and provides the structural preview rendered in the page editor canvas.
2. **Logic**: `getProperties` inspects `DatagridPreviewProps` and calls `hidePropertyIn`/`hidePropertiesIn`/`hideNestedPropertiesIn` to conditionally remove props from the UI. For example, `size` is hidden when `width !== "manual"`, `exportNumberFormat` when `exportType !== "number"`, filter widgets when `!columnsFilterable`, and all selection subproperties cascade from `itemSelection`. `getPreview` builds a structural preview using the preview API (container, rowLayout, dropzone, text, selectable). `check` (re-exported from consistency-check) provides validation errors.
3. **Behavior documented**: Column captions in the property panel show: header text, content source (attribute name, dynamic text expression, or "Custom content"), width mode, and alignment. Columns with `hidable !== "no"` are shown with a distinct background in the preview. `filtersPlaceholder`, `emptyPlaceholder`, and `customPagination` dropzones appear conditionally.
4. **User-facing**: Yes — Mendix Studio Pro developers see the property panel and canvas preview.
5. **New learning**: `getCustomCaption` returns the datasource caption for use as the widget name in Studio Pro's page overview. The `canHideDataSourceHeader` flag (Studio Pro >= 9.20) controls whether datasource header is hidden in dropzones.

---

## src/consistency-check.ts

1. **Purpose**: Design-time validation that runs in Studio Pro to catch misconfiguration before deployment.
2. **Logic**: `check` iterates columns and runs three validators: `checkDisplaySettings` (attribute required when `showContentAs="attribute"`), `checkSortingSettings` (attribute required when column is sortable), `checkHidableSettings` (warning when hidable column has no caption). Also runs `checkSelectionSettings` which raises an error when `onClickTrigger="single"` and `itemSelectionMethod="rowClick"` are combined — an ambiguous configuration.
3. **Behavior documented**: Sortable columns absolutely require an `attribute` binding — dynamic text or custom content cannot be sorted. The hiding warning is severity "warning" not "error". The click-method conflict is a hard error requiring the user to choose double-click trigger or checkbox selection method.
4. **User-facing**: Yes — errors/warnings appear in Studio Pro's consistency check panel, blocking deployment.
5. **New learning**: The selection ambiguity check only fires when `onClick` is non-null AND `itemSelection !== "None"`, meaning: a grid with no selection and no onClick never triggers this check. The combination of single-click row trigger + row-click selection is caught because both handlers compete for the same DOM event.

---

## src/components/Widget.tsx

1. **Purpose**: The top-level layout compositor — assembles the grid regions from named sub-components.
2. **Logic**: Renders `WidgetRoot` → `WidgetTopBar` + `WidgetHeader` + `WidgetContent` (containing `Grid` → `GridHeader` + `SelectAllBar` + `RefreshStatus` + `GridBody` → `RowsRenderer` + `MockHeader` + `EmptyPlaceholder`) + `WidgetFooter` + `SelectionProgressDialog` + `ExportProgressDialog`.
3. **Behavior documented**: `SelectAllBar` and `SelectionProgressDialog` handle the multi-page "select all" flow as separate UI layers. `ExportProgressDialog` is a modal overlay that receives the `onExportCancel` callback. `MockHeader` (inside `GridBody`) renders a skeleton header during loading states. The layout is purely compositional with no state logic.
4. **User-facing**: Yes — defines the visible structure and region ordering.
5. **New learning**: `MockHeader` lives inside `GridBody` rather than `GridHeader`, suggesting it renders a placeholder within the scrollable body area during the initial skeleton loading phase, not as a replacement for the real header.

---

## src/components/GridHeader.tsx

1. **Purpose**: Renders the header row — the `<div role="rowgroup">` containing column headers, select-all checkbox, and the column selector button.
2. **Logic**: Reads `visibleColumns` from `columnsStore`. If `!columnsStore.loaded`, renders `HeaderSkeletonLoader`. Otherwise renders one `Header` per column inside `ColumnProvider`, plus `CheckboxColumnHeader` and `ColumnSelector` (when `columnsHidable=true`). Manages drag state: `dragOver` (target column + position) and `isDragging` (prev/current/next column IDs) as React state.
3. **Behavior documented**: Only visible (non-hidden) columns appear in the header. Drag state is shared as props to all `Header` instances so each can show drop-before/drop-after indicators relative to the dragged column. `ColumnSelector` receives `availableColumns` (all columns, including hidden) for the hide/show menu.
4. **User-facing**: Yes — directly visible header row.
5. **New learning**: The drag state tracks not just the dragged column but also its immediate neighbors (previous and next column IDs). This is used in `Header` to apply hysteresis when determining drop-before vs drop-after near the dragged column's original position, avoiding jitter.

---

## src/components/Header.tsx

1. **Purpose**: Single column header cell — handles sort, drag-reorder, resize, and optional filter widget.
2. **Logic**: Reads `canDrag`, `canSort`, `canResize` from global config AND column-level flags (both must be true). Sort props add `onClick`/`onKeyDown` (Enter/Space) and `role="button"` to the inner div. `aria-sort` is set to "ascending"/"descending"/"none" when sortable. `aria-label` is "sort {caption}" for sortable columns. Sort icons cycle: `FaArrowsAltV` (unsorted), `FaLongArrowAltUp` (asc), `FaLongArrowAltDown` (desc). Drag logic uses the HTML Drag-and-Drop API with `data-column-id` dataset attributes for identification.
3. **Behavior documented**: Sort is a two-level gate: `columnsSortable` (global) AND `column.canSort` (per-column). When dragging, `pointerEvents: none` is applied to sort and filter areas to prevent accidental interactions. Drop indicator is a CSS class (`drop-before`/`drop-after`) on the column being hovered.
4. **User-facing**: Yes — directly visible and interactive.
5. **New learning**: The hysteresis in `handleDragOver` uses a ±10px bias from center: if the existing drop target is "after", the threshold shifts left by 10px before flipping to "before". This prevents jitter when hovering near the midpoint.

---

## src/components/GridBody.tsx

1. **Purpose**: The scrollable body container (role="rowgroup") plus a `ContentGuard` that manages the first-load and next-batch loading states.
2. **Logic**: `ContentGuard` (MobX observer) checks `loaderVM.isFirstLoad` and `loaderVM.isFetchingNextBatch`. On first load: shows spinner (full container) or `RowSkeletonLoader` (uses actual `itemCount` if already known, else `pageSize`). On fetching next batch: children render first, then spinner or skeleton appended below. `useBodyScroll` hook wires `onScroll` for virtual scrolling.
3. **Behavior documented**: The skeleton on first load shows a number of rows equal to the actual item count when it is already known (e.g., re-navigation with cached count), otherwise falls back to `pageSize`. During next-batch fetches, existing rows remain visible and the loading indicator appears below them — no content flash.
4. **User-facing**: Yes — loading states are user-visible.
5. **New learning**: `useBorderTop` flag on `RowSkeletonLoader` is `true` only on first load (the skeleton is the topmost visible content). During next-batch loading it is `false` because rows above already have their borders.

---

## src/components/Row.tsx

1. **Purpose**: Single data row — renders the checkbox cell, data cells, and optional selector cell for a single `ObjectItem`.
2. **Logic**: Uses `selectActions.isSelected(item)` to derive `aria-selected`. Adds `tr-selected` class to selected rows. `checkboxColumnEnabled` conditionally renders `CheckboxCell`. Maps `columns` to `DataCell` components, adjusting `columnIndex` when checkbox is present (+1 offset). `showSelectorCell` adds a `SelectorCell` placeholder at the end. First row gets `borderTop` styling.
3. **Behavior documented**: `aria-selected` is `undefined` (not set) when `selectionType === "None"` — matching the ARIA grid pattern where non-selectable grids omit the attribute. `clickable` prop controls pointer cursor on cells.
4. **User-facing**: Yes — the visible row structure.
5. **New learning**: Column index passed to `DataCell` accounts for the presence of the checkbox column (+1 offset). This is important for keyboard navigation — the `KeyNavProvider` uses these indices for focus grid positioning.

---

## src/components/DataCell.tsx

1. **Purpose**: MobX observer rendering a single data cell — connects column content rendering, cell event handlers, and keyboard navigation focus management.
2. **Logic**: `useFocusTargetProps` provides `tabIndex` and `ref` for keyboard grid navigation. `eventsController.getProps(item)` returns all event handlers (onClick, onDoubleClick, onMouseDown, onKeyDown, onKeyUp, onFocus). Cell content is computed via `computed(() => column.renderCellContent(item)).get()` — a MobX computed for fine-grained reactivity. Dynamic CSS class comes from `column.columnClass(item)`.
3. **Behavior documented**: Content rendering uses a MobX `computed` wrapper — changes to the item's observable properties trigger only the affected cell to re-render, not the entire row. `previewAsHidden` renders the cell differently in Studio Pro preview for unavailable or hidden columns.
4. **User-facing**: Yes — the cell content is user-visible.
5. **New learning**: The `computed()` call around `renderCellContent` is a deliberate MobX optimization. Without it, any observable change in the item would trigger the full `DataCell` component re-render; with it, only when the specific cell value changes.

---

## src/components/CheckboxCell.tsx & CheckboxColumnHeader.tsx

1. **Purpose**: `CheckboxCell` renders the per-row selection checkbox; `CheckboxColumnHeader` renders the select-all checkbox header.
2. **Logic**: Both use `useCheckboxEventsHandler` (or similar injection hook) for event wiring. `CheckboxColumnHeader` uses `ThreeStateCheckBox` supporting indeterminate state (some rows selected). Selection column header includes an `.sr-only` span with configurable text (e.g., "Select single row", "Select all rows") for screen reader announcement. Single-selection columns show sr-only text rather than a visible checkbox.
3. **Behavior documented**: E2E tests confirm: `.widget-datagrid-col-select .sr-only` contains "Select single row" text for single-selection checkbox mode. The sr-only element is `position: absolute` with `width: 1px` — visually hidden but in DOM. WCAG 2.1 AA violations must be zero per axe-core scan.
4. **User-facing**: Yes — checkboxes are directly interactive.
5. **New learning**: The `ThreeStateCheckBox` indeterminate state means when some (but not all) rows are selected, the header checkbox shows a mixed/indeterminate state — a standard WAI-ARIA pattern for "select all" in multi-select grids.

---

## src/components/ColumnSelector.tsx

1. **Purpose**: Floating dropdown menu (powered by @floating-ui/react) for hiding and showing columns at runtime.
2. **Logic**: Renders a button that opens a `<ul role="menu">` with `<li role="menuitemcheckbox">` items, one per hidable column. Items are disabled when only one column remains visible (preventing hiding the last column). Uses Floating UI for positioning (likely `useFloating` with flip/shift middleware). Supports keyboard interaction (Enter/Space to toggle, Escape to close).
3. **Behavior documented**: E2E confirms: clicking a column item toggles visibility. When only 1 checkbox remains checked, additional clicks and keyboard presses (Enter, Space) have no effect — the last visible column cannot be hidden. Role is `menuitemcheckbox` with `aria-checked` reflecting current visibility state.
4. **User-facing**: Yes — visible button and dropdown.
5. **New learning**: CHANGELOG v3.9.0 explicitly notes the role changed to `menuitemcheckbox` as an accessibility improvement — prior versions used a different ARIA pattern. The column selector button itself has `aria-label` for screen readers.

---

## src/components/WidgetRoot.tsx / WidgetContent.tsx / WidgetTopBar.tsx / WidgetHeader.tsx / WidgetFooter.tsx

1. **Purpose**: Layout wrapper components that provide the CSS structure and contextual class names for each region of the grid.
2. **Logic**: `WidgetRoot` is the outermost `<div class="widget-datagrid">`, and adds conditional classes for selection state (`widget-datagrid-selectable-rows`, `widget-datagrid-selection-method-checkbox`, `widget-datagrid-selection-method-click`, `widget-datagrid-selecting-all-pages`, `widget-datagrid-exporting`). `WidgetTopBar` and `WidgetFooter` render pagination based on `pagingPosition`. `WidgetHeader` renders the `filtersPlaceholder` container.
3. **Behavior documented**: The selection method class on the root element enables CSS-driven cursor changes — `widget-datagrid-selection-method-click` adds pointer cursor to rows, while `widget-datagrid-selection-method-checkbox` does not (fixed in v3.9.0).
4. **User-facing**: Yes — provides all structural CSS classes.
5. **New learning**: CHANGELOG v3.9.0 bug fix: rows incorrectly showed pointer cursor with checkbox selection. The fix involved conditionally applying the selection method class — CSS cursor style is now driven by this class, not by the `clickable` prop alone.

---

## src/features/data-export/ (ExportController, useDataExport, ExportProgressDialog)

1. **Purpose**: Full Excel export pipeline — orchestrates datasource snapshots, progress reporting, and the export modal UI.
2. **Logic**: `useDataExport` hook connects props/stores to the export controller. `ExportController` manages the export state machine: idle → loading → done/error. Uses `nanoevents` for progress events (`loadstart`, `progress`, `loadend`). `DSExportRequest` batches datasource fetches. `ExportProgressDialog` shows `@radix-ui/react-progress` progress bar. Returns abort function passed to `Widget`.
3. **Behavior documented**: Export generates an XLSX file. E2E test confirms: 50 rows exported with correct column types — dates formatted as "2/15/1983" (locale-dependent), birth year as integer 1983, enum as string "Black", name as string. Only columns with `exportValue` (for customContent) or attribute/dynamicText are included.
4. **User-facing**: Yes — dialog is user-visible during export.
5. **New learning**: The datasource must be "snapshotted" during export — the controller temporarily overrides pagination limits to fetch all data, then restores original settings. This is a side effect of the stateless datasource API.

---

## src/features/row-interaction/ (selection, cell events)

1. **Purpose**: Encapsulates all selection logic and cell event routing — keyboard selection, mouse selection, range selection, and single vs. multi-page select-all.
2. **Logic**: `CellEventsController` and `CheckboxEventsController` implement the `EventsController` interface. Selection mode "toggle" enables Ctrl+click range extension and Shift+click range selection. Mode "clear" means each click replaces the current selection. `SelectFx` calls `itemSelection.setSelection()`. `keepSelection` preserves selected items across page changes by maintaining a separate selection set.
3. **Behavior documented**: Shift+click in E2E test: first row clicked, then 5th row Shift+clicked → 5 rows selected (rows 0–4 inclusive). Single-selection row-click uses `modifiers: ["Shift"]` in tests, suggesting Shift+click is the trigger for single-selection row-click mode.
4. **User-facing**: Yes — all selection interactions are user-initiated.
5. **New learning**: Single-selection row-click mode in E2E uses Shift+click, not plain click — this is an unusual but intentional UX pattern, likely to avoid accidental selection when rows have onClick actions.

---

## src/helpers/state/ (PersonalizationStorage, ColumnsStore)

1. **Purpose**: Manages column personalization state — persisting and restoring column visibility, order, sizes, sort, and filters.
2. **Logic**: `PersonalizationStorageSettings` schema v3: stores `name`, `schemaVersion: 3`, `settingsHash` (hash of all columnIds), `columns[]` (id, hidden, size), `columnFilters[]`, `customFilters[]`, `columnOrder[]`, `sortOrder[]`. Two backends: `LocalStoragePersonalizationStorage` (key = `{name}_{hash}`) and `AttributePersonalizationStorage` (reads/writes JSON to an EditableValue). `ColumnsStore` observes changes and writes back reactively.
3. **Behavior documented**: E2E confirms the full JSON structure saved to attribute storage: `{name, schemaVersion: 3, settingsHash: "1530160614", columns: [{columnId, hidden}], columnFilters: [], customFilters: [], sortOrder: [], columnOrder: ["0","1"]}`. Settings hash encodes column IDs — structural changes (adding/removing columns) invalidate stored settings.
4. **User-facing**: No — background persistence only.
5. **New learning**: `columnId` values are string indices ("0", "1", etc.) not semantic names — order is positional, not by attribute name. `settingsHash` is a numeric string (CRC-style), computed from concatenated columnIds.

---

## src/model/ (DI container, configs, services)

1. **Purpose**: The dependency injection wiring layer — creates the brandi container, binds all services and stores, and exposes typed injection hooks.
2. **Logic**: `useDatagridContainer(props)` is called once on mount to build the container. Static `DatagridConfig` is derived from props and frozen (no reactivity needed for configuration that doesn't change at runtime). Tokens in `model/tokens.ts` define typed DI keys for every injected service. `DerivedPropsGate` pattern bridges reactive MobX props into the DI container.
3. **Behavior documented**: `DatagridConfig.isInteractive` is true when the widget has an onClick action or row selection enabled — used to apply the `clickable` prop to cells. The container is isolated per widget instance (`isolated` flag), ensuring nested grids do not share state.
4. **User-facing**: No — pure infrastructure.
5. **New learning**: `DatagridConfig` is frozen at mount time from the initial props. Runtime prop changes go through `DerivedPropsGate` reactive atoms rather than through the config object — keeping config as a stable reference for non-reactive consumers.

---

## e2e/DataGrid.spec.js

1. **Purpose**: E2E tests for core capabilities: export, sorting, column hiding, onClick context, manual column width, visual regression, and accessibility.
2. **Logic**: Uses Playwright + axe-core. Export test: clicks export button, downloads XLSX, parses with `xlsx` library, verifies 50 rows and first two rows' field values and types. Sorting: clicks column header, verifies SVG icon changes (arrows-alt-v → long-arrow-alt-up → long-arrow-alt-down), verifies first visible gridcell value. Hiding: uses `.column-selector-button` and `.column-selectors > li` to toggle; verifies last-column protection.
3. **Behavior documented**: Default sort shows `arrows-alt-v` icon (bidirectional). ASC sort shows `long-arrow-alt-up`. DESC shows `long-arrow-alt-down`. Export produces 4 columns: "First name" (string), "Birth date" (date, locale-formatted "M/D/YYYY"), "Birth year" (integer), "Color (enum)" (string). Hiding last column is prevented by click, Enter, and Space key — all have no effect. WCAG 2.1 AA scan passes with zero violations.
4. **User-facing**: Tests are not user-facing; the behaviors they verify are.
5. **New learning**: The axe-core scan excludes `.mx-name-navigationTree3` — an external navigation widget known to have accessibility issues not owned by this widget. This is a deliberate exclusion pattern to prevent false positives.

---

## e2e/DataGridSelection.spec.js

1. **Purpose**: E2E tests for single and multi selection modes (checkbox and row-click), plus accessibility validation.
2. **Logic**: Tests single checkbox selection (click first input → screenshot), single row-click (Shift+click first `.td`), multi checkbox (click two inputs → screenshot), multi row-click (click first `.td`, Shift+click 5th `.td`). Accessibility tests: verifies `.widget-datagrid-col-select .sr-only` text contains "Select single row" and is `position: absolute` with `width: 1px`. WCAG 2.1 AA scan on both single and multi selection grids.
3. **Behavior documented**: Single-selection row-click uses `Shift` modifier. Multi-selection row-click uses plain click + Shift+click for range selection. Sr-only column header text exists for single-selection checkbox grids. WCAG 2.1 AA passes for both modes.
4. **User-facing**: Tests are not user-facing; the behaviors they verify are.
5. **New learning**: `isHidden` check in the accessibility test verifies `position === "absolute" && (width === "1px" || clip === "rect(0px, 0px, 0px, 0px)")` — this is the actual `.sr-only` CSS implementation pattern used here (both clip-based and width-based variants checked).

---

## e2e/filtering/DataGridFilteringSingle.spec.js

1. **Purpose**: E2E tests for per-column filter widgets — boolean dropdown and enum dropdown filters.
2. **Logic**: Tests boolean filter (Yes/No selection → row count changes to 11), enum filter (Cyan → 9 rows, Red/Blue values confirmed), text filter (company name), and filter reset (empty option restores all rows). Uses `role="combobox"` for filter dropdowns and `role="option"` for options.
3. **Behavior documented**: Filtering reduces visible rows matching the filter value. Boolean filter: 10 data rows + 1 header = 11 total `role="row"` elements when filtering "Yes". Reset: clicking the empty option in a dropdown restores all rows. Enum values ("Cyan", "Red", "Blue") render as text in gridcells.
4. **User-facing**: Tests are not user-facing; the behaviors they verify are.
5. **New learning**: The filter dropdown uses `role="combobox"` — this is from the `@mendix/widget-plugin-filtering` drop_down filter widget. Filter state is per-column and applied independently. The row count includes the header row in the `[role="row"]` selector count.

---

## CHANGELOG.md (last 3 versions)

1. **Purpose**: Records notable changes by version for users upgrading the widget.
2. **Logic**: v3.9.0 (2026-03-23): Accessibility improvements (column selector role → `menuitemcheckbox`), single-selection column header accessibility, `Loaded rows` attribute for virtual/load-more modes, `Page`/`Page size`/`Total count` attributes for virtual/load-more, cursor fix (no pointer on checkbox selection), Android export crash fix, `Page` attribute sync fix for button pagination, `Total count` fix for virtual/load-more. v3.8.1 (2026-02-19): Export dialog freeze fix. v3.8.0 (2026-01-16): Footer spacing fix, `keepSelection` for single selection, virtual scrolling + horizontal scroll fix, Dutch translations, Excel export types (number/date format), `Page`/`Page size` attributes, custom pagination, auto-select first row.
3. **Behavior documented**: `Page` attribute was broken for button pagination until v3.9.0. `Total count` for virtual/load-more was broken until v3.9.0. Export dialog could freeze until v3.8.1 fix. Pointer cursor on checkbox-mode rows was incorrect until v3.9.0.
4. **User-facing**: Yes — release notes visible to Mendix developers.
5. **New learning**: The `Loaded rows` attribute (v3.9.0) only applies to virtual scrolling and load-more pagination modes — not buttons mode (which has full page data always). The `Page`/`Page size` attributes were introduced in v3.8.0 for buttons mode, then extended in v3.9.0 for virtual/load-more.
