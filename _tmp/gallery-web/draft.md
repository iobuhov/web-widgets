# gallery-web — Spec Draft

Widget package: `packages/pluggableWidgets/gallery-web`
Source: `iobuhov/web-widgets`

---

## src/Gallery.xml

**1. What is the purpose of this file?**
The Mendix widget descriptor that declares every configurable property of the Gallery widget. It is the authoritative source for property names, types, defaults, captions, and grouping. Studio Pro reads this file to render the property panel and generate `GalleryProps.d.ts`.

**2. What kind of logic is described in this file?**
Declarative property schema: data source (`datasource`, `content`, `filtersPlaceholder`), responsive column counts per breakpoint (`desktopItems`, `tabletItems`, `phoneItems`, each 1–12), click trigger enum (single/double), click action, selection (None/Single/Multi), selection behavior flags (`autoSelect`, `itemSelectionMode` toggle/clear, `keepSelection`), selection counter position (top/bottom/off), loading type (spinner/skeleton), refresh interval and indicator, three pagination modes (paging buttons/virtual scrolling/load more), custom pagination drop zone, paging position (top/bottom/both), dynamic page-control attributes, empty placeholder mode, dynamic item CSS class expression, personalization storage (attribute vs. browser local storage), and accessibility text templates.

**3. What part of behavior can be documented from this file?**
- `itemSelection` supports None, Single, Multi; Multi unlocks `keepSelection`, `selectionCountPosition`, `clearSelectionButtonLabel`.
- `itemSelectionMode` (toggle/clear) defines whether clicking a selected item deselects it.
- `autoSelect` (default false) automatically selects the first item when enabled.
- `onClickTrigger` (single/double) controls which mouse event fires `onClick`.
- `pagination` chooses between paging buttons, virtual scrolling, or "Load More" button.
- `dynamicPageSize` and `dynamicPage` are Integer attributes enabling programmatic page control.
- `totalCountValue` and `dynamicItemCount` are write-back Integer attributes.
- `stateStorageType` attribute stores personalization JSON in a Mendix Unlimited String attribute; localStorage stores it in browser local storage (not tied to a Mendix user).
- `refreshInterval` (integer, seconds, default 0 = disabled) sets an automatic datasource refresh interval.
- The widget is declared `offlineCapable="true"`.

**4. Is it user-facing?**
Indirectly: this file drives the Studio Pro configuration UI. End-users do not see the XML, but every behavior option the developer can configure originates here.

**5. What new did you learn from this file?**
The widget supports a custom pagination drop zone (`customPagination`) as an alternative to the built-in paging controls. The `dynamicItemCount` attribute reflects the number of currently-loaded items (read-only attribute pattern). `filterSectionTitle` and `emptyMessageTitle` are aria-label text templates for screen-reader navigation landmarks.

---

## typings/GalleryProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript types from `Gallery.xml`. Provides strongly-typed prop interfaces for `GalleryContainerProps` (runtime) and `GalleryPreviewProps` (Studio Pro preview). Not hand-edited; regenerated on widget build.

**2. What kind of logic is described in this file?**
Type definitions only: enum string unions, container props with Mendix SDK types (`ListValue`, `ListActionValue`, `ListExpressionValue`, `ListWidgetValue`, `SelectionSingleValue`, `SelectionMultiValue`, `EditableValue`, `DynamicValue`), and preview props with simplified types for the design-mode renderer.

**3. What part of behavior can be documented from this file?**
- `itemSelection` is typed `SelectionSingleValue | SelectionMultiValue | undefined` — its absence means selection is disabled.
- `dynamicPageSize`, `dynamicPage`, `totalCountValue`, `dynamicItemCount` are `EditableValue<Big>` (Mendix Integer stored as `big.js`).
- `itemClass` is `ListExpressionValue<string>` — evaluated per-item, enabling per-row dynamic CSS classes.
- `ariaLabelItem` is `ListExpressionValue<string>` — per-item aria-label evaluated from a data expression.
- `onClick` is `ListActionValue` — a per-item action.
- `onSelectionChange` is `ActionValue` (not per-item) — fires once when selection changes.

**4. Is it user-facing?**
No — internal TypeScript contract. Generated file; no manual changes.

**5. What new did you learn from this file?**
`GalleryPreviewProps.itemSelection` is typed as the string union `"None" | "Single" | "Multi"` in preview mode (simplified from the SDK value types), confirming that Studio Pro passes a plain string for the selection prop in design-mode rendering.

---

## src/Gallery.tsx

**1. What is the purpose of this file?**
The Mendix plugin entry point. Exports the `Gallery` function that Mendix will call with widget props on each React render cycle.

**2. What kind of logic is described in this file?**
It delegates all logic to `useGalleryContainer(props)` which builds a brandi IoC container, then wraps `<GalleryWidget>` in `<ContainerProvider container={container} isolated>`. The `isolated` flag means each widget instance has its own DI container, preventing cross-instance state leakage.

**3. What part of behavior can be documented from this file?**
The widget uses brandi (a TypeScript IoC container library) combined with MobX for reactive state. Every prop update is fed through the container rather than being passed directly as React props — enabling fine-grained MobX reactivity without prop drilling.

**4. Is it user-facing?**
No — this is the framework integration point. End-users interact with the rendered output.

**5. What new did you learn from this file?**
The `isolated` prop on `ContainerProvider` is critical: it ensures that each Gallery widget instance on a page has fully independent state, which matters for pages with multiple Gallery instances.

---

## src/Gallery.editorConfig.ts

**1. What is the purpose of this file?**
Controls Studio Pro property panel visibility and provides design-time validation errors and the structure-mode preview.

**2. What kind of logic is described in this file?**
`getProperties`: hides properties based on current values (conditional visibility). `check`: returns validation errors. `getPreview`: renders the structure-mode preview. `getCustomCaption`: shows the datasource caption in the widget tree.

**3. What part of behavior can be documented from this file?**
Property visibility constraints (behavioral rules enforceable at design time):
- `emptyPlaceholder` slot hidden when `showEmptyPlaceholder !== "custom"`.
- `onSelectionChange`, `itemSelectionMode`, `autoSelect` hidden when `itemSelection === "None"`.
- `keepSelection`, `selectionCountPosition`, `clearSelectionButtonLabel` hidden when `itemSelection !== "Multi"`.
- Personalization properties hidden when neither `storeFilters` nor `storeSort` is enabled. `stateStorageAttr` and `onConfigurationChange` hidden when storage type is `localStorage`.
- For "buttons" pagination: `showTotalCount` and `dynamicItemCount` hidden. For other modes: `showPagingButtons` and custom pagination properties are hidden. `loadMoreButtonCaption` only shown for `loadMore` mode.
- **Validation error**: if `itemSelection !== "None"` AND `onClick` is set AND `onClickTrigger === "single"`, Studio Pro reports an error: "The item click action is ambiguous." Developer must use double-click trigger or disable selection to resolve.
- Column count (desktop/tablet/phone) must be between 1 and 12 inclusive; Studio Pro enforces this.

**4. Is it user-facing?**
No — affects the developer/designer experience in Studio Pro only.

**5. What new did you learn from this file?**
The single-click + selection + onClick combination is explicitly blocked with a Studio Pro error, not merely a runtime warning. This is a hard design-time constraint: selection with onClick requires double-click trigger.

---

## src/Gallery.editorPreview.tsx

**1. What is the purpose of this file?**
Implements the React component shown inside Studio Pro's design/xray/structure preview mode. Also provides `useProvideSortAPI` with a stub `SortStoreHost` for the preview environment.

**2. What kind of logic is described in this file?**
Renders a simplified gallery layout using preview props: top bar (selection counter if Multi + top, pagination if top), header (filters placeholder), content (item grid using desktop/tablet/phone column counts), footer (Load More button or pagination if bottom, selection counter if Multi + bottom). Custom pagination is rendered via a drop zone placeholder.

**3. What part of behavior can be documented from this file?**
- Selection counter is shown only when `itemSelection === "Multi"` — confirmed that Single selection has no counter UI.
- `selectionCountPosition` controls whether counter appears at `top` or `bottom`.
- `useCustomPagination` combined with `pagingPosition` determines where custom pagination renders (top, bottom, or both).
- Pagination buttons visible when `showTotalCount || pagination === "buttons"` AND `!useCustomPagination`.
- Preview renders 3 rows × column count items for each breakpoint using CSS classes `visible-md visible-lg`, `visible-sm`, `visible-xs`.

**4. Is it user-facing?**
No — Studio Pro design canvas only.

**5. What new did you learn from this file?**
The preview content area uses a comment `"selectable_DO_NOT_REMOVE!_ALWAYS_RENDER!"` on the first item — it must always be rendered for drag-and-drop in Studio Pro to work correctly. This is a Studio Pro design tooling constraint, not runtime behavior.

---

## src/components/GalleryWidget.tsx

**1. What is the purpose of this file?**
Assembles the full widget component tree by composing all sub-components.

**2. What kind of logic is described in this file?**
Pure composition: Root → [TopBar(TopBarControls) → Header → RefreshStatus → Content(Items) → EmptyPlaceholder → Footer(FooterControls)].

**3. What part of behavior can be documented from this file?**
The rendering order defines the visual DOM structure: top bar controls (including pagination or counter at top) appear before the filter header, then the refresh indicator, then the items grid, then the empty placeholder (shown when items = 0), then the footer controls (pagination/counter at bottom).

**4. Is it user-facing?**
Yes — this is the runtime component. Its output is what end-users see.

**5. What new did you learn from this file?**
`EmptyPlaceholder` is rendered after `Content` (items) — the empty state is sibling to, not a replacement of, the items container. The view model controls visibility of each section via MobX computed properties.

---

## src/components/GalleryRoot.tsx

**1. What is the purpose of this file?**
Renders the outermost `<div class="widget-gallery">` element with className, style, and data-focusindex attributes.

**2. What kind of logic is described in this file?**
MobX observer component. Consumes `GalleryRootViewModel` via hook. Applies: `className` (user-configured class + "widget-gallery"), `style` (inline CSS from props), `data-focusindex` = tabIndex (defaults to 0).

**3. What part of behavior can be documented from this file?**
- `data-focusindex` on the root div is a Mendix-specific attribute used by the platform's focus management system. Default tabIndex is 0.
- The root div always has the `widget-gallery` class, plus any additional classes from the `class` prop.

**4. Is it user-facing?**
Yes — the root HTML element of the widget. The `data-focusindex` attribute participates in Mendix focus management.

**5. What new did you learn from this file?**
`data-focusindex` (not `tabIndex`) is used on the root — this is a Mendix platform convention for focus tracking, distinct from native tab order.

---

## src/components/GalleryItems.tsx

**1. What is the purpose of this file?**
Renders the responsive grid of gallery items wrapped in keyboard navigation context.

**2. What kind of logic is described in this file?**
MobX observer. Reads items from `useItems()`, config (column counts), texts (aria-label for listbox), and `focusController`. Returns null when items count < 1. Renders `<ListBox>` with `<KeyNavProvider>` wrapping individual `<ListItem>` elements.

**3. What part of behavior can be documented from this file?**
- When the datasource is empty, `GalleryItems` renders nothing (null), allowing `EmptyPlaceholder` to take over.
- The `aria-label` on the listbox container comes from `texts.listboxAriaLabel` (configured via `ariaLabelListBox` prop).
- `KeyNavProvider` enables 2D keyboard navigation across the grid; it receives a `focusController` that knows the grid dimensions.

**4. Is it user-facing?**
Yes — renders the visible item grid.

**5. What new did you learn from this file?**
The keyboard navigation is grid-aware (2D): `KeyNavProvider` uses the column layout to allow up/down/left/right arrow movement across rows and columns, not just linear tab order.

---

## src/components/ListBox.tsx

**1. What is the purpose of this file?**
Renders the items container div with ARIA listbox/list semantics and responsive CSS column classes.

**2. What kind of logic is described in this file?**
Stateless functional component. Applies CSS classes: `widget-gallery-items`, `widget-gallery-lg-{n}`, `widget-gallery-md-{n}`, `widget-gallery-sm-{n}`. Sets `role="listbox"` and `aria-multiselectable` when selection is enabled, `role="list"` otherwise.

**3. What part of behavior can be documented from this file?**
- `role="listbox"` + `aria-multiselectable={true}` for Multi selection — conforms to WAI-ARIA listbox pattern.
- `role="listbox"` + `aria-multiselectable={false}` for Single selection.
- `role="list"` for no-selection mode — a simpler semantic.
- CSS classes `widget-gallery-lg-N`, `widget-gallery-md-N`, `widget-gallery-sm-N` drive the responsive grid layout via CSS.

**4. Is it user-facing?**
Yes — the container element for all gallery items, with ARIA semantics visible to screen readers.

**5. What new did you learn from this file?**
The responsive column system is CSS-driven through class names (`lg`, `md`, `sm` suffixes with column count), not inline grid styles. This means the breakpoint CSS must define `.widget-gallery-lg-N`, `.widget-gallery-md-N`, `.widget-gallery-sm-N` grid rules.

---

## src/components/ListItem.tsx

**1. What is the purpose of this file?**
Renders a single gallery item as an interactive div with selection state, keyboard navigation, ARIA props, and optional click wrapper.

**2. What kind of logic is described in this file?**
MobX observer. Computes `isSelected` from `selectActions.isSelected(item)`. Determines `clickable` = hasOnClick OR selection active. Gets aria props from `getAriaProps`. Gets grid position from `getPositionFn(itemIndex)` for keyboard navigation. Gets event handlers from `eventsVM.getProps(item)`. If clickable via onClick, wraps content in `<ListItemButton>`.

**3. What part of behavior can be documented from this file?**
- `widget-gallery-clickable` class added when item has onClick or selection is enabled.
- `widget-gallery-selected` class added when item is selected.
- `data-selected={true}` attribute set when selected (undefined when not, so attribute is absent from DOM).
- `itemClass` expression result is applied as an additional CSS class per item.
- Items with `onClick` configured wrap their content in `<ListItemButton>` (role="button"), enabling keyboard activation. Items without onClick render content directly.

**4. Is it user-facing?**
Yes — the core visual and interactive element of the gallery.

**5. What new did you learn from this file?**
The `ListItemButton` wrapper is only applied when `hasOnClick` is true, not when only selection is active. Selection interaction is handled at the ListItem level through event handlers, while click actions require the button wrapper for proper keyboard semantics.

---

## src/components/ListItemButton.tsx

**1. What is the purpose of this file?**
A div with `role="button"` that translates Enter/Space key presses into click events, enabling keyboard activation of gallery items with onClick actions.

**2. What kind of logic is described in this file?**
Renders a `<div role="button" class="widget-gallery-item-button">`. Maintains a `pressed` boolean flag (singleton per module). On `keydown` Enter/Space: sets pressed=true. On `keyup` Enter/Space (if same target and pressed): dispatches a synthetic `MouseEvent("click")` bubbling up, then resets pressed.

**3. What part of behavior can be documented from this file?**
- Only Enter and Space keys trigger activation.
- The `isOwn` check ensures the keydown must have originated on this exact element (not bubbled from a child), preventing accidental activation from nested interactive elements.
- The synthetic click event bubbles, so parent click handlers receive it.
- `preventAndStop` is called on both keydown and keyup to prevent default browser behavior (e.g., scrolling on Space).

**4. Is it user-facing?**
Yes — the interactive wrapper for clickable gallery items that enables keyboard access.

**5. What new did you learn from this file?**
The `kbdHandlers` object is a module-level singleton (created once), not per-instance. This means the `pressed` state is shared across all instances of `ListItemButton` on the page. Practically this is fine for sequential keyboard use but is a subtle implementation detail.

---

## src/features/item-interaction/action-handlers.ts

**1. What is the purpose of this file?**
Creates event handler entries for the gallery item's click/keyboard action execution.

**2. What kind of logic is described in this file?**
Exports `createActionHandlers(execActionFx)` which returns an array of event entries for the event-switch system: `onClick` (single click, no meta/ctrl modifier), `onDoubleClick` (double click, no meta/ctrl), `onOwnSpaceKeyDown` (prevents default scroll), and `onSpaceOrEnter` pair (keydown tracks press, keyup executes if same key and pressed). Space without shift triggers action; Enter always triggers action.

**3. What part of behavior can be documented from this file?**
- **Modifier key exclusion**: clicks with Meta or Ctrl held do NOT fire the action — these modifiers are reserved for range/multi-selection operations.
- **Single vs Double click**: controlled by `clickTrigger` context value passed by the event-switch filter.
- **Space key**: triggers action on keyup (after keydown on same target), but NOT when Shift is held (shift+space is reserved for selection).
- **Enter key**: always triggers action on keyup (no shift restriction).
- `onOwnSpaceKeyDown` fires `preventDefault` to suppress browser scroll behavior when space is pressed on a gallery item.

**4. Is it user-facing?**
Yes — defines the keyboard and mouse interaction behavior for all gallery items with an onClick action.

**5. What new did you learn from this file?**
Space+Shift does NOT trigger the onClick action — it is silently filtered out because the `canExecOnSpaceOrEnter` function returns false when `event.shiftKey` is true for Space. This is intentional: Shift+Space is used for selection range extension.

---

## src/features/item-interaction/keyboard-utils.ts

**1. What is the purpose of this file?**
Provides a filter utility that prevents gallery keyboard handlers from intercepting events originating from input/textarea elements inside gallery item content.

**2. What kind of logic is described in this file?**
`withInputEventsFilter` wraps multiple `onKeyDown` event entries into one. If the event target is `HTMLInputElement` or `HTMLTextAreaElement`, all entries are skipped. Otherwise, each entry's filter and handler are evaluated normally.

**3. What part of behavior can be documented from this file?**
Text input fields and textareas placed inside gallery item content slots will not trigger gallery keyboard navigation or selection shortcuts. This is a critical behavioral constraint: arrow keys in an input field inside a gallery item navigate the cursor, not the gallery grid.

**4. Is it user-facing?**
Yes — preserves correct behavior when content widgets include text inputs.

**5. What new did you learn from this file?**
The input filter applies specifically to `onKeyDown` events, not `onClick` or `onKeyUp`. So keyboard navigation (arrows, shift+space, etc.) is blocked from inputs, but click-based actions still work normally.

---

## src/features/item-interaction/get-item-aria-props.ts

**1. What is the purpose of this file?**
Computes the ARIA props for each gallery item based on selection type and selection state.

**2. What kind of logic is described in this file?**
`getAriaProps(selectionType, isSelected, label?)`: returns `role="option"`, `aria-selected`, `tabIndex=0`, `aria-label` for Single/Multi selection; returns `role="listitem"`, no `aria-selected`, no `tabIndex`, `aria-label` for no-selection mode.

**3. What part of behavior can be documented from this file?**
- Selectable items (Single or Multi) have `role="option"` and `tabIndex=0` — they are keyboard focusable and participate in the listbox pattern.
- Non-selectable items have `role="listitem"` and no tabIndex — they are not keyboard-focusable individually (no native focus).
- `aria-label` is set on every item if `ariaLabelItem` expression is configured, regardless of selection type.

**4. Is it user-facing?**
Yes — ARIA attributes directly affect screen reader experience.

**5. What new did you learn from this file?**
Non-selectable items have no tabIndex, meaning they are not in the tab order. Keyboard navigation for non-selectable items is handled by other mechanisms (the KeyNavProvider), not by native tab focus.

---

## src/features/settings-storage/GallerySettingsSync.service.ts

**1. What is the purpose of this file?**
Synchronizes filter and sort state between MobX stores and the configured storage backend (attribute or browser), enabling personalization persistence.

**2. What kind of logic is described in this file?**
MobX reactions: one reaction writes the current state to storage (debounced 250ms, structural equality check); another reaction reads from storage (fires immediately on setup) and calls `fromJSON`. Validates schema version (must be 1) before applying; logs a warning and resets storage if invalid. `toJSON`/`fromJSON` serialize filter and sort state conditional on `storeFilters`/`storeSort` config flags.

**3. What part of behavior can be documented from this file?**
- Writes are debounced 250ms — rapid filter/sort changes don't flood storage.
- Invalid or version-mismatched stored settings are discarded and storage is reset to null.
- Filter and sort storage are independently controlled by `storeFilters` and `storeSort` flags.
- Schema version is 1; a future version change would invalidate existing stored settings.

**4. Is it user-facing?**
Yes — the persistence behavior is directly experienced by users (personalized filters/sort survive page reload).

**5. What new did you learn from this file?**
`fireImmediately: true` on the read reaction means stored settings are applied as soon as the service sets up — on first render, the gallery immediately restores the user's last filter/sort state before any user interaction.

---

## src/features/settings-storage/AttributeStorage.ts

**1. What is the purpose of this file?**
Implements `ObservableStorage` using a Mendix Unlimited String attribute as the backend.

**2. What kind of logic is described in this file?**
Reads: parses the attribute's string value as JSON; returns null if empty or on JSON parse error (resets the attribute to empty string on error). Writes: serializes data as JSON string and sets via `EditableValue.setValue()`. The `data` getter is a MobX `computed.struct`.

**3. What part of behavior can be documented from this file?**
- Personalization stored in a Mendix attribute requires an Unlimited String attribute (confirmed by XML description).
- Malformed JSON in the attribute is silently reset to empty string — no user-visible error.
- The attribute must be an `EditableValue` (writable), not read-only.
- This storage is tied to the Mendix user session/entity — if the attribute is on a user object, settings are per-user.

**4. Is it user-facing?**
Indirectly — the stored JSON is not shown to users, but its presence/absence affects their personalization experience.

**5. What new did you learn from this file?**
The `setData` method treats an empty string `""` as `null` (resets the attribute to empty). This normalizes the storage so that clearing settings removes the attribute value entirely.

---

## src/features/settings-storage/BrowserStorage.ts

**1. What is the purpose of this file?**
Implements `ObservableStorage` using browser `localStorage` as the backend.

**2. What kind of logic is described in this file?**
`data` getter: reads from `localStorage.getItem(key)`, parses JSON, returns null on error. `setData`: calls `localStorage.setItem(key, JSON.stringify(data))`. No MobX observability — the storage key is fixed at construction time.

**3. What part of behavior can be documented from this file?**
- Storage key is derived from the widget configuration (not documented here — set in `create-settings-storage.ts`).
- Browser storage is not tied to a Mendix user — different users on the same browser share the same stored settings.
- This is a behavioral constraint documented in the XML description: "This configuration is not tied to a Mendix user."
- No debouncing at this level — debouncing is handled by the sync service.

**4. Is it user-facing?**
Indirectly — browser-stored personalization persists across sessions for the same browser profile.

**5. What new did you learn from this file?**
`BrowserStorage` does not implement `SetupComponent` (no `setup()` method), unlike `AttributeStorage`. It has no lifecycle management and no MobX reactivity — it is a plain synchronous read/write adapter.

---

## src/model/services/QueryParams.service.ts

**1. What is the purpose of this file?**
Wires the filter condition and sort order from MobX observable stores into the Mendix `QueryService`, which controls the datasource query.

**2. What kind of logic is described in this file?**
Two MobX reactions (`fireImmediately: true`): one updates `query.setSortOrder` on sort changes, one updates `query.setFilter` on filter condition changes. Both fire on setup to apply initial state immediately.

**3. What part of behavior can be documented from this file?**
- Filter changes and sort changes independently trigger datasource re-queries.
- The initial filter and sort state (potentially restored from personalization storage) is applied on first render via `fireImmediately`.
- `QueryService` is the single point of datasource query control — all filtering and sorting goes through it.

**4. Is it user-facing?**
Yes — drives what data is fetched and displayed in response to filter/sort changes.

**5. What new did you learn from this file?**
The filter and sort MobX stores are decoupled from `QueryService` — the service just observes them and syncs. This means filter/sort state can be set from multiple sources (user interaction, personalization restore, external event) and will always be reflected in the query.

---

## src/model/configs/Gallery.config.ts

**1. What is the purpose of this file?**
Builds the static `GalleryConfig` object from container props for use across the widget's service layer.

**2. What kind of logic is described in this file?**
Factory function `galleryConfig(props)`: generates a unique ID (`${name}:Gallery@${uuid}`), maps selection props (selectionEnabled, selectionType, selectionMode, keepSelection, autoSelect), column counts, and a `filtersChannelName` (`${id}:events`). `refreshIntervalMs` is hardcoded to 0 (not read from props here — set elsewhere).

**3. What part of behavior can be documented from this file?**
- Each gallery widget instance has a unique ID based on widget name + UUID — this prevents cross-widget interference.
- `filtersChannelName` enables the event bus pattern: external filter widgets publish events to this channel; the gallery subscribes. The channel name is unique per widget instance.
- `selectionEnabled` = `itemSelection !== undefined`. If the `itemSelection` prop is not connected (undefined), selection is disabled regardless of other settings.

**4. Is it user-facing?**
No — internal configuration object.

**5. What new did you learn from this file?**
The `filtersChannelName` is how the Gallery subscribes to filter events from child filter widgets (placed in the `filtersPlaceholder` slot). The channel is namespaced to the widget instance UUID to prevent cross-gallery contamination on pages with multiple galleries.

---

## src/model/configs/GalleryPagination.config.ts

**1. What is the purpose of this file?**
Computes the complete pagination configuration from widget props, capturing all interdependencies between pagination settings.

**2. What kind of logic is described in this file?**
Factory function `galleryPaginationConfig(props)`. Key computed values: `initPageSize` (0 if dynamicPageSize connected, else constPageSize), `customPaginationEnabled`, `dynamicPageEnabled`, `dynamicPageSizeEnabled`, `isLimitBased` (virtualScrolling or loadMore), `requestTotalCount` (buttons mode OR showTotalCount OR totalCountValue connected), `paginationKind` (`"custom"` if custom, else `"${pagination}.${showPagingButtons}"`).

**3. What part of behavior can be documented from this file?**
- When `dynamicPageSize` attribute is connected, the initial datasource fetch requests 0 items — no data is loaded until the attribute value is read and applied on setup. This prevents a flash of data with the wrong page size.
- `requestTotalCount` is true for buttons pagination (needed for page count display), when `showTotalCount` is on, or when `totalCountValue` is connected (write-back attribute).
- Virtual scrolling and Load More use limit-based pagination (`isLimitBased`), not offset/page-based.
- `paginationKind` is a compound string like `"buttons.always"` or `"virtualScrolling.auto"` used for component selection.

**4. Is it user-facing?**
No — internal configuration driving pagination component selection and datasource behavior.

**5. What new did you learn from this file?**
The `initPageSize=0` when `dynamicPageSize` is connected is a deliberate design to avoid loading data with an incorrect initial limit. The `DynamicPaginationFeature` sets the real limit once the attribute is available, preventing wasted server requests.

---

## src/model/services/Loader.service.ts

**1. What is the purpose of this file?**
Exposes loading state properties used to control the loading indicator and spinner/skeleton rendering.

**2. What kind of logic is described in this file?**
MobX computed properties: `isFirstLoad`, `isFetchingNextBatch`, `isRefreshing`. The `isRefreshing` logic: if `showSilentRefresh` is true, includes both silent and non-silent refreshes; otherwise shows only non-silent (explicit) refreshes. `showRefreshIndicator` returns false if `refreshIndicator` prop is disabled, otherwise returns `isRefreshing`.

**3. What part of behavior can be documented from this file?**
- A "silent refresh" (e.g., triggered by `refreshInterval`) does not show the refresh indicator unless `showSilentRefresh` is enabled.
- An explicit refresh (e.g., user action) always shows the refresh indicator when `refreshIndicator` is true.
- `showRefreshIndicator` is the single computed value consumed by `RefreshStatus` component.

**4. Is it user-facing?**
Indirectly — controls whether the user sees the progress bar during data refresh.

**5. What new did you learn from this file?**
There are two categories of refresh: "silent" (background, e.g., interval) and explicit (foreground). The `showSilentRefresh` config flag determines whether background refreshes show the progress indicator to the user. This prevents the progress bar from flashing on every auto-refresh interval.

---

## src/model/services/SelectionGate.service.ts

**1. What is the purpose of this file?**
Adapts the Gallery's gate props to the `SelectionDynamicProps` interface required by the shared selection helper from `@mendix/widget-plugin-grid`.

**2. What kind of logic is described in this file?**
`SelectionGate extends MappedGate<GalleryGateProps, SelectionDynamicProps>`. The `map` function extracts `itemSelection → selection`, `datasource`, and `onSelectionChange` from gate props.

**3. What part of behavior can be documented from this file?**
- The selection system is implemented in `@mendix/widget-plugin-grid` (shared package), not gallery-specific code.
- `onSelectionChange` action fires when the selection changes — it has access to the full datasource context to determine what is selected.

**4. Is it user-facing?**
No — internal adapter.

**5. What new did you learn from this file?**
The Gallery's selection feature reuses the `widget-plugin-grid` selection implementation. This means selection behavior (single/multi selection logic, range selection, keyboard selection) is shared code across Mendix grid-like widgets.

---

## src/model/services/Layout.service.ts

**1. What is the purpose of this file?**
Provides responsive layout calculations: determines the active breakpoint, number of columns, number of rows, and a grid-position function used for 2D keyboard navigation.

**2. What kind of logic is described in this file?**
MobX observable `width` tracking window resize events. Breakpoints: phone < 768px, tablet 768–991px, desktop ≥ 992px. `numberOfColumns` = min(configured columns for current breakpoint, itemCount) — prevents empty columns. `numberOfRows` = ceil(itemCount / columns). `getPositionFn` returns a function that maps a linear item index to `{columnIndex, rowIndex}`.

**3. What part of behavior can be documented from this file?**
- Phone breakpoint: < 768px. Tablet: 768–991px. Desktop: ≥ 992px. These are fixed in code (not from CSS variables; v3.0.1 fixed a separate issue where custom-variables breakpoints weren't followed in CSS).
- `numberOfColumns` is capped at `itemCount` — a 3-column gallery showing 2 items will use 2 columns, not 3.
- The grid position function enables 2D keyboard navigation (arrow keys navigate by column within a row and by row between rows).

**4. Is it user-facing?**
Yes — determines the visual grid layout and keyboard navigation behavior.

**5. What new did you learn from this file?**
The breakpoints (768px, 992px) are hardcoded in the Layout service. The v3.0.1 fix was for CSS (custom-variables), not for this service. Both must be consistent for correct responsive behavior.

---

## src/components/RefreshStatus.tsx

**1. What is the purpose of this file?**
Renders an HTML `<progress>` element as a visual refresh indicator when data is being loaded.

**2. What kind of logic is described in this file?**
MobX observer. Reads `loaderVM.showRefreshIndicator` and `loaderVM.isRefreshing`. Returns null if either is false. Renders: outer `<div>` → `<div class="mx-refresh-container">` → `<progress class="mx-refresh-indicator" />`.

**3. What part of behavior can be documented from this file?**
- Uses a native HTML `<progress>` element (indeterminate, no value attribute) for the loading animation.
- CSS classes follow Mendix conventions (`mx-` prefix), styled by the Mendix theme.
- The component is rendered between the Header (filters) and the Content (items) in the widget tree.

**4. Is it user-facing?**
Yes — the refresh indicator is a visible progress bar shown to end users during data loading.

**5. What new did you learn from this file?**
The progress indicator appears between the filter bar and the item grid. This positioning means it is visually prominent and not hidden behind items, providing clear feedback during refresh.

---

## src/components/GalleryHeader.tsx

**1. What is the purpose of this file?**
Renders the filter/sort section and provides context APIs (filter, sort, selection) to child widgets in the `filtersPlaceholder` slot.

**2. What kind of logic is described in this file?**
MobX observer. If `filtersPlaceholder` is not provided, returns null. Otherwise, wraps the placeholder in three React context providers: `FilterAPI.Provider` (from `@mendix/widget-plugin-filtering`), `SortAPI.Provider` (from `@mendix/widget-plugin-sorting`), and `SelectionContext.Provider` (from `@mendix/widget-plugin-grid`). Wrapped in `<section class="widget-gallery-header widget-gallery-filter">`.

**3. What part of behavior can be documented from this file?**
- The `filtersPlaceholder` slot provides filter, sort, and selection context to any widgets placed there — e.g., DataGrid filter widgets (text filter, dropdown filter, date filter, etc.) can be placed here and will communicate with the Gallery.
- If no widgets are placed in `filtersPlaceholder`, the entire header section is omitted from the DOM.
- Selection context is provided to the filter section, enabling filter widgets to interact with selection state.

**4. Is it user-facing?**
Yes — the filter/sort header is visible to users when filter widgets are configured.

**5. What new did you learn from this file?**
The Gallery provides all three context types to the filters area: FilterAPI, SortAPI, and SelectionContext. This makes the filters placeholder a rich integration point — not just filtering but also sorting (via DropdownSort widget) and selection awareness.

---

## e2e/Gallery.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for Gallery sorting, filtering, onClick, and accessibility.

**2. What kind of logic is described in this file?**
Tests: (1) default sort order applied; (2) sort by Age via dropdown sort widget changes order; (3) text filter by "Leo"; (4) number filter by "32" yields 2 items; (5) date filter by "10/10/1986"; (6) enum dropdown filter by "QA Engineer"; (7) onClick context passes item data (shows "You've clicked at Ana Carol's face." in popup); (8) WCAG 2.1 AA accessibility scan passes.

**3. What part of behavior can be documented from this file?**
- Sorting: datasource default sort order is applied at render. Dropdown sort widgets in the filter placeholder can change the sort dynamically.
- Filtering: text, number, date, and enum (dropdown) filters work via the standard filter widget protocol.
- onClick: the clicked item's data context is passed to the action — confirmed by popup text containing item attribute values.
- Accessibility: the gallery (`.mx-name-gallery1`) passes WCAG 2.1 AA automated checks.

**4. Is it user-facing?**
Yes — verifies end-to-end user interactions.

**5. What new did you learn from this file?**
Number filter showing exactly 2 items for "32" confirms numeric filtering is exact/equality-based (not contains). The session logout after each test (`window.mx.session.logout()`) is required to avoid Mendix license limit of 5 concurrent sessions in e2e tests.

---

## e2e/GallerySelection.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for Gallery single and multi selection behavior and accessibility.

**2. What kind of logic is described in this file?**
Tests: (1) single selection — navigate to `/p/single-selection`, click an image item, verify visual snapshot; (2) multi selection — navigate to `/p/multi-selection`, hold Shift and click 3 items, verify snapshot; (3) accessibility scan on multi-selection page passes WCAG 2.1 AA.

**3. What part of behavior can be documented from this file?**
- **Single selection** (e2e-confirmed): clicking an image inside a gallery item triggers single selection.
- **Multi selection** (e2e-confirmed): holding Shift while clicking items selects multiple items (range selection). This confirms Shift+click is the range selection gesture in e2e.
- Multi-selection page passes WCAG 2.1 AA with selected items.
- Selection visual feedback is captured via Playwright snapshots.

**4. Is it user-facing?**
Yes — confirms selection behavior from the end-user perspective.

**5. What new did you learn from this file?**
Multi-selection uses Shift+click, not Ctrl+click. Ctrl/Meta+click is excluded from action handlers (used for modifier-aware selection in `createActionHandlers`). The e2e test confirms Shift+click as the multi-selection gesture.

---

## src/components/__tests__/GalleryRoot.spec.tsx

**1. What is the purpose of this file?**
Unit snapshot test for the `GalleryRoot` component.

**2. What kind of logic is described in this file?**
Creates a mock gallery container with `tabIndex=42` and `style={{color:"blue"}}`, renders `<GalleryRoot>` inside a `ContainerProvider`, and asserts the rendered HTML matches the snapshot.

**3. What part of behavior can be documented from this file?**
- `tabIndex` and `style` props are passed through to the root div via the view model.
- The container system correctly wires up from props through the DI container to the rendered output — confirmed integration test.

**4. Is it user-facing?**
No — unit/snapshot test.

**5. What new did you learn from this file?**
The snapshot test uses `createGalleryContainer` directly, confirming that the DI container approach is testable without mocking — the test provides real (mock) props and gets a real container.

---

## src/features/item-interaction/__tests__/item-keyboard.spec.tsx

**1. What is the purpose of this file?**
Comprehensive unit tests for Gallery item keyboard interaction and action/selection event routing.

**2. What kind of logic is described in this file?**
Tests the event-switch system with various keyboard events and selection types: Shift+Space (selection, not action); Cmd/Ctrl+A (selectAll only in Multi mode); Arrow keys with Shift (onSelectAdjacent only in Multi); PageUp/PageDown/Home/End with Shift (onSelectAdjacent in Multi); Space and Enter (both fire onExecuteAction); Enter fires regardless of selection type; Space fires action even when selection is Single/Multi (without Shift); keydown on a different element followed by keyup on item does not fire action.

**3. What part of behavior can be documented from this file?**
- **Shift+Space**: calls `onSelect` (selection toggle), NOT `onExecuteAction` — confirmed unit test.
- **Cmd/Ctrl+A**: calls `onSelectAll("selectAll")` only when `selectionType === "Multi"`. No effect for None or Single.
- **Shift+Arrow (Up/Down/Left/Right)**: calls `onSelectAdjacent` with direction and size in Multi mode. All four arrow directions work for adjacent selection extension.
- **Shift+PageUp/PageDown/Home/End**: calls `onSelectAdjacent` with code-based navigation in Multi mode.
- **Space (no Shift) and Enter**: both call `onExecuteAction` for any selection type (None, Single, Multi).
- **Key pair integrity**: action fires only if keydown was on the same target as keyup. Cross-element key sequences do not fire the action.

**4. Is it user-facing?**
No — unit tests. But documents the exact keyboard behavior spec.

**5. What new did you learn from this file?**
Left/Right arrow keys also trigger `onSelectAdjacent` in Multi mode (in addition to Up/Down), which makes sense for a 2D grid layout. PageUp/Down and Home/End navigate by column count, confirming that grid-aware keyboard navigation jumps across multiple rows.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents all notable changes across every released version of gallery-web.

**2. What kind of logic is described in this file?**
Versioned entries with Added/Fixed/Changed/Breaking changes sections, following Keep a Changelog format.

**3. What part of behavior can be documented from this file?**
Key version milestones:
- **v1.0.0 (2021-09-28)**: Initial release. Multiple filtering options, responsive desktop/tablet/phone columns, reuse of Data Grid filter widgets.
- **v1.3.0 (2023-03-29)**: Selection feature introduced (Single/Multi, onSelectionChange).
- **v1.4.0 (2023-10-31)**: Range selection (Shift+click), Select All (Cmd/Ctrl+A), keyboard item selection changed to Shift+Space. Breaking: new div wrapping each clickable item (HTML/CSS change).
- **v1.5.0 (2024-03-06)**: Keyboard navigation with arrow keys; Shift+arrow for multi-selection range extension.
- **v1.8.0 (2024-04-30)**: `itemSelectionMode` prop (toggle/clear); `onClickTrigger` prop (single/double).
- **v3.0.1 (2025-07-01)**: Fixed breakpoints not following `custom-variables` CSS file.
- **v3.1.0 (2025-07-24)**: Personalization — filter/sort state stored in attribute or browser localStorage. Fixed default sort order not being applied.
- **v3.2.0 (2025-08-18)**: Removed XPath metadata from storage to improve integration with external services.
- **v3.3.0 (2025-08-28)**: Refresh indicator (progress bar on datasource change).
- **v3.6.0 (2025-10-01)**: Fixed errors when Gallery is used inside an iframe.
- **v3.6.1 (2025-10-14)**: Fixed checkbox state not in sync with selection state.
- **v3.7.0 (2025-11-11)**: Configurable selection count visibility (top/bottom/off); customizable clear selection button label.
- **v3.8.0 (2026-01-16)**: Dutch translations; refresh interval property (auto-refresh on interval); fixed footer spacing; fixed row count display in virtual scroll.
- **v3.9.0 (2026-03-23)**: Dynamic pagination attributes (Page, Page size, Total count, Loaded rows); custom pagination drop zone; auto-select feature for Single and Multi selection modes.

**4. Is it user-facing?**
No — developer/operator documentation. But records behavioral changes that affect users.

**5. What new did you learn from this file?**
v3.6.0 fixed iframe compatibility, which means using Gallery inside Mendix popup dialogs or embedded iframes was broken before that version. v3.2.0 removed XPath metadata from personalization storage specifically to fix integration issues with other services — this was a breaking storage format change (though settings stored before would be invalidated by the schema validation).
