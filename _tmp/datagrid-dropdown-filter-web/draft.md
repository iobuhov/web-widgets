# Draft: datagrid-dropdown-filter-web

Widget: **Drop-down Filter** (`DatagridDropdownFilter`)
Package: `@mendix/datagrid-dropdown-filter-web` v3.10.0
Minimum Mendix: 9.17.0 | Offline capable: yes

---

## src/DatagridDropdownFilter.tsx

**1. Purpose:** Top-level container entry point for the widget. Routes rendering to one of two branches — `AttrFilter` (attribute-based filtering) or `RefFilter` (association-based filtering) — based on the `baseType` prop. Wrapped with `withPreloader` to block rendering while `defaultValue?.status === "loading"`.

**2. Logic:** A single branching function (`Container`) with two arms: `baseType === "attr"` renders `AttrFilter`, otherwise `RefFilter`. The `withPreloader` HOC adds a loading guard for async default values.

**3. Behavioral documentation:** The widget defers rendering until `defaultValue` resolves. This is a constraint: if a developer provides a dynamic default value that is slow to load, the widget will not render until it arrives. The two operating modes (attribute vs. association) are mutually exclusive; the `baseType` selection determines the entire downstream rendering path.

**4. User-facing:** Not directly — it is the container glue layer. End users see the rendered `AttrFilter` or `RefFilter` output.

**5. New findings:** The loading guard is scoped only to `defaultValue`, not to other async props (e.g., `refOptions`). This means association mode may render before its options datasource is ready.

---

## src/DatagridDropdownFilter.xml

**1. Purpose:** Mendix widget descriptor — defines all configurable properties, their types, defaults, and Studio Pro editor visibility. Canonical source for all user-facing configuration options.

**2. Logic:** Properties are grouped into `General > Data source`, `General > General`, `General > Configurations`, `General > Events`, `Advanced > Accessibility`, and `Advanced > Texts`. Conditional visibility of properties is enforced at editor-config level (see `editorConfig.ts`).

**3. Behavioral documentation:** Key property constraints:
- `baseType` ("attr" | "ref"): determines the filtering mode; "attr" supports Enum and Boolean attributes only.
- `attrChoice` ("auto" | "linked"): "auto" works only when the widget is placed inside a Data Grid column (reads from column context); "linked" requires an explicit `linkedDs` datasource.
- `auto` (boolean): when true, options are auto-derived from enum/boolean universe; when false, `filterOptions` (custom caption/value pairs) must be provided.
- `emptyOptionCaption`: hidden by editor config when `filterable` or `multiSelect` is true.
- `clearable`: hidden (and implicitly `true`) when `filterable` is true.
- `selectedItemsStyle` ("text" | "boxes") and `selectionMethod` ("checkbox" | "rowClick"): only shown when both `filterable` and `multiSelect` are true, and `selectionMethod` additionally requires `selectedItemsStyle === "boxes"`.
- `fetchOptionsLazy`: association mode only; enables deferred loading of reference options (with caveat: personalization value restoration is limited in lazy mode).
- `valueAttribute` (String only): stores the current filter selection as a string; associations are not supported for persistence.
- `refCaptionSource` ("attr" | "exp"): selects whether reference option captions come from an attribute or an expression.

**4. User-facing:** Fully user-facing — every property in this file is configurable in Studio Pro.

**5. New findings:** `offlineCapable="true"` is declared, indicating the widget can function offline. `refSearchAttr` is required only when `filterable=true`; it is hidden otherwise. The help URL points to Data Grid 2 docs section 7.2.

---

## typings/DatagridDropdownFilterProps.d.ts

**1. Purpose:** Auto-generated TypeScript interface derived from the XML descriptor. Provides compile-time type safety for all widget props in source code.

**2. Logic:** Declares `DatagridDropdownFilterContainerProps` (runtime) and `DatagridDropdownFilterPreviewProps` (Studio editor preview). Enumerates all prop types including union types for enumeration props.

**3. Behavioral documentation:** `attr: AttributeMetaData<string | boolean>` confirms the attribute filter supports only Enum (`string`) and Boolean attribute types. `refOptions?: ListValue` is optional, meaning the widget won't crash if the datasource is not configured, though the HOC will throw at runtime if it's missing. `valueAttribute?: EditableValue<string>` confirms the saved attribute must be String type (not integer, date, etc.).

**4. User-facing:** Not user-facing — developer/tooling artifact only.

**5. New findings:** `parentChannelName` does not appear in this generated interface, meaning it is injected at runtime by HOCs (not a widget prop). The preview props expose `refOptions` as `{} | { caption: string } | { type: string } | null` — used for design-time rendering without runtime data.

---

## src/components/AttrFilter.tsx

**1. Purpose:** Implements the attribute-based dropdown filter. Selects between two sub-paths: "auto" (store provided by parent Data Grid column) and "linked" (store constructed from an explicit datasource).

**2. Logic:** `AutoAttrFilter = withParentProvidedEnumStore(Connector)` pulls the filter store from the parent column context. `LinkedAttrFilter = withAttrGuard(withFilterAPI(withLinkedEnumStore(Connector)))` applies attribute validity check, connects to filter API, and constructs a local enum store. Both paths render `EnumFilterContainer` via `Connector`.

**3. Behavioral documentation:** In "auto" mode the widget is tightly coupled to its parent Data Grid column; it must be placed inside a column header to receive the store. In "linked" mode the `withFilterAPI` HOC connects to the broader filter context (e.g., a filter group). The `Connector` maps widget-level props to the container's API, resolving optional `DynamicValue<string>` props to plain strings with empty-string defaults.

**4. User-facing:** The rendered `EnumFilterContainer` is user-facing; this component itself is wiring.

**5. New findings:** `parentChannelName` is threaded through from the `FilterAPI` to the container, enabling cross-widget filter channel communication. `ariaLabel?.value ?? ""` means if `ariaLabel` is not set by the developer, screen readers get an empty label — no fallback label is generated.

---

## src/components/RefFilter.tsx

**1. Purpose:** Implements the association-based dropdown filter. Always uses the "linked" path — there is no "auto" mode for associations.

**2. Logic:** `RefFilter = withAttrGuard(withLinkedRefStore(Connector))`. The `withAttrGuard` checks `refEntity.filterable`. The `withLinkedRefStore` constructs a `RefFilterStore` from the filter API context. `Connector` renders `RefFilterContainer` with mapped props.

**3. Behavioral documentation:** Association filters always require an explicit entity and `refOptions` datasource — there is no column-context auto-detection for reference filtering. The same `parentChannelName` threading as `AttrFilter` applies. `emptyOptionCaption`, `emptySelectionCaption`, and `placeholder` are passed as plain strings with empty defaults when not configured.

**4. User-facing:** The rendered `RefFilterContainer` is user-facing; this component is wiring.

**5. New findings:** The absence of an "auto" path for association mode is architecturally significant: the developer must always explicitly configure `refEntity` and `refOptions` for association filtering.

---

## src/components/typings.ts

**1. Purpose:** Defines the internal prop interfaces that HOCs inject into the `Connector` functions: `EnumFilterProps` and `RefFilterProps`.

**2. Logic:** Each interface carries a `filterStore` (the reactive MobX store) and an optional `parentChannelName` string (for filter channel communication).

**3. Behavioral documentation:** The `filterStore` is the reactive state source for option lists, selection state, and filter conditions. `parentChannelName` is only set in certain filter channel configurations and enables external Set_Filter/Reset_Filter JS actions.

**4. User-facing:** Not user-facing — internal interface only.

**5. New findings:** The separation of `EnumFilterStore` and `RefFilterStore` types means the two filter modes have distinct store implementations with potentially different capabilities (e.g., lazy loading is a `RefFilterStore` feature).

---

## src/hocs/withAttrGuard.tsx

**1. Purpose:** Guards the wrapped component against non-filterable attributes or associations. Renders an error alert if the bound attribute is not filterable.

**2. Logic:** Checks `props.attr.filterable` (for attribute mode) or `props.refEntity.filterable` (for association mode). If false, renders a Bootstrap `danger` Alert with a link to Mendix attribute limitations docs.

**3. Behavioral documentation:** This is a hard behavioral constraint: if the configured attribute does not have filtering enabled in the Mendix data model, the widget will display an error message instead of the dropdown. The check is on the attribute's metadata at runtime. The comment notes this HOC should be kept in sync with a `withAttributeGuard` HOC elsewhere.

**4. User-facing:** The error alert is user-visible in the app, but it is intended as a developer-facing configuration error rather than an end-user message.

**5. New findings:** The guard applies to both linked (`withAttrGuard` wraps `LinkedAttrFilter`) and association (`withAttrGuard` wraps `RefFilter`) paths, but NOT to the "auto" path (`AutoAttrFilter`) — meaning auto mode relies on the parent column's own guard.

---

## src/hocs/withLinkedEnumStore.tsx

**1. Purpose:** HOC that constructs an `EnumStoreProvider` for the "linked" attribute filter path. Injects the `filterStore` prop into the wrapped component.

**2. Logic:** Uses `useSetup` to create an `EnumStoreProvider` with the `FilterAPI` from context and a config of `{ attributes: [props.attr], dataKey: props.name }`. The store is stable across re-renders (constructed once via `useSetup`).

**3. Behavioral documentation:** The `dataKey` is the widget's `name` prop (Mendix-assigned widget ID), used for identifying the filter in the filter channel. This HOC is only used in the "linked" path, meaning it requires an explicit attribute and datasource — the `FilterAPI` must be available in context (provided by the parent Data Grid or gallery).

**4. User-facing:** Not user-facing — infrastructure only.

**5. New findings:** The `EnumStoreProvider` takes an array of attributes (`[props.attr]`), suggesting the store design supports multi-attribute scenarios, though the widget currently only passes one attribute.

---

## src/hocs/withLinkedRefStore.tsx

**1. Purpose:** HOC that constructs a `RefFilterStore` provider for association filtering. Manages the reactive gate pattern for derived props and handles `FilterAPI` context errors.

**2. Logic:** Uses `GateProvider`/`DerivedPropsGate` to reactively update store props on each render. The `mapProps` function converts widget-level props to `RequiredProps`, selecting caption source (attribute or expression) and search attribute ID based on `refCaptionSource`. Wraps in a two-level HOC: inner `StoreProvider` (has `filterAPI`) and outer `FilterAPIProvider` (retrieves `filterAPI`).

**3. Behavioral documentation:** Critical behavioral constraint: if `refCaptionSource === "attr"`, the `refCaption` attribute is used as the search attribute ID as well (implicitly linking caption to search). If `refCaptionSource === "exp"`, `refSearchAttr` is used for search and can differ from the caption expression. If `refOptions` or `refEntity` are missing at runtime, the HOC throws a runtime error (not a graceful alert). `FilterAPI` errors (missing context) are rendered as a Bootstrap danger alert.

**4. User-facing:** The `FilterAPI` error alert is visible to end users (configuration error state).

**5. New findings:** The `initCond: null` in `RefFilterStore` construction means association filters always start with no filter condition applied (no default by this mechanism — default value is handled by `withPreloader` gating and the store's own initialization).

---

## src/hocs/withParentProvidedEnumStore.tsx

**1. Purpose:** HOC for "auto" mode — retrieves an `EnumFilterStore` directly from the parent column's filter context. No store construction; purely a context consumer.

**2. Logic:** Calls `useFilterAPI()` to get the filter API, then checks that the provider is a "direct" type with a "select" store type. Returns errors for: missing context, provider error, missing store, or store type mismatch.

**3. Behavioral documentation:** Behavioral constraint: the "auto" path requires the widget to be placed inside a Data Grid column with a properly configured enum/boolean attribute filter. If the parent does not provide a "direct" store, or if the store type is not "select", the widget renders an error alert. The `slctAPI` ref ensures referential stability of the filter store across re-renders.

**4. User-facing:** Error alerts are user-visible (configuration error states).

**5. New findings:** The `EMISSINGSTORE` and `EStoreTypeMisMatch` errors are standardized error objects from `@mendix/widget-plugin-filtering/errors`, suggesting a shared error vocabulary across filter widgets. The "select" store type requirement means this HOC only works with enum/boolean columns, not text/number/date filters.

---

## src/DatagridDropdownFilter.editorConfig.ts

**1. Purpose:** Studio Pro editor configuration — controls property visibility and provides structure preview rendering.

**2. Logic:** `getProperties` dynamically hides properties based on other property values. `getPreview` renders a bordered row-layout with the `emptySelectionCaption` text and a chevron icon. `check` runs validation (currently returns no errors).

**3. Behavioral documentation:** Key visibility rules that encode behavioral constraints:
- `emptyOptionCaption` is irrelevant (hidden) when `filterable` or `multiSelect` is true.
- `clearable` is hidden when `filterable` is true (always clearable in filterable mode — this is an implicit behavioral rule).
- `refSearchAttr` is only relevant when `filterable` is true; hidden otherwise.
- `filterInputPlaceholderCaption` is only relevant when `filterable` is true (no input is shown otherwise).
- `selectedItemsStyle` only applies when both `filterable` and `multiSelect` are enabled.
- `selectionMethod` (checkbox vs row-click) is only relevant when `selectedItemsStyle === "boxes"` in filterable+multiselect mode.
- Ref-mode properties (`fetchOptionsLazy`, `refCaption*`, `refEntity`, `refOptions`, `refSearchAttr`) are hidden in attribute mode and vice versa.
- In "auto" attribute mode, `linkedDs` and `attr` are hidden; in "linked" mode, `filterOptions` is hidden when `auto=true`.

**4. User-facing:** The structure preview is rendered in Studio Pro — developer-facing only.

**5. New findings:** The `check` function returns an empty array, meaning no static validation errors are generated — all constraint enforcement happens at runtime via HOC guards rather than design-time.

---

## src/DatagridDropdownFilter.editorPreview.tsx

**1. Purpose:** Renders the widget preview in Mendix Studio (lightweight design-time rendering).

**2. Logic:** Renders a `Select` control with empty options list. `getPreviewValue` returns the first truthy value among: `defaultValue`, `emptySelectionCaption`, or a fallback of "Search" (if filterable) / "Select" (otherwise). Uses `enableStaticRendering(true)` from MobX for SSR-safe preview.

**3. Behavioral documentation:** The preview only renders the outer `Select` shell — no real options, no filter store. The displayed text communicates the configuration intent to the developer. `clearable` is set to `!props.clearable` (inverted — shows as non-clearable in preview if clearable is disabled).

**4. User-facing:** Studio design-time only — not shown to end users.

**5. New findings:** `showCheckboxes: false` is hardcoded in preview, so the multiselect checkbox appearance is never previewed in Studio. The preview faithfully reflects the `clearable` state by inverting it for the `empty` prop.

---

## e2e/DataGridDropDownFilter.spec.js

**1. Purpose:** End-to-end Playwright tests for attribute-based dropdown filter functionality — enum, boolean, default values, and accessibility.

**2. Logic:** Tests cover: (1) visual snapshot baseline; (2) enum single/multi-select filtering and resulting data grid rows; (3) boolean filtering ("Yes"/"No" options); (4) default value initialization (single/multi mode, boolean/enum); (5) WCAG 2.1 AA accessibility scan.

**3. Behavioral documentation:** Confirmed behavioral facts:
- Enum filter: single selection filters to matching rows; multi-selection filters to union of matching rows (OR logic).
- Boolean filter: options are presented as "Yes" and "No" (third item in list). Selecting "None" (empty option) shows all 4 rows.
- Default value: when a default is configured, the data grid applies the filter on page load before any user interaction; confirmed for boolean and enum in both single and multi modes.
- Multi-mode default values: multiple comma-separated values (e.g., "Blue,Cyan,Red") are supported for enum multi-select initialization.
- Session cleanup is performed after each test (Mendix session logout) due to license limits.

**4. User-facing:** Yes — tests directly exercise user-facing widget behavior.

**5. New findings:** The accessibility scan runs WCAG 2.1 AA checks (`wcag21aa`) using axe-core, excluding `navigationTree3`. The widget passes with zero violations. The filter condition is applied immediately on option click with a 300ms delay wait, suggesting near-instant filter propagation.

---

## e2e/DataGridDropDownFilterAssociation.spec.js

**1. Purpose:** End-to-end Playwright tests for association-based dropdown filter functionality — single-select, multi-select, and state persistence.

**2. Logic:** Tests cover: (1) single-select: list contains 21 items (20 companies + 1 empty "Nada" option at top); (2) selecting an option filters data grid rows and displays selected value in toggle button; (3) multi-select: "Role" entity filter renders multiple role options and filters rows; (4) state persistence: checked options remain checked after menu close and reopen.

**3. Behavioral documentation:** Confirmed behavioral facts:
- Association filter list: empty option ("None/Nada") is always shown at the top (first item) of the list.
- After selecting a company, the data grid shows only rows associated to that company (2 rows confirmed for one company).
- Multi-select: selecting one role returns 4 matching rows (3 data + 1 header). Options are confirmed as role names (e.g., "Economist", "Public librarian", "Prison officer").
- State persistence: a checked option remains checked after closing and reopening the dropdown menu. Multiple selections also persist.

**4. User-facing:** Yes — tests directly exercise user-facing association filter behavior.

**5. New findings:** The association filter page is at `/p/associations-filter` — a separate test page from the main filter page. The multi-select role filter uses `.mx-name-drop_downFilter1` while single-select uses `.mx-name-drop_downFilter2`, confirming multiple filter instances can coexist on the same page. The toggle button (`.widget-dropdown-filter-toggle`) displays the selected value text.

---

## src/__tests__/DatagridDropdownFilter.spec.tsx

**1. Purpose:** Unit test for the widget's React rendering using Jest + Testing Library. Confirms correct rendering when a valid `FilterAPI` context is provided.

**2. Logic:** Sets up an `EnumFilterStore` with a single enum attribute (universe: ["a", "b", "c"]), injects it via a React context at `com.mendix.widgets.web.filterable.filterContext.v2`, renders the widget with `commonProps`, and asserts a `combobox` role element appears.

**3. Behavioral documentation:** The rendered output uses `combobox` ARIA role — consistent with a single-select non-filterable dropdown. The filter context key (`com.mendix.widgets.web.filterable.filterContext.v2`) is the contract between the parent widget and this filter widget. `EnumFilterStore` is initialized with `null` initial condition (no active filter). `commonProps` sets `attrChoice: "auto"`, `baseType: "attr"`, `clearable: false`, `filterable: false`, `multiSelect: false`, `selectedItemsStyle: "text"`, `selectionMethod: "checkbox"`.

**4. User-facing:** Not user-facing — unit test infrastructure.

**5. New findings:** The snapshot test path is `__snapshots__/DatagridDropdownFilter.spec.tsx.snap`. The `waitFor(() => getByRole("combobox"))` suggests async rendering — the component does not synchronously render the final state (likely awaiting context resolution). `ariaLabel: dynamicValue("AriaLabel")` is set in test props, confirming the combobox role element receives the aria-label.

---

## shared/widget-plugin-dropdown-filter/src/containers/EnumFilterContainer.tsx

**1. Purpose:** Core rendering container for enum/boolean attribute filtering. Selects between three UI frontend types (Select, Combobox, TagPicker) based on `filterable` and `multiselect` configuration.

**2. Logic:** `useFrontendType` determines: `filterable=false` → `select` (EnumSelectController + Select); `filterable=true, multiselect=false` → `combobox` (EnumComboboxController + Combobox); `filterable=true, multiselect=true` → `tagPicker` (EnumTagPickerController + TagPicker). Uses `GateProvider`/`DerivedPropsGate` for reactive prop updates. `usePickerJSActions` wires Set_Filter/Reset_Filter JS actions. Wrapped with `withCustomOptionsGuard` for custom option validation.

**3. Behavioral documentation:** Three distinct UI modes map to different interaction patterns: (1) Select — simple dropdown list, supports checkboxes for multiselect, no text input; (2) Combobox — text input for filtering options, single value; (3) TagPicker — text input + tag display for multi-value selection. The `withCustomOptionsGuard` validates each custom option's value against the store's `isValidValue` method — invalid values (e.g., non-enum values) cause the entire container to render an error alert.

**4. User-facing:** Yes — renders the actual dropdown UI.

**5. New findings:** `withCustomOptionsGuard` validates options using `useState` (once on mount), so option validation is not reactive — changes to `filterOptions` after mount are not re-validated. `usePickerJSActions` integrates JS action support (`Set_Filter`/`Reset_Filter`) in all three frontend modes.

---

## shared/widget-plugin-dropdown-filter/src/containers/RefFilterContainer.tsx

**1. Purpose:** Core rendering container for association/reference filtering. Same three-frontend-type architecture as `EnumFilterContainer` but for `RefFilterStore`.

**2. Logic:** Same `useFrontendType` switch. Reference-specific controllers: `RefSelectController`, `RefComboboxController`, `RefTagPickerController`. `useOnScrollBottom` is used by all three modes to trigger lazy loading when the user scrolls to the bottom of the option list. NOT wrapped with `withCustomOptionsGuard` (custom options not supported for reference filters).

**3. Behavioral documentation:** Association filter has scroll-triggered lazy loading built in at the container level (100px trigger zone). The absence of `withCustomOptionsGuard` confirms that custom `filterOptions` configuration is only applicable to attribute mode, not association mode. `RefFilterContainer` is exported directly (no guard wrapper), meaning it trusts the upstream HOC chain to validate refs.

**4. User-facing:** Yes — renders the association filter dropdown UI.

**5. New findings:** `onFocus={ctrl1.handleFocus}` is passed in `RefSelectController` but not in `EnumSelectController`, suggesting reference filters may have focus-triggered behavior (e.g., triggering an initial options load on focus). All three ref controllers receive `onMenuScroll={handleMenuScroll}` for lazy loading, while enum controllers do not — enum options are always fully loaded upfront.

---

## shared/widget-plugin-dropdown-filter/src/helpers/useFrontendType.ts

**1. Purpose:** Determines which UI frontend type (select/combobox/tagPicker) to render based on `filterable` and `multiselect` props.

**2. Logic:** Three-way mapping: `filterable=false` → `select`; `filterable=true && multiselect=false` → `combobox`; `filterable=true && multiselect=true` → `tagPicker`.

**3. Behavioral documentation:** This is the canonical behavioral rule for UI mode selection. Users cannot directly choose "combobox" or "tagPicker" — these modes are activated by combining `filterable=true` with `multiSelect`. The `select` mode (non-filterable) is the default and simplest UI.

**4. User-facing:** Not directly — pure logic helper, but determines user-visible UI.

**5. New findings:** There is no way to get a `tagPicker` without `filterable=true`. Conversely, enabling `filterable=true` without `multiSelect` always gives a `combobox` (not a `select`). The `selectionMethod` and `selectedItemsStyle` props only matter in `tagPicker` mode but are defined in the widget XML for the whole widget.

---

## shared/widget-plugin-dropdown-filter/src/hocs/withCustomOptionsGuard.tsx

**1. Purpose:** Validates custom filter options (`filterOptions`) against the active `EnumFilterStore`, preventing invalid values from being applied.

**2. Logic:** On mount, iterates all `filterOptions` and calls `filterStore.isValidValue(value)` for each. If any option is invalid, renders a Bootstrap danger alert with the first error message. Uses `useState` for one-time validation.

**3. Behavioral documentation:** Behavioral constraint: custom options must be valid enum values (or Boolean string representations) that the filter store recognizes. An invalid option (e.g., a string value that doesn't match any enum key) causes the entire widget to render an error alert instead of the dropdown. This guard is applied to enum/boolean attribute mode only, not association mode.

**4. User-facing:** The error alert is user-visible when invalid options are configured.

**5. New findings:** The `value` is coerced to string via template literal: `` `${opt.value?.value}` ``. If `opt.value?.value` is undefined, it becomes the string `"undefined"`, which will likely fail `isValidValue`, causing an error for unconfigured option expressions.

---

## CHANGELOG.md

**1. Purpose:** Version history documenting all notable changes to the widget.

**2. Logic:** Follows Keep a Changelog format with Semantic Versioning.

**3. Behavioral documentation — selected version highlights:**

**v3.10.0 (2026-05-06):** Fixed filter selector dropdown placement on small viewports. Behavioral constraint: the filter's positioning menu now intelligently selects the best placement based on available viewport space.

**v3.0.4 (2025-08-05) [BREAKING]:** Separated text configurations for three distinct text areas: `emptyOptionCaption` (the "None" option in the list), `emptySelectionCaption` (shown when nothing is selected), and `filterInputPlaceholderCaption` (input placeholder in filterable mode). Prior configurations must be reconfigured. Also: enhanced keyboard navigation.

**v3.0.3 (2025-07-24):** Fixed "Use Lazy Load" flag not being read correctly — the widget was always loading lazily in association mode regardless of the setting.

**v2.10.0 (2025-02-20):** New settings added to make the dropdown look and behave like a combobox widget. HTML updated for modern accessibility standards.

**v2.9.0 (2024-09-20) [BREAKING]:** Removed the "Group key" property (which had been added in v2.8.0 and immediately removed).

**v2.7.0 (2024-06-19):** Added `Set_Filter` JS action hook. Updated `Reset_Filter` to support reset to default value (not just clear).

**v2.4.0 (2023-05-01) [BREAKING]:** Default value attribute is now only used as the initial value; subsequent changes to it after widget mount are ignored. This is a key behavioral constraint.

**v2.3.0 (2023-02-17):** Added association filtering support (first version with `ref` mode).

**v1.3.0 (2021-07-07):** Added Boolean attribute support.

**4. User-facing:** Indirectly — behavioral changes affect end-user experience.

**5. New findings:** The widget has had three breaking changes: v2.0.0 (renamed), v2.4.0 (default value semantics), v2.9.0 (group key removed), v3.0.4 (text configuration split). The current version (3.10.0) is the 19th release since v1.3.0, reflecting active ongoing development.
