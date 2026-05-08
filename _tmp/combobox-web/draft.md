# combobox-web — Source Extraction Draft

Widget: `@mendix/combobox-web` v2.8.0  
Package: `com.mendix.widget.web.Combobox.mpk`  
Min Mendix: 10.22.0, reactReady: true  
Key deps: `downshift ^7.6.2`, `match-sorter ^8.1.0`

---

## src/Combobox.tsx

**1. Purpose:** Root container component — initializes the selector and routes rendering to `SingleSelection` or `MultiSelection`.

**2. Logic:**
- Calls `useGetSelector(props)` to get cached selector instance
- Builds `commonProps` shared by both modes: tabIndex, inputId, labelId, readOnlyStyle, ariaRequired, ariaLabel, noOptionsText, plus event handlers (`onEnterEvent`, `onLeaveEvent`, `onChangeFilterInputEvent`)
- Routes on `selector.type`: "multi" → `<MultiSelection>`, else → `<SingleSelection>`
- Passes `filterType`, `selectedItemsStyle`, `selectionMethod`, `showFooter`, `filterInputDebounceInterval`, custom content props

**3. Behavioral documentation:** Similar to checkbox-radio-selection but adds search/filter capability, lazy loading, multi-select tags mode, SelectAll, footer content, and 5 distinct events. Single vs multi routing uses the same 6-selector factory pattern.

**4. User-facing?** Yes — container component.

**5. New learnings:** `filterInputDebounceInterval` (configurable debounce delay) is passed through to `useGetSelector` — the debounce is managed at the hook level, not the component. The widget has both onChange (for context/static) and onChangeDatabase (for database source).

---

## src/Combobox.xml

**1. Purpose:** Widget property descriptor — 7 property groups, supporting 3 sources and all selection modes.

**2. Logic:** Key properties beyond checkbox-radio-selection-web equivalents:
- `filterType` (contains/containsExact/startsWith/none) — how user input filters the option list
- `filterInputDebounceInterval` — integer (ms) for debouncing `onChangeFilterInputEvent`
- `selectionMethod` (checkbox/rowclick) — how multi-select menu items are selected
- `selectedItemsStyle` (text/boxes) — multi-select display: comma-separated text or removable tags
- `selectAllButton` — boolean to show/hide SelectAll in multi menu header
- `showFooter` — boolean for custom content in menu footer
- `menuFooterContent` — widget slot for footer (ListWidgetValue for association/database, widgets for static)
- 5 events: `onChangeEvent`, `onChangeDatabaseEvent`, `onEnterEvent`, `onLeaveEvent`, `onChangeFilterInputEvent`
- `optionsSourceAssociationCustomContent` / `optionsSourceDatabaseCustomContent` — per-option widget slots
- `lazyLoading` — boolean; `loadingType` (spinner/skeleton)
- `ariaRequired`, `ariaLabel`, `ariaLabelFor` — accessibility

**3. Behavioral documentation:** The combobox is significantly more configurable than checkbox-radio-selection. Filter type determines server-side vs client-side filtering: "none" disables filtering. `lazyLoading` enables incremental datasource loading as the user scrolls the dropdown. Footer allows embedding additional widgets (e.g., "Create new" button) below the options list.

**4. User-facing?** Yes — Studio Pro property panel.

**5. New learnings:** `onChangeFilterInputEvent` is a separate action (not onChange) triggered when the filter input text changes — enables server-side search patterns. `ariaLabelFor` is a third accessibility prop distinct from `ariaLabel` and the Mendix Label system.

---

## src/helpers/types.ts

**1. Purpose:** Core TypeScript interfaces — extends similar pattern from checkbox-radio-selection with additional combobox-specific features.

**2. Logic:** 
- `OptionsProvider` has `datasourceFilter` (for database-mode server filtering), `hasMore`/`loadMore` (lazy loading), `isLoading`
- `SingleSelector` adds: `onEnterEvent`, `onLeaveEvent`, `onChangeFilterInputEvent`, `lazyLoading`, `loadingType`
- `MultiSelector` adds same events plus `selectedItems` array, `selectAllState`, `selectAll()`, `selectableItems`
- `CaptionsProvider.render()` returns ReactNode — supports custom content widgets
- `FilterType` enum: `"contains" | "containsExact" | "startsWith" | "none"`

**3. Behavioral documentation:** The selector abstraction is richer than checkbox-radio-selection — it handles lazy loading state, events, and multi-select state (selectAll, selectedItems) in the interface. Components program against these interfaces without knowing the data source.

**4. User-facing?** No — TypeScript types.

**5. New learnings:** `selectAllState` is a tri-state (`"checked" | "unchecked" | "indeterminate"`) computed by the selector — needed for the SelectAll checkbox UI. `selectableItems` in `MultiSelector` is the list of option IDs that can still be selected (i.e., not yet selected), used to populate the dropdown.

---

## src/helpers/getSelector.ts

**1. Purpose:** Factory function — same routing pattern as checkbox-radio-selection, but creates combobox-specific selector classes.

**2. Logic:**
```
source === "context":
  ["enumeration", "boolean"] → EnumBoolSingleSelector
  "association" + Reference  → AssociationSingleSelector
  "association" + ReferenceSet → AssociationMultiSelector

source === "database":
  Selection.Multi → DatabaseMultiSelectionSelector
  else           → DatabaseSingleSelectionSelector

source === "static" → StaticSingleSelector
```

**3. Behavioral documentation:** Identical routing logic to checkbox-radio-selection. Selector is created once and cached.

**4. User-facing?** No — factory.

**5. New learnings:** Same 6-selector architecture as checkbox-radio-selection-web, but the combobox selector classes are separate implementations with search/filter/lazy-load capabilities.

---

## src/components/SingleSelection/SingleSelection.tsx

**1. Purpose:** Single-select combobox UI component using Downshift.

**2. Logic:**
- Uses `useDownshiftSingleSelectProps` for Downshift integration
- Input shows current selection's caption when closed; shows search term while typing
- `readOnlyStyle === "text"`: renders as plain text (no input, no button)
- `readOnlyStyle === "bordered"`: renders disabled input with no clear button
- Clear button shown when `selector.clearable && !isReadOnly && currentId != null`
- Backspace on empty input clears selection if clearable
- `ComboboxWrapper` wraps the input + toggle button + validation
- `SingleSelectionMenu` shows the dropdown with options
- Lazy loading: `useLazyLoading` provides `onScroll` for the menu; spinner/skeleton when loading

**3. Behavioral documentation:** Standard dropdown combobox. In text mode, shows just the selected value as static text. In bordered mode, shows a disabled input. In editable mode, typing filters the option list (via `setSearchTerm`). The toggle button opens/closes the dropdown. Validation error shown in `ComboboxWrapper`.

**4. User-facing?** Yes — rendered UI.

**5. New learnings:** The component doesn't directly filter options — it calls `selector.options.setSearchTerm(inputValue)` and the selector's OptionsProvider handles both client-side filtering (enum/static) and server-side (database with `filterType`). Downshift handles keyboard navigation (ArrowUp/Down, Enter, Escape).

---

## src/components/SingleSelection/SingleSelectionMenu.tsx

**1. Purpose:** Dropdown menu for single-select mode.

**2. Logic:**
- Positioned via `ComboboxMenuWrapper` which uses `useMenuStyle()` for viewport collision detection
- Shows `<Loader>` (spinner or skeleton) when `selector.options.isLoading`
- Shows `noOptionsText` when no options available
- Each option calls `selector.caption.render(optionId)` — supports custom widget content
- `alwaysOpen` mode (for Studio Pro preview): disables Downshift highlighting, shows menu statically
- Lazy loading: attaches `onScroll` handler to the list

**3. Behavioral documentation:** Renders the dropdown overlay. The menu uses viewport detection to flip position (above vs below the input) when near screen edges. Custom content widgets are rendered per option when configured.

**4. User-facing?** Yes — dropdown UI.

**5. New learnings:** `alwaysOpen` is specifically for Studio Pro canvas preview — it forces the menu open and disables highlighting so the preview looks static. The menu's position is computed by `useMenuStyle` which likely uses Floating UI or a similar positioning library.

---

## src/components/MultiSelection/MultiSelection.tsx

**1. Purpose:** Multi-select combobox UI component with two display modes.

**2. Logic:**
- Uses `useDownshiftMultiSelectProps` for Downshift multi-select integration
- `selectedItemsStyle === "text"`: selected items shown as comma-separated text below input (no individual remove)
- `selectedItemsStyle === "boxes"`: selected items shown as removable tag chips (each has X button)
- Input triggers filtering via `setSearchTerm`
- Backspace/ArrowLeft on empty input: moves focus to last selected tag (for keyboard removal)
- Space key: toggles selection of highlighted option (keyboard selection without closing menu)
- Clear all button removes all selected items (shown when clearable && !readonly && any selected)
- `SelectAllButton` passed as `menuHeaderContent` to menu when `selectAllButton=true`
- Each tag's X button calls `selector.setValue(currentIds.filter(id => id !== item))`

**3. Behavioral documentation:** Rich multi-select with keyboard-first UX. The "boxes" mode shows tags inside the input field. Backspace navigation to tags enables keyboard-only removal. SelectAll header controls all selections at once. The menu stays open while selecting multiple items.

**4. User-facing?** Yes — rendered UI.

**5. New learnings:** `selectedItemsStyle === "text"` is for read-only display of multi selections as text (cannot be edited in this mode presumably). The menu uses `selectionMethod` to determine whether individual checkboxes or row-clicks toggle items.

---

## src/components/MultiSelection/MultiSelectionMenu.tsx

**1. Purpose:** Dropdown menu for multi-select mode.

**2. Logic:**
- Each option renders a checkbox (if `selectionMethod === "checkbox"`) or highlighted row (if `rowclick`)
- Shows checkbox checked state based on `selector.currentId?.includes(optionId)`
- `menuHeaderContent` slot (SelectAllButton) shown above options
- `menuFooterContent` slot (custom widget) shown below options
- Lazy loading scroll handler on the list

**3. Behavioral documentation:** Menu stays open after selection (handled by Downshift state reducer). Options show check state for currently selected items. Footer content enables "Create new item" or similar patterns.

**4. User-facing?** Yes — dropdown UI.

**5. New learnings:** The menu renders both a header and footer content slot, making it more extensible than typical dropdowns. `selectionMethod` switching between checkbox and rowclick is a UX-only difference — both call the same toggle function.

---

## src/hooks/useDownshiftSingleSelectProps.ts

**1. Purpose:** Downshift `useCombobox` configuration for single-select behavior.

**2. Logic:**
- State reducer handles: clear input on toggle/selection/escape, prevent auto-fill on item selection, preserve open state on blur (for alwaysOpen), maintain highlight on focus
- `onInputValueChange`: calls `setSearchTerm(inputValue)` when text changes
- `onIsOpenChange`: calls `onAfterSearchTermChange` callback when menu closes (to reset search)
- `onSelectedItemChange`: calls `selector.setValue()` on selection
- Item string conversion via `selector.caption.get(item)` for display

**3. Behavioral documentation:** Downshift manages keyboard navigation (arrows, enter, escape) and ARIA attributes automatically. The state reducer customizes behavior: input is cleared after selection (not populated with selected value), preventing the common "filled input after select" UX issue.

**4. User-facing?** No — hook.

**5. New learnings:** The input displays the search term, not the selected item's caption — the selected caption is shown in the `ComboboxWrapper` trigger button area. This is different from a typical combobox where the input shows the selected value.

---

## src/hooks/useDownshiftMultiSelectProps.ts

**1. Purpose:** Downshift `useMultipleSelection` + `useCombobox` combined configuration for multi-select.

**2. Logic:**
- `useMultipleSelection` manages the selected items array with `getDropdownProps` for keyboard interaction
- Custom `toggleSelectedItem(item)`: adds to or removes from `selector.currentId` array, then calls `selector.setValue()`
- State reducer: prevents menu closing on item selection (key: keeps menu open for multi-picking)
- Backspace/Delete in empty input → triggers `activeIndex` management for tag removal focus
- `onSelectedItemChange` calls `toggleSelectedItem` to sync with selector

**3. Behavioral documentation:** The menu never closes when selecting items — users must explicitly close it (Escape, click outside, Tab). Backspace on empty input moves focus to the last selected tag. The internal `activeIndex` tracks which tag is "focused" for keyboard removal.

**4. User-facing?** No — hook.

**5. New learnings:** The key behavioral difference from single-select: `stateChangeTypes.ItemClick` and `stateChangeTypes.InputKeyDownEnter` both prevent menu closure in the state reducer. This enables rapid multi-selection without reopening the dropdown.

---

## src/hooks/useLazyLoading.ts

**1. Purpose:** Manages incremental datasource loading as the dropdown list scrolls.

**2. Logic:**
- Wraps `@mendix/widget-plugin-grid`'s `useInfiniteControl`
- Tracks `firstLoad` flag: on first menu open, calls `loadMore()` if not all items loaded
- Resets on `datasourceFilter` or `readOnly` changes
- Returns `onScroll` handler that triggers `loadMore()` when scrolled near the bottom
- `loadingType` determines whether spinner or skeleton is shown while loading

**3. Behavioral documentation:** On first open, loads initial batch. As user scrolls, loads more items. Reset on filter change ensures fresh results. Works with both association and database data sources.

**4. User-facing?** No — hook.

**5. New learnings:** `useInfiniteControl` from `widget-plugin-grid` handles the scroll-position calculation and limit management. The combobox delegates pagination mechanics to this shared utility.

---

## src/hooks/useGetSelector.ts

**1. Purpose:** Creates and caches the selector, adding debounced filter input event.

**2. Logic:**
- Creates selector once via `getSelector(props)` and caches in `useRef`
- Calls `selector.updateProps(props)` every render
- Debounces `onChangeFilterInputEvent` by `filterInputDebounceInterval` (default 200ms) using a custom debounce wrapper
- `onAfterSearchTermChange` callback triggers forced re-render via `setInput({})`

**3. Behavioral documentation:** Same caching pattern as checkbox-radio-selection. The debounce prevents firing `onChangeFilterInputEvent` on every keystroke — only after the user pauses.

**4. User-facing?** No — hook.

**5. New learnings:** The debounce for `onChangeFilterInputEvent` is configurable via `filterInputDebounceInterval` property — this enables server-side search with network latency considerations (don't fire on every key, wait for typing pause).

---

## src/hooks/useActionEvents.ts

**1. Purpose:** Provides `onFocus`/`onBlur` handlers for enter/leave events.

**2. Logic:**
- `onFocus`: fires `onEnterEvent` via `executeAction` and `selector.onEnterEvent?.()`
- `onBlur`: checks if `relatedTarget` is outside `currentTarget` subtree before firing `onLeaveEvent` — prevents spurious leave when focus moves between child elements within the widget

**3. Behavioral documentation:** The focus-leave guard ensures `onLeaveEvent` fires only when the user truly leaves the widget (not when moving between the input and dropdown). Essential for form validation patterns where leave triggers validation.

**4. User-facing?** No — hook.

**5. New learnings:** The `currentTarget.contains(relatedTarget)` check is the standard pattern for detecting "true blur" on a composite widget. Without this, a click on a dropdown option would fire leave + enter in rapid succession.

---

## package.json

**1. Purpose:** Package manifest.

**2. Logic:** v2.8.0, `downshift ^7.6.2`, `match-sorter ^8.1.0`, min Mendix 10.22.0. Dependencies on `widget-plugin-grid` (infinite scroll), `widget-plugin-hooks` (debounce), `widget-plugin-component-kit` (Alert, If, etc.).

**3. Behavioral documentation:** `match-sorter` handles client-side fuzzy matching for enum/static/association options. `downshift` provides accessible combobox keyboard/ARIA behavior.

**4. User-facing?** No — manifest.

**5. New learnings:** Min Mendix 10.22.0 is very recent (more restrictive than most other widgets). `match-sorter ^8.1.0` — the major version jump from 6.x suggests significant API changes were adopted. `widget-plugin-grid` is shared with data grid widgets for infinite scroll functionality.

---

## CHANGELOG.md

**1. Purpose:** Release history.

**2. Logic:** Key milestones:
- **v2.8.0**: Fixed "On selection" event for database mode
- **v2.x**: Various fixes for focus, filtering, accessibility, keyboard behavior
- **v1.6.0**: Lazy loading with spinner/skeleton loaders added
- **v1.5.0**: Read-only style option added
- **v1.3.0**: Static values support added
- **v1.2.0**: Database list feature added
- **v1.1.0**: SelectAll button, footer content added
- **v1.0.0**: Initial release

**3. Behavioral documentation:** The widget went from basic association/enum combobox to a full-featured multi-source, multi-mode selection widget over its history. Database mode had a bug in "On selection" event as late as v2.8.0 (fixed in latest).

**4. User-facing?** No — developer documentation.

**5. New learnings:** The "On selection" event for database mode was broken until v2.8.0 — important for users upgrading from earlier versions. The widget's versioning (2.x) despite being relatively feature-complete suggests active development cadence.

---

## Helper Files Summary

**LazyLoadProvider.ts**: Manages datasource `setLimit()` calls for pagination. Returns `undefined` limit (unlimited) when lazy loading is disabled; increments limit on scroll; returns 0 when readonly to avoid loading. Integrates with `useInfiniteControl` from `widget-plugin-grid`.

**datasourceFilter.ts**: Creates Mendix `FilterCondition` objects for server-side database filtering. Supports `contains` (substring), `containsExact` (exact), `startsWith` prefix, and `none` (no filter). Uses Mendix filter builders — type-safe filter construction. Returns `undefined` for empty search or `filterType === "none"`.

**SelectAllButton.tsx**: Tri-state checkbox (`checked`/`unchecked`/`indeterminate`) in the menu header. Disabled when no options. Clicking toggles all items. The label's click is prevented from double-firing.

**ComboboxWrapper.tsx**: Input container with conditional CSS classes (`active`, `disabled`, `form-control-static`, `multiselect`). Shows dropdown arrow icon or spinner when lazy-loading. Renders `ValidationAlert` for validation errors.

**ComboboxMenuWrapper.tsx**: Menu container with `useMenuStyle()` for viewport-aware positioning (flip above/below). Header/footer slots prevent mousedown events from closing the menu. Attaches lazy scroll handler to the `<ul>`. Shows `NoOptionsPlaceholder` when empty.

**e2e/Combobox.spec.js**: 11 Playwright tests covering all data source types (association, association-rowclick, enum, boolean, static, database), readonly mode, footer content, filtering, removing single/all selections, and keyboard shortcuts (Backspace clear, filter typing). Uses screenshot-based visual assertions.

**typings/ComboboxProps.d.ts**: Auto-generated types — `SelectionMethodEnum` (checkbox/rowclick), `SelectedItemsStyleEnum` (text/boxes), `FilterTypeEnum` (contains/containsExact/startsWith/none), `LoadingTypeEnum` (spinner/skeleton). `ComboboxContainerProps` has ~105 properties.

---

## Summary

**Widget:** combobox-web v2.8.0 — searchable dropdown selection widget.

**Purpose:** Provides a combobox (searchable dropdown) for single and multi-select scenarios backed by the same 3 data sources as checkbox-radio-selection (context association/enum/boolean, database, static).

**Architecture:**

```
Combobox (container)
  └─ useGetSelector → getSelector() → 6 selector classes (same as checkbox-radio-selection)
  └─ SingleSelection (enum/association-Reference/database-Single/static)
  │    └─ useDownshiftSingleSelectProps (Downshift combobox)
  │    └─ SingleSelectionMenu (dropdown)
  └─ MultiSelection (association-ReferenceSet, database-Multi)
       └─ useDownshiftMultiSelectProps (Downshift multi)
       └─ MultiSelectionMenu (dropdown with checkboxes/rowclick)
```

**Key behaviors vs checkbox-radio-selection:**

| Feature | checkbox-radio-selection | combobox |
|---------|--------------------------|---------|
| UI style | inline radio/checkbox group | dropdown overlay |
| Search/filter | No | Yes (contains/startsWith/none) |
| Lazy loading | No | Yes (scroll-based pagination) |
| Multi display | checkboxes | text or tag boxes |
| SelectAll | No | Yes (optional) |
| Footer content | No | Yes (custom widget slot) |
| Events | onChange only | onChange + onEnter + onLeave + onFilterInput |
| Min Mendix | 10.7.0 | 10.22.0 |

**Key behaviors:**
- Filter input debounced (configurable ms) before firing `onChangeFilterInputEvent`
- Database source: `filterType` drives server-side `FilterCondition` creation via Mendix filter builders
- Enum/static/association: client-side filtering via `match-sorter`
- Multi-select menu stays open after each selection
- Read-only "text" style: show only selected value(s) as plain text
- Viewport-aware menu positioning (flips above/below input)
- 11 Playwright e2e tests, limited unit tests
