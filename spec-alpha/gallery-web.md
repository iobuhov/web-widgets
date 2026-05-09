# Gallery

## Purpose

The Gallery widget renders a responsive, configurable grid of Mendix data items with support for filtering, sorting, single and multi-item selection, multiple pagination strategies, personalization (filter/sort state persistence), and keyboard-accessible navigation. It is suited for use cases where a developer needs to display a list of items with custom content layouts — such as product catalogs, employee directories, image grids, or document libraries — and requires built-in integration with Mendix filter, sort, and selection plugin widgets.

---

## User Scenarios

### [P1] Browse items in a responsive grid

**Given** a datasource is connected and items are available  
**When** the user loads a page containing the Gallery  
**Then** items are displayed in a responsive grid whose column count adapts to viewport width: desktop (≥ 992 px) uses `desktopItems`, tablet (768–991 px) uses `tabletItems`, phone (< 768 px) uses `phoneItems`; column count is capped at the actual item count so no empty columns are shown

#### Edge Cases
- When the datasource is empty and `showEmptyPlaceholder` is `"custom"`, the configured empty placeholder widget is rendered instead of the item grid.
- When `dynamicPageSize` attribute is connected, no items are loaded on the initial render (page size = 0) until the attribute provides a value.

### [P2] Filter and sort items

**Given** filter and/or sort widgets are placed in the `filtersPlaceholder` slot  
**When** the user interacts with a filter (text, date, number, dropdown) or sort widget  
**Then** the Gallery re-queries its datasource with the updated filter condition or sort order and re-renders with the matching items; the refresh indicator (progress bar) is shown when `refreshIndicator` is enabled and the refresh is not silent

#### Edge Cases
- Multiple filter widgets in the filters placeholder accumulate filter conditions independently.
- If filter or sort state is persisted (`storeFilters` / `storeSort`), the stored state is applied immediately on first render before any user interaction.

### [P3] Select a single item

**Given** `itemSelection` is set to `"Single"`  
**When** the user clicks an item (with `onClickTrigger = "single"`) or presses Space/Enter on a focused item  
**Then** that item is selected; any previously selected item is deselected; `onSelectionChange` fires

#### Edge Cases
- If `itemSelectionMode = "toggle"`, clicking the already-selected item deselects it. If `itemSelectionMode = "clear"`, clicking it keeps it selected.
- If `autoSelect = true`, the first item is automatically selected on initial render.
- Clicking with Ctrl/Meta held does NOT trigger the onClick action (reserved for selection modifiers).

### [P4] Select multiple items

**Given** `itemSelection` is set to `"Multi"`  
**When** the user clicks items while holding Shift  
**Then** all items between the anchor and the clicked item are selected (range selection); `onSelectionChange` fires

**When** the user presses Ctrl/Cmd+A  
**Then** all items in the datasource are selected

**When** the user presses Shift+Space on a focused item  
**Then** that item's selection is toggled (not the onClick action)

**When** the user presses Shift+Arrow (Up/Down/Left/Right) on a focused item  
**Then** the selection is extended in that direction within the 2D grid

#### Edge Cases
- When `keepSelection = true`, selection is retained across page/filter changes.
- A selection counter showing the number of selected items is displayed at the position configured by `selectionCountPosition` (top/bottom/off).
- The clear selection button label is configured via `clearSelectionButtonLabel`.

### [P5] Click an item to execute an action

**Given** `onClick` is configured and `itemSelection` is set to `"None"`  
**When** the user clicks (or double-clicks, per `onClickTrigger`) an item, or presses Space or Enter on a focused item  
**Then** the configured `onClick` list action fires with the clicked item's data context

**When** `onClickTrigger = "single"` AND `itemSelection` is not `"None"`  
**Then** Studio Pro reports a design-time error: "The item click action is ambiguous." The developer MUST use `onClickTrigger = "double"` or disable selection to resolve.

#### Edge Cases
- Space+Shift does NOT fire the onClick action — it triggers selection instead.
- Keyboard events originating from an `<input>` or `<textarea>` inside a gallery item's content slot do NOT propagate to gallery navigation or action handlers.

### [P6] Navigate items via keyboard

**Given** items are rendered in a grid  
**When** the user focuses a gallery item and presses arrow keys (Up/Down/Left/Right)  
**Then** focus moves through the 2D grid: left/right moves within a row, up/down moves between rows by column count

#### Edge Cases
- Non-selectable items (selection = "None") are not individually focusable via Tab — keyboard navigation is provided by the KeyNavProvider mechanism only.
- Selectable items have `role="option"` and `tabIndex=0`, participating in native tab order.

### [P7] Paginate through items

**Given** `pagination` is set to `"buttons"`  
**When** the user clicks the paging navigation buttons  
**Then** the datasource advances to the next or previous page; total page count is shown when `showTotalCount` is enabled

**Given** `pagination` is set to `"virtualScrolling"`  
**When** the user scrolls to the bottom of the item list  
**Then** the next batch of items is loaded and appended to the existing list

**Given** `pagination` is set to `"loadMore"`  
**When** the user clicks the Load More button (`loadMoreButtonCaption`)  
**Then** the next batch of items is appended to the existing list

#### Edge Cases
- When `dynamicPage` and `dynamicPageSize` Integer attributes are connected, external logic can programmatically control the current page and page size.
- `totalCountValue` (write-back Integer attribute) is updated with the total item count when `requestTotalCount` is enabled.
- Virtual scrolling and Load More use limit-based (not offset) pagination (`isLimitBased = true`).
- `customPagination` drop zone can replace built-in paging controls; `useCustomPagination` must be enabled.

### [P8] Personalize filter and sort state

**Given** `storeFilters` and/or `storeSort` are enabled  
**When** the user changes filter or sort settings  
**Then** the state is written to the configured storage backend (debounced 250 ms) and survives page reload

**When** the user returns to the page  
**Then** the stored filter/sort state is applied immediately on first render before any interaction

#### Edge Cases
- With `stateStorageType = "attribute"`, settings are stored in a Mendix Unlimited String attribute. Settings are per-user if the attribute is on a user entity.
- With `stateStorageType = "localStorage"`, settings are stored in browser localStorage. Settings are shared across all Mendix users on the same browser profile.
- Malformed JSON or schema version mismatch (schema version ≠ 1) silently resets storage to null.

### [P9] Auto-refresh items on interval

**Given** `refreshInterval` > 0 (seconds)  
**When** the configured interval elapses  
**Then** the datasource is re-queried; the refresh indicator is shown only if `showSilentRefresh` is also enabled; an explicit user-triggered refresh always shows the indicator when `refreshIndicator` is on

---

## Functional Requirements

- **FR-001**: The system MUST render items in a responsive grid with independently configurable column counts for desktop (≥ 992 px), tablet (768–991 px), and phone (< 768 px) breakpoints. Each breakpoint accepts 1–12 columns.
- **FR-002**: The actual number of rendered columns MUST be capped at the current item count so that no empty columns are displayed.
- **FR-003**: The system MUST support three pagination modes: `buttons` (offset-based paging), `virtualScrolling` (limit-based infinite scroll), and `loadMore` (limit-based load-more button). Custom pagination via drop zone is supported as an alternative to all three.
- **FR-004**: The system MUST support three item selection modes: `None`, `Single`, and `Multi`. The `itemSelection` prop MUST accept the Mendix `SelectionSingleValue | SelectionMultiValue` SDK types; absence of the prop MUST disable selection.
- **FR-005**: In Multi selection mode, the system MUST support range selection via Shift+click, individual toggle via Shift+Space, range extension via Shift+Arrow keys (Up/Down/Left/Right/PageUp/PageDown/Home/End), and select-all via Ctrl/Cmd+A.
- **FR-006**: Click and keyboard actions (onClick) MUST NOT fire when Ctrl or Meta modifier is held on click. Space+Shift MUST trigger selection, not the onClick action.
- **FR-007**: When `onClickTrigger = "single"` AND `itemSelection ≠ "None"` AND `onClick` is configured, Studio Pro MUST report a validation error: "The item click action is ambiguous."
- **FR-008**: Keyboard events originating from `HTMLInputElement` or `HTMLTextAreaElement` elements inside a gallery item's content MUST NOT trigger gallery navigation or action handlers.
- **FR-009**: The system MUST provide three React context APIs to widgets in the `filtersPlaceholder` slot: `FilterAPI`, `SortAPI`, and `SelectionContext`. If no widgets are placed in the slot, the header section MUST be omitted from the DOM.
- **FR-010**: The system MUST support personalization of filter and sort state. Storage MUST be debounced by 250 ms. Invalid or version-mismatched stored state MUST be silently discarded and storage reset.
- **FR-011**: When `stateStorageType = "attribute"`, the target attribute MUST be an `EditableValue<string>` (Mendix Unlimited String). When `stateStorageType = "localStorage"`, settings are keyed by widget instance ID.
- **FR-012**: When `refreshInterval > 0`, the datasource MUST be re-queried automatically on the configured interval. Silent refreshes MUST NOT display the progress bar unless `showSilentRefresh` is enabled.
- **FR-013**: Each widget instance MUST have a unique ID (`${name}:Gallery@${uuid}`) to prevent cross-instance state leakage, especially for filter channel names and DI container isolation.
- **FR-014**: The system MUST assign `role="listbox"` with `aria-multiselectable` to the items container when selection is enabled; `role="list"` when selection is `None`.
- **FR-015**: Selectable items MUST have `role="option"`, `aria-selected`, and `tabIndex=0`. Non-selectable items MUST have `role="listitem"` with no tabIndex.
- **FR-016**: The widget MUST support per-item dynamic CSS class expressions (`itemClass`), per-item ARIA label expressions (`ariaLabelItem`), and a listbox ARIA label (`ariaLabelListBox`).
- **FR-017**: When `dynamicPageSize` is connected, the initial datasource fetch MUST request 0 items; the real page size MUST be set once the attribute value is available.
- **FR-018**: The widget MUST be offline-capable.

---

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `datasource` | `ListValue` | — | Data source | The items data source |
| `content` | `ListWidgetValue` | — | Content | Widget(s) rendered per item |
| `filtersPlaceholder` | `ListWidgetValue?` | — | Filters | Slot for filter/sort widgets; provides FilterAPI, SortAPI, SelectionContext |
| `desktopItems` | `number` (1–12) | `4` | Desktop items | Column count at desktop breakpoint (≥ 992 px) |
| `tabletItems` | `number` (1–12) | `3` | Tablet items | Column count at tablet breakpoint (768–991 px) |
| `phoneItems` | `number` (1–12) | `2` | Phone items | Column count at phone breakpoint (< 768 px) |
| `itemSelection` | `SelectionSingleValue \| SelectionMultiValue \| undefined` | — | Selection | Enables selection; None if absent |
| `itemSelectionMode` | `"toggle" \| "clear"` | `"toggle"` | Selection mode | Clicking selected item: toggle deselects, clear keeps selected |
| `autoSelect` | `boolean` | `false` | Auto-select first item | Automatically selects first item on render |
| `keepSelection` | `boolean` | `false` | Keep selection | Retains selection across page/filter changes (Multi only) |
| `selectionCountPosition` | `"top" \| "bottom" \| "off"` | `"off"` | Selection count | Position of selection counter (Multi only) |
| `clearSelectionButtonLabel` | `DynamicValue<string>?` | — | Clear selection label | Label for the clear-selection button (Multi only) |
| `onSelectionChange` | `ActionValue?` | — | On selection change | Action fired when selection changes |
| `onClick` | `ListActionValue?` | — | On click | Per-item action |
| `onClickTrigger` | `"single" \| "double"` | `"single"` | Click trigger | Mouse event that fires onClick |
| `pagination` | `"buttons" \| "virtualScrolling" \| "loadMore"` | `"buttons"` | Pagination | Pagination strategy |
| `showPagingButtons` | `"always" \| "auto"` | `"auto"` | Show paging buttons | When to show paging controls (buttons mode) |
| `showTotalCount` | `boolean` | `false` | Show total count | Display total item count in paging controls |
| `loadMoreButtonCaption` | `DynamicValue<string>?` | — | Load more caption | Label for the Load More button |
| `useCustomPagination` | `boolean` | `false` | Use custom pagination | Replaces built-in pagination with drop zone |
| `customPagination` | `widget?` | — | Custom pagination | Drop zone for custom pagination widget |
| `pagingPosition` | `"top" \| "bottom" \| "both"` | `"bottom"` | Paging position | Where paging controls are rendered |
| `dynamicPage` | `EditableValue<Big>?` | — | Page (attribute) | Integer attribute for programmatic page control |
| `dynamicPageSize` | `EditableValue<Big>?` | — | Page size (attribute) | Integer attribute for programmatic page size |
| `totalCountValue` | `EditableValue<Big>?` | — | Total count (attribute) | Write-back Integer attribute receiving total item count |
| `dynamicItemCount` | `EditableValue<Big>?` | — | Loaded count (attribute) | Write-back Integer attribute receiving loaded item count |
| `showEmptyPlaceholder` | `"none" \| "custom"` | `"none"` | Empty placeholder | Show custom widget when datasource is empty |
| `emptyPlaceholder` | `widget?` | — | Empty placeholder slot | Widget rendered when datasource is empty |
| `itemClass` | `ListExpressionValue<string>?` | — | Item class | Per-item dynamic CSS class expression |
| `storeFilters` | `boolean` | `false` | Store filters | Persist filter state between sessions |
| `storeSort` | `boolean` | `false` | Store sort | Persist sort state between sessions |
| `stateStorageType` | `"attribute" \| "localStorage"` | `"localStorage"` | Storage type | Backend for personalization persistence |
| `stateStorageAttr` | `EditableValue<string>?` | — | Storage attribute | Mendix Unlimited String attribute (attribute mode only) |
| `onConfigurationChange` | `ActionValue?` | — | On configuration change | Action fired when stored personalization changes |
| `refreshInterval` | `number` (s) | `0` | Refresh interval | Auto-refresh interval; 0 = disabled |
| `showRefreshIndicator` | `boolean` | `false` | Show refresh indicator | Display progress bar during datasource refresh |
| `ariaLabelListBox` | `DynamicValue<string>?` | — | Listbox aria-label | Accessible label for the items container |
| `ariaLabelItem` | `ListExpressionValue<string>?` | — | Item aria-label | Per-item accessible label expression |
| `filterSectionTitle` | `DynamicValue<string>?` | — | Filter section title | Aria-label for the filter header landmark |
| `emptyMessageTitle` | `DynamicValue<string>?` | — | Empty message title | Aria-label for the empty state section |

---

## Changelog

### v3.9.0 (2026-03-23)
- **Added**: Dynamic pagination attributes (`dynamicPage`, `dynamicPageSize`, `totalCountValue`, `dynamicItemCount`); custom pagination drop zone; `autoSelect` feature for Single and Multi selection.

### v3.8.0 (2026-01-16)
- **Added**: Dutch translations; `refreshInterval` property for automatic datasource refresh; configurable `selectionCountPosition`; customizable `clearSelectionButtonLabel`.
- **Fixed**: Footer spacing; row count display in virtual scroll.

### v3.7.0 (2025-11-11)
- **Added**: Configurable selection count visibility (top/bottom/off); customizable clear selection button label.

---

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] The breakpoints (768 px, 992 px) are hardcoded in `Layout.service.ts` and independently in CSS. Is there a documented requirement for custom-variable breakpoint support beyond what v3.0.1 fixed in CSS?
- [ ] `BrowserStorage` shares settings across all Mendix users on the same browser. Is this intended behavior, or should each Mendix user have an isolated localStorage key (e.g., keyed by user name or session ID)?
- [ ] v3.2.0 removed XPath metadata from personalization storage, which invalidates settings stored before that release. Is there a migration path or user notification for this breaking change?
- [ ] Selection behavior within an iframe (fixed in v3.6.0): are there known remaining edge cases with Gallery used in Mendix popup dialogs?
- [ ] Personalization schema version is hardcoded as `1`. What is the upgrade strategy when a future schema change necessitates a version bump?
