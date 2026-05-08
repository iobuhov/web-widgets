# Combobox (combobox-web)

## Purpose

The Combobox widget provides a searchable dropdown selection control for single and multi-select scenarios in Mendix. It supports three data sources (context association/enum/boolean, database, and static values), client- and server-side filtering, lazy loading for large datasets, removable tag chips for multi-select display, SelectAll, custom footer content, and five distinct event hooks. Minimum Mendix version: 10.22.0.

## User Scenarios

### [P1] Single selection from a dropdown
**Given** a Combobox bound to an association attribute (single Reference)  
**When** the user clicks the dropdown toggle or starts typing  
**Then** the dropdown opens showing matching options; selecting an option closes the menu and sets the Mendix attribute  

#### Edge Cases
- The input field shows the search term while typing; the selected caption is shown in the trigger area (not the input)
- Pressing Escape closes the menu and clears the search term
- If no options match, the configured `noOptionsText` is shown
- When `readOnlyStyle=text`, the widget renders as plain text with no input; when `readOnlyStyle=bordered`, a disabled input is shown

### [P2] Multi-select with tag display (boxes mode)
**Given** a Combobox bound to a ReferenceSet with `selectedItemsStyle=boxes`  
**When** the user selects multiple options  
**Then** each selected item appears as a removable tag chip; the menu stays open after each selection  

#### Edge Cases
- Pressing Backspace on an empty input moves keyboard focus to the last tag for removal
- The X button on a tag removes that item from the selection
- Clear All button (when visible) removes all selected items at once
- When `selectedItemsStyle=text`, selected items are shown as comma-separated text below the input without individual remove buttons

### [P3] Multi-select with SelectAll
**Given** a Combobox with `selectAllButton=true` in multi-select mode  
**When** the dropdown is open  
**Then** a SelectAll checkbox appears in the menu header; clicking it selects or deselects all visible options  

#### Edge Cases
- The SelectAll checkbox shows an indeterminate state when some (but not all) options are selected
- SelectAll is disabled when no options are available
- SelectAll affects only the currently loaded options (not unloaded pages when lazy loading is active)

### [P4] Filter options by typing
**Given** a Combobox with `filterType=contains` (or startsWith/containsExact)  
**When** the user types in the input field  
**Then** the option list is filtered to show only matching options  

#### Edge Cases
- For enum, static, and association sources: filtering is performed client-side using `match-sorter` fuzzy matching
- For database sources: filtering creates a Mendix `FilterCondition` and applies it server-side
- The `onChangeFilterInputEvent` action fires after `filterInputDebounceInterval` ms (default 200ms) of no typing
- When `filterType=none`, the input does not filter options; the full list is always shown

### [P5] Lazy loading for large datasets
**Given** a Combobox with `lazyLoading=true` and a large data source  
**When** the dropdown is opened  
**Then** an initial batch of items is loaded; as the user scrolls to the bottom, more items load incrementally  

#### Edge Cases
- A spinner or skeleton loader (configurable via `loadingType`) is shown while items are loading
- On filter text change, the loaded items are reset and a fresh page is loaded
- When `lazyLoading=false`, all items are loaded upfront

### [P6] Custom footer content
**Given** a Combobox with `showFooter=true` and a widget configured in the footer slot  
**When** the dropdown is open  
**Then** the custom widget (e.g., a "Create new item" button) is rendered below the option list  

#### Edge Cases
- Footer content mousedown events do not close the menu
- Header content (SelectAll) mousedown events also do not close the menu

### [P7] Enter and leave events for form validation
**Given** a Combobox with `onEnterEvent` and `onLeaveEvent` configured  
**When** the user focuses into the widget  
**Then** `onEnterEvent` fires; when the user moves focus completely outside the widget, `onLeaveEvent` fires  

#### Edge Cases
- `onLeaveEvent` does NOT fire when focus moves between child elements within the widget (e.g., from input to dropdown option); it fires only on true widget-level blur
- This makes `onLeaveEvent` safe to use as a form-field validation trigger

## Functional Requirements

- FR-001: The widget MUST support three data sources: context (association Reference/ReferenceSet, enum, boolean), database, and static values.
- FR-002: For single-select, the widget MUST route to `SingleSelection`; for multi-select (ReferenceSet or database multi), the widget MUST route to `MultiSelection`.
- FR-003: The multi-select menu MUST remain open after each item selection to allow rapid multi-selection.
- FR-004: Client-side filtering (enum, static, association) MUST use `match-sorter` for fuzzy matching. Server-side filtering (database) MUST use Mendix `FilterCondition` builders.
- FR-005: The `onChangeFilterInputEvent` action MUST be debounced by `filterInputDebounceInterval` ms (default 200 ms).
- FR-006: When `lazyLoading=true`, the widget MUST load more items when the user scrolls near the bottom of the dropdown. Items MUST be reset and reloaded when the filter text changes.
- FR-007: When `lazyLoading=true`, a `loadingType` indicator (spinner or skeleton) MUST be shown while items are loading.
- FR-008: The SelectAll checkbox MUST support three states: checked (all selected), unchecked (none selected), indeterminate (some selected).
- FR-009: The `onLeaveEvent` action MUST fire only when focus moves completely outside the widget (not between child elements).
- FR-010: The menu MUST flip its position above the input when there is insufficient viewport space below.
- FR-011: Header and footer content mousedown events MUST NOT close the dropdown menu.
- FR-012: In `readOnlyStyle=text` mode, the widget MUST render as plain static text with no interactive controls.
- FR-013: Backspace on an empty input in multi-select mode MUST move keyboard focus to the last selected tag.
- FR-014: The widget MUST be compatible with the Mendix React client (`reactReady: true`).
- FR-015: When `filterType=none`, the option list MUST NOT be filtered regardless of input content.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `optionsSourceType` | `"context"` \| `"database"` \| `"static"` | `"context"` | Data source | The type of data source: context (association/enum/boolean), database list, or static values. |
| `filterType` | `"contains"` \| `"containsExact"` \| `"startsWith"` \| `"none"` | `"contains"` | Filter type | How user input filters the option list. `none` disables filtering. |
| `filterInputDebounceInterval` | `number` | `200` | Debounce interval (ms) | Milliseconds to wait after typing before firing `onChangeFilterInputEvent`. |
| `selectionMethod` | `"checkbox"` \| `"rowclick"` | `"checkbox"` | Selection method | Multi-select item interaction: checkbox or row click. |
| `selectedItemsStyle` | `"text"` \| `"boxes"` | `"boxes"` | Selected items display | How selected items appear in multi-select mode: comma-separated text or removable tag chips. |
| `selectAllButton` | `boolean` | `false` | Show SelectAll | Adds a tri-state SelectAll checkbox in the multi-select menu header. |
| `showFooter` | `boolean` | `false` | Show footer | Enables the custom content slot below the option list. |
| `menuFooterContent` | `ListWidgetValue` / `widgets` | — | Footer content | Widget(s) rendered in the dropdown footer (e.g., "Create new" button). |
| `lazyLoading` | `boolean` | `false` | Lazy loading | Enables scroll-based incremental loading of data source items. |
| `loadingType` | `"spinner"` \| `"skeleton"` | `"spinner"` | Loading indicator | Style of loading indicator shown during lazy load. |
| `noOptionsText` | `string` | — | No options text | Message shown when no options match the current filter. |
| `clearable` | `boolean` | `true` | Clearable | Shows a clear button to remove the current selection. |
| `readOnlyStyle` | `"text"` \| `"bordered"` | `"bordered"` | Read-only style | How the widget displays in read-only mode: plain text or disabled input. |
| `ariaRequired` | `boolean` | `false` | Required | Sets `aria-required` on the input element. |
| `ariaLabel` | `string` | — | ARIA label | Accessible label for the input. |
| `ariaLabelFor` | `string` | — | ARIA label for | ID of an external element that labels this input. |
| `onChangeEvent` | `ActionValue` | — | On change | Action fired when the selection changes (context/static source). |
| `onChangeDatabaseEvent` | `ActionValue` | — | On change (database) | Action fired when the selection changes (database source). |
| `onEnterEvent` | `ActionValue` | — | On enter | Action fired when the widget gains focus. |
| `onLeaveEvent` | `ActionValue` | — | On leave | Action fired when focus moves completely outside the widget. |
| `onChangeFilterInputEvent` | `ActionValue` | — | On filter change | Action fired (debounced) when the filter input text changes; enables server-side search patterns. |

## Changelog

**2.8.0** — Fixed "On selection" event not firing for database source.

**1.6.0** — Added lazy loading with spinner/skeleton loaders.

**1.5.0** — Added read-only style option.

**1.3.0** — Added static values data source support.

**1.2.0** — Added database list data source.

**1.1.0** — Added SelectAll button and footer content slot.

**1.0.0** — Initial release.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] When `filterType=none` is used with `lazyLoading=true`, what is the initial load limit and when does lazy load trigger?
- [ ] Does SelectAll in multi-select mode with lazy loading select only loaded items or trigger a full load?
- [ ] What is the behavior when `onChangeFilterInputEvent` is used together with `filterType=none`?
- [ ] Is the `optionsSourceAssociationCustomContent` / `optionsSourceDatabaseCustomContent` per-option custom widget slot limited in any way (e.g., no interactive elements)?
