# Draft: datagrid-number-filter-web

Widget package: `packages/pluggableWidgets/datagrid-number-filter-web`  
Extracted by: worker  
Date: 2026-05-09

---

## src/DatagridNumberFilter.tsx

**1. What is the purpose of this file?**  
Root entry point for the DatagridNumberFilter widget. Selects between two rendering paths based on the `attrChoice` prop: "auto" (parent DataGrid or Gallery provides the filter store via React context) or "linked" (the widget owns and manages its own store using explicitly configured attributes and a linked datasource).

**2. What kind of logic is described in this file?**  
Routing logic only. Three HOC pipelines are assembled at module level: `Container` wraps `NumberFilterContainer` with a preloader; `FilterAuto` wraps `Container` with the parent-provided store HOC; `FilterLinked` wraps `Container` with attribute guard, filter API context, and linked store. The default export selects between them based on `attrChoice`.

**3. What part of behavior can be documented from this file?**  
The `attrChoice` prop is the switch point for integration modes. When `attrChoice === "auto"`, the widget expects its parent (Data Grid 2 or Gallery) to inject a `Number_InputFilterInterface` store via React context. When `attrChoice === "linked"`, the widget creates its own store from configured attributes. The `withPreloader` HOC delays rendering until `isLoadingDefaultValues` returns false, preventing initialization with a pending default value.

**4. Is it user-facing?**  
Not directly — invisible routing layer. Users never see it but its routing determines which configuration path applies.

**5. What new did you learn from this file?**  
The "linked" mode requires both `withAttributeGuard` and `withFilterAPI`, meaning those HOCs enforce that a valid numeric attribute and filter API context are present before rendering. The preloader ensures the widget is never initialized with a `loading` default value expression.

---

## src/DatagridNumberFilter.xml

**1. What is the purpose of this file?**  
Widget descriptor XML defining all user-configurable properties in Mendix Studio Pro. This is the contract between the platform and the widget.

**2. What kind of logic is described in this file?**  
Property declarations: types, captions, enumerations, and groupings. Covers four property groups: General (attribute mode, defaults, placeholder, adjustable, delay), Configurations (saved attribute), Events (onChange), and Accessibility (screen reader captions).

**3. What part of behavior can be documented from this file?**  
- `attrChoice` (enum: "auto" | "linked") controls attribute source mode.  
- `attributes` list property links to `linkedDs` and accepts only AutoNumber, Decimal, Integer, or Long attribute types.  
- `defaultFilter` (enum, default "equal") sets the initial filter comparison operator — 8 options: greater, greaterEqual, equal, notEqual, smaller, smallerEqual, empty, notEmpty.  
- `defaultValue` (Decimal expression) sets the initial filter value.  
- `adjustable` (boolean, default true) allows end-users to switch the filter type at runtime.  
- `delay` (integer, default 500ms) debounces filter application after the user stops typing.  
- `valueAttribute` (EditableValue, accepts AutoNumber, Decimal, Integer, Long) stores the last filter value for persistence.  
- `onChange` (action) fires when value or filter type changes.  
- Two screen reader captions: `screenReaderButtonCaption` (comparison button) and `screenReaderInputCaption` (input element, default "Search" in English).

**4. Is it user-facing?**  
Yes — defines everything developers configure in Studio Pro, which directly determines end-user behavior.

**5. What new did you learn from this file?**  
The widget is categorized under "Data controls" in both Studio and Studio Pro. It supports offline capability (`offlineCapable="true"`). The `attributes` list is only relevant in "linked" mode; in "auto" mode, the parent DataGrid provides the filterable attribute. The `screenReaderInputCaption` has built-in translations for en_US ("Search"), de_DE ("Suche"), and nl_NL ("Zoeken").

---

## src/DatagridNumberFilter.editorConfig.ts

**1. What is the purpose of this file?**  
Defines Studio Pro editor configuration: which properties are shown or hidden based on current prop values, and what the widget looks like in structure/design mode canvas preview.

**2. What kind of logic is described in this file?**  
`getProperties` hides irrelevant properties based on `adjustable` and `attrChoice`. `getPreview` builds a structure preview with an optional filter type icon (when adjustable), a separator, and placeholder text in a bordered row.

**3. What part of behavior can be documented from this file?**  
- When `adjustable=false`, the `screenReaderButtonCaption` property is hidden (no filter type button exists).  
- When `attrChoice === "auto"`, the `attributes` list and `linkedDs` properties are hidden.  
- The preview renders SVG icons for all 8 filter types in both light and dark variants (equal, notEqual, greater, greaterEqual, smaller, smallerEqual, empty, notEmpty).

**4. Is it user-facing?**  
Only in Studio Pro's design canvas — no runtime behavior.

**5. What new did you learn from this file?**  
The preview composition uses `@mendix/widget-plugin-platform`'s structure preview API. The filter icon in the preview matches the configured `defaultFilter`, giving developers a visual indicator of the default comparison mode before running the app.

---

## src/DatagridNumberFilter.editorPreview.tsx

**1. What is the purpose of this file?**  
Provides the React preview component used by Studio Pro's page editor to render the widget in design/xray/structure mode.

**2. What kind of logic is described in this file?**  
Enables MobX static rendering. The `Preview` function creates two `InputStore` instances (one seeded with `defaultValue`, one empty), then renders `InputWithFiltersComponent` with non-interactive handlers for the preview context.

**3. What part of behavior can be documented from this file?**  
The preview is non-interactive but visually accurate — it reflects the configured `defaultFilter`, `adjustable`, placeholder, and screen reader captions. Two `InputStore` instances are created per preview render, matching the runtime controller's dual-input model.

**4. Is it user-facing?**  
Only in Studio Pro design mode — not at runtime.

**5. What new did you learn from this file?**  
The controller manages two `InputStore` instances even for single-value filtering — this mirrors the `InputWithFiltersComponent` API which expects `inputStores` as a tuple. Static MobX rendering is enabled to prevent unnecessary subscription activity in the preview context.

---

## src/components/NumberFilterContainer.tsx

**1. What is the purpose of this file?**  
Primary runtime container that bridges the widget props and the filter store to the `InputWithFilters` presentation layer. MobX observer component.

**2. What kind of logic is described in this file?**  
Uses `useSetup` to create a `NumberFilterController`; subscribes to external `Reset_Filter` and `Set_Filter` events; calls `useBasicSync` to sync `valueAttribute` and `onChange` with the filter store; then renders `InputWithFilters` with controller-derived state.

**3. What part of behavior can be documented from this file?**  
- `NumberFilterController` initializes with: `changeDelay` (debounce), `defaultFilter`, `adjustableFilterFunction`, `defaultValue`, and a `disableInputs` callback that returns `true` when the selected filter function is "empty" or "notEmpty" (no numeric value needed for those comparisons).  
- The 8 filter labels (displayed in the dropdown) are: "Greater than", "Greater than or equal", "Equal", "Not equal", "Smaller than", "Smaller than or equal", "Empty", "Not empty".  
- `parentChannelName` enables cross-widget event scoping (filter group coordination).  
- `tabIndex` is passed through to the UI.  
- External `Reset_Filter` event triggers `controller.handleResetValue`; external `Set_Filter` triggers `controller.handleSetValue`.

**4. Is it user-facing?**  
Yes — this component controls the direct user interaction layer.

**5. What new did you learn from this file?**  
The input is disabled (not just hidden) when the filter function is "empty" or "notEmpty" — this is a runtime behavioral constraint enforced by the controller. The UUID-based `id` is generated once per component instance via a ref, ensuring stable accessibility IDs across re-renders. The component is explicitly typed as `observer`, meaning it tracks MobX observables from the controller.

---

## src/components/typings.ts

**1. What is the purpose of this file?**  
Defines the `NumberFilterProps` interface shared between the container and HOCs — the contract any HOC must fulfill to wrap `NumberFilterContainer`.

**2. What kind of logic is described in this file?**  
Type definitions only: `filterStore: Number_InputFilterInterface` (required) and `parentChannelName?: string` (optional).

**3. What part of behavior can be documented from this file?**  
Every runtime HOC path must provide a `Number_InputFilterInterface` store. The optional `parentChannelName` enables channel-scoped event delivery for filter group scenarios.

**4. Is it user-facing?**  
No — internal interface definition.

**5. What new did you learn from this file?**  
The `filterStore` type (`Number_InputFilterInterface`) comes from `@mendix/widget-plugin-filtering`, the shared internal filter package, confirming the number filter integrates with the standard Mendix filter store abstraction.

---

## src/hocs/withLinkedNumberStore.tsx

**1. What is the purpose of this file?**  
Higher-order component that creates a `NumberStoreProvider` (MobX store) from the widget's explicitly configured attributes and filter API context, then passes the resulting store to the wrapped component.

**2. What kind of logic is described in this file?**  
Uses `useSetup` to instantiate `NumberStoreProvider` once per mount. The provider is constructed with `filterAPI`, the list of `AttributeMetaData<Big>` attributes, and a `dataKey` (`props.name`) for store identity.

**3. What part of behavior can be documented from this file?**  
- Used only when `attrChoice === "linked"` (Custom mode).  
- Requires `attributes` array and `name` prop.  
- The `dataKey: props.name` means the filter store is keyed by the widget's Mendix name — ensures uniqueness when multiple linked number filters are on the same page.  
- `parentChannelName` is threaded from the `filterAPI` context.

**4. Is it user-facing?**  
Indirectly — enables the "Custom" configuration mode where the widget manages its own filter store.

**5. What new did you learn from this file?**  
The `NumberStoreProvider` is from the shared `@mendix/widget-plugin-filtering` package, consistent with other filter widgets. Attribute values are typed as `AttributeMetaData<Big>` (using `big.js`), confirming all numeric attribute types are handled as arbitrary-precision decimals internally.

---

## src/hocs/withParentProvidedNumberStore.tsx

**1. What is the purpose of this file?**  
Higher-order component that reads the number filter store from the parent widget's React context (Data Grid 2 or Gallery) and passes it to the wrapped component. Handles all error states.

**2. What kind of logic is described in this file?**  
`useNumberFilterAPI` reads the filter context via `useFilterAPI`. Validates: context is present, provider has no error, store type is "direct", store is input-type, and store is a number filter (via `isNumberFilter`). Returns a Result monad. On error renders `<Alert bootstrapStyle="danger">`. Caches the resolved API object in a ref.

**3. What part of behavior can be documented from this file?**  
- Used when `attrChoice === "auto"`.  
- If placed outside a supported parent widget, renders an error: `EMISSINGSTORE` message.  
- If the parent provides a store of the wrong type (e.g., a text filter store), a type mismatch error is shown from `EStoreTypeMisMatch("number filter", ...)`.  
- The `numberAPI.current ??= {...}` pattern caches the resolved store reference across re-renders to prevent unnecessary re-renders of the wrapped component.

**4. Is it user-facing?**  
Yes — the error alert is shown to developers (and potentially end users) when the widget is placed incorrectly.

**5. What new did you learn from this file?**  
The error alert rendered by this HOC confirms the unit test snapshot: "The filter widget must be placed inside the column or header of the Data grid 2.0 or inside header of the Gallery widget." The `isNumberFilter` check (from `@mendix/widget-plugin-filtering/stores/input/store-utils`) validates the store's `arg1.type` is numeric, not just that it's an input-type store.

---

## src/utils/widget-utils.ts

**1. What is the purpose of this file?**  
Provides the `isLoadingDefaultValues` predicate used by the preloader HOC to delay rendering until the default value is resolved.

**2. What kind of logic is described in this file?**  
Single function: checks if `props.defaultValue?.status === "loading"`. Returns boolean.

**3. What part of behavior can be documented from this file?**  
The widget defers rendering while the `defaultValue` expression is in a loading state. This ensures the filter is initialized exactly once with the correct default — no flash of empty state. Unlike the date filter, this widget checks only one default value (no start/end date range).

**4. Is it user-facing?**  
Indirectly — prevents visual flash or incorrect initial filter state.

**5. What new did you learn from this file?**  
The number filter has a simpler default value structure than the date filter — only a single `defaultValue` expression, no start/end pair. The preloader check is correspondingly simpler.

---

## typings/DatagridNumberFilterProps.d.ts

**1. What is the purpose of this file?**  
Auto-generated TypeScript type definitions from the widget XML. Provides the full typed interface for the widget's container props and preview props.

**2. What kind of logic is described in this file?**  
Type declarations only. Defines `AttrChoiceEnum`, `AttributesType`, `DefaultFilterEnum`, `DatagridNumberFilterContainerProps`, and `DatagridNumberFilterPreviewProps`.

**3. What part of behavior can be documented from this file?**  
Complete container props interface:  
- `name`, `class`, `style?`, `tabIndex?` — standard Mendix widget props  
- `attrChoice: "auto" | "linked"`  
- `attributes: AttributesType[]` — list of numeric attributes (`AttributeMetaData<Big>`)  
- `defaultValue?: DynamicValue<Big>`  
- `defaultFilter: DefaultFilterEnum` (8 options)  
- `placeholder?: DynamicValue<string>`  
- `adjustable: boolean`  
- `delay: number`  
- `valueAttribute?: EditableValue<Big>`  
- `onChange?: ActionValue`  
- `screenReaderButtonCaption?`, `screenReaderInputCaption?: DynamicValue<string>`

**4. Is it user-facing?**  
No — internal type definitions.

**5. What new did you learn from this file?**  
All numeric values (default, stored, input) use `Big` from `big.js` for arbitrary-precision decimal handling. The `className` field in preview props is marked `@deprecated since version 9.18.0` — use `class` instead. `linkedDs` is referenced only in the XML (for attribute binding) and is absent from the generated TypeScript props.

---

## typings/FilterType.d.ts

**1. What is the purpose of this file?**  
Re-exports `DefaultFilterEnum` from the generated props under the alias `FilterType`.

**2. What kind of logic is described in this file?**  
Single type re-export. No logic.

**3. What part of behavior can be documented from this file?**  
`FilterType` is identical to `DefaultFilterEnum`: `"greater" | "greaterEqual" | "equal" | "notEqual" | "smaller" | "smallerEqual" | "empty" | "notEmpty"`.

**4. Is it user-facing?**  
No — internal type alias for external consumers of this package.

**5. What new did you learn from this file?**  
This re-export allows other packages to import the filter type union without depending on the generated props file directly.

---

## e2e/DataGridNumberFilter.spec.js

**1. What is the purpose of this file?**  
End-to-end Playwright tests for the DataGrid Number Filter widget.

**2. What kind of logic is described in this file?**  
Tests cover: visual screenshot baseline; number filtering result correctness; default value initialization on page load; and WCAG 2.1 AA accessibility scan using axe-core.

**3. What part of behavior can be documented from this file?**  
- **Visual baseline**: A screenshot comparison confirms all datagrid and filter elements render correctly (threshold 0.1).  
- **Number filtering**: Typing "12" into the filter input (`mx-name-datagrid1 input`) causes the datagrid to show exactly 2 rows with cell values `["12", "test3", "test3", ""]` — e2e-confirmed filtering by equality.  
- **Default value initialization**: On `/p/filter_init_condition`, `mx-name-dataGrid21` pre-filters to rows matching year 1987 — shows "Delia1987" and "Lizzie1987" with paging status "1 to 2 of 2". The filter is applied immediately after page load (not requiring user interaction).  
- **Accessibility**: WCAG 2.1 AA violations are zero (excluding navigation tree).  
- Session logout is performed after each test to stay within the 5-session Mendix license limit.

**4. Is it user-facing?**  
Tests only, but confirms end-user-facing behaviors.

**5. What new did you learn from this file?**  
The default value test uses a numeric year value (1987) as the filter, confirming Integer-type attribute filtering is e2e-verified. The equality filter is the default mode confirmed in this test (no filter type switch performed). The `/p/filter_init_condition` route is shared with other filter widgets for testing initial condition behavior.

---

## src/components/__tests__/DatagridNumberFilter.spec.tsx

**1. What is the purpose of this file?**  
Unit tests for the root `DatagridNumberFilter` component, covering rendering, event handling, external events, default value behavior, and multi-instance uniqueness.

**2. What kind of logic is described in this file?**  
Tests cover: render snapshot with single attribute (Long type); render with multiple attributes (Long + Decimal); error alert when no FilterAPI context; `onChange` and `valueAttribute.setValue` on filter value change; external reset event (with and without default value); `defaultValue` initialization and no-sync-after-mount behavior; external `Set_Filter` event; and unique `aria-controls` IDs across instances.

**3. What part of behavior can be documented from this file?**  
- Supported attribute types confirmed in tests: Long, Decimal (for multi-attribute scenario).  
- Without FilterAPI context, renders: "The filter widget must be placed inside the column or header of the Data grid 2.0 or inside header of the Gallery widget."  
- On filter value change, both `onChange.execute()` and `attribute.setValue(new Big("10"))` are called once (after debounce delay runs).  
- **Reset behavior (no default)**: After typing "42", reset event clears input to "" and calls `attribute.setValue(undefined)`.  
- **Reset behavior (with default)**: After typing "42", reset event (with `true` flag) restores input to "123" (the `defaultValue`) and calls `attribute.setValue(Big(123))`.  
- **defaultValue constraint**: `defaultValue` is only used as an initial value. Changing `defaultValue` prop after mount (undefined → number, or number → undefined) does NOT update the filter input — confirmed by two explicit tests.  
- **Set_Filter event**: emitting `set.value` with `{numberValue: Big(100)}` updates input to "100" and calls `attribute.setValue(Big(100))`.  
- Multiple instances render with different `aria-controls` attributes on the filter type button.  
- FilterAPI is injected via global window key `"com.mendix.widgets.web.filterable.filterContext.v2"`.

**4. Is it user-facing?**  
Tests only, but directly confirms user-facing behaviors.

**5. What new did you learn from this file?**  
The debounce delay default in tests is 1000ms (the `commonProps` sets `delay: 1000`), while the production default is 500ms. The reset event uses a boolean flag to distinguish "reset to default" (`true`) from "clear all" (`false`). External `Set_Filter` events are scoped to the widget's `name` (`"filter-test"` in the test), not the channel name.

---

## src/components/__tests__/__snapshots__/DatagridNumberFilter.spec.tsx.snap

**1. What is the purpose of this file?**  
Jest snapshot file capturing the expected HTML output for the DatagridNumberFilter component in different states.

**2. What kind of logic is described in this file?**  
Three snapshots: single-attribute render, multiple-attribute render, and no-context error render.

**3. What part of behavior can be documented from this file?**  
- DOM structure: outer `div.filter-container.{class}` with `data-focusindex="0"`.  
- Filter selector: `div.filter-selector > div.filter-selector-content > button.btn.btn-default.filter-selector-button.button-icon.equal` with `role="combobox"`, `aria-haspopup="listbox"`, `aria-label="Equal"`, and `aria-controls` pointing to the hidden listbox.  
- Hidden listbox: `ul.filter-selectors.hidden` with `aria-label="Select filter type"` using `position: fixed`.  
- Input: `input.form-control.filter-input` with `type="text"` and `aria-label="filter"`.  
- Error state: `div.alert.alert-danger` containing the placement error message.  
- Single and multiple attribute renders produce identical HTML structure (the store difference is internal, not structural).

**4. Is it user-facing?**  
Tests only — confirms the rendered HTML structure users interact with.

**5. What new did you learn from this file?**  
The listbox uses `position: fixed` (not absolute) — consistent with the `popperProps.strategy = "fixed"` approach used in the date filter to prevent clipping in scrollable containers. The filter type button's `aria-label` reflects the currently selected filter function name ("Equal" for default), providing an accessible description without a visible label.

---

## CHANGELOG.md

**1. What is the purpose of this file?**  
Documents the version history of the datagrid-number-filter-web widget from v2.0.0 (2021-09-28) to v3.10.0 (2026-05-06).

**2. What kind of logic is described in this file?**  
No logic — version entries with Added, Changed, Fixed, and Breaking changes categories.

**3. What part of behavior can be documented from this file?**  
Selected behavioral findings by version:

- **v3.10.0 (2026-05-06):** Fixed filter selector dropdown placement on small viewports. Fixed keyboard focus jumping away from filter controls when "Empty" or "Not empty" is selected.  
- **v3.9.0 (2026-03-23):** Fixed crash when Saved attribute (`valueAttribute`) is configured.  
- **v3.8.1 (2026-02-19):** Filter selector dropdown now auto-selects best placement based on available space.  
- **v2.9.2 (2025-05-19):** Fixed number filter not working in some locales — behavioral constraint: locale affects how numbers are parsed from user input.  
- **v2.9.1 (2025-03-20):** Fixed: non-adjustable default filters were overridden by personalization settings — when `adjustable=false`, the filter type is protected from personalization.  
- **v2.8.4 (2024-11-13):** Breaking: filter type select button now opens on Enter, Space, and arrow keys. Fixed `onChange` not triggered for "empty"/"not empty" filter. Improved type mismatch error message. Added screen reader improvements.  
- **v2.8.0 (2024-09-20):** Breaking: removed "Group key" property.  
- **v2.7.0 (2024-09-13):** Added Group key for filter groups. (Removed in v2.8.0.)  
- **v2.6.0 (2024-06-19):** Added `Set_Filter` action hook. Updated `Reset_Filter` to support reset-to-default.  
- **v2.5.0 (2024-03-27):** Added external events hook (subscribe to `Reset_Filter` and `Set_Filter`).  
- **v2.4.0 (2023-05-01):** Breaking: default value is now initial value only — changes after mount are ignored.  
- **v2.3.0 (2022-05-11):** Added "empty" and "not empty" filter options.  
- **v2.0.2 (2021-10-13):** Added `valueAttribute` persistence and `onChange` event. Added "Enable advanced options" toggle for Studio users.  
- **v2.0.0 (2021-09-28):** Added Gallery compatibility. Added toolbox category. Renamed to "Number filter".

**4. Is it user-facing?**  
The documented behaviors are user-facing; the file itself is for developers.

**5. What new did you learn from this file?**  
v2.9.2's locale fix is significant: it confirms that number parsing is locale-sensitive (e.g., decimal separators may differ by locale). v2.4.0's "initial value only" default behavior is a breaking change — consistent with the date filter's same breaking change in v2.5.0. v2.8.0 removed the Group key property that was just added in v2.7.0 — this was a short-lived feature replaced by a different mechanism.
