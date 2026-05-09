# Draft: datagrid-text-filter-web

Extracted by worker on 2026-05-09. Covers all source files and local workspace dependencies.

---

## src/DatagridTextFilter.tsx

**Purpose:** Main widget entry point. Selects the correct HOC-wrapped component based on the `attrChoice` prop and renders it with all container props forwarded.

**Logic:** Two component variants are composed at module load time. `FilterAuto` wraps `TextFilterContainer` with `withParentProvidedStringStore` (reads the filter store from a parent context) and `withPreloader` (suspends rendering while defaults load). `FilterLinked` adds `withAttributeGuard`, `withFilterAPI`, and `withLinkedStringStore` on top — this path requires the developer to explicitly configure a datasource and attributes. The exported default function switches between them based on `props.attrChoice === "auto"`.

**Behavioral constraints from this file:**
- When `attrChoice === "auto"`, the widget reads its filter store from a parent widget's context (e.g., Data Grid 2); no datasource or attributes need to be configured by the developer.
- When `attrChoice === "linked"` (Custom), the widget manages its own filter store via an explicit linked datasource and attribute list; it must pass attribute guard and filter API checks.
- The `withPreloader` HOC calls `isLoadingDefaultValues` to block rendering until `defaultValue` has resolved — the widget shows nothing until the initial value is available.

**User-facing:** No — this is the orchestration layer; end users see only the rendered `InputWithFilters` component.

**New findings:** The dual-path architecture cleanly separates auto-wiring (context-driven) from explicit-wiring (developer-configured), using HOC composition rather than conditional logic inside the component itself. Both paths share the same `TextFilterContainer` rendering layer.

---

## src/DatagridTextFilter.xml

**Purpose:** Widget descriptor declaring all configurable properties exposed to Mendix Studio/Studio Pro.

**Logic:** Defines the widget as pluggable, offline-capable, and categorized under "Data Controls" in the toolbox. Declares the `attrChoice` enumeration (auto/linked), the optional `linkedDs` datasource, the `attributes` list (String only), `defaultValue` (String expression), `defaultFilter` (11 options), `placeholder` (text template), `adjustable` (boolean), `delay` (integer, default 500ms), `valueAttribute` (String/HashString), `onChange` action, and two accessibility text-template props.

**Behavioral constraints from this file:**
- `attrChoice` defaults to `"auto"`.
- Only `String` attribute types are accepted in the `attributes` list; the filter targets String columns exclusively.
- `valueAttribute` accepts both `String` and `HashString`, allowing the current filter value to be persisted in hashed form.
- `adjustable` defaults to `true`; when false, the comparison-type button is hidden and `screenReaderButtonCaption` is also hidden by the editor config.
- `delay` defaults to 500ms — the debounce period before the filter is applied after user input.
- `offlineCapable="true"` means the widget works in offline Mendix apps.
- Default filter value is `"contains"`.

**User-facing:** Yes — all properties here are configurable by the Mendix developer.

**New findings:** The `linkedDs` property is declared with `isLinked="true"` and `isList="true"`, which enables the filter to drive a list datasource directly. The `helpUrl` links to the Data Grid 2 documentation section on text filters.

---

## typings/DatagridTextFilterProps.d.ts

**Purpose:** Auto-generated TypeScript type definitions derived from `DatagridTextFilter.xml`. Provides compile-time types for all widget props.

**Logic:** Exports `AttrChoiceEnum` (`"auto" | "linked"`), `AttributesType` (object with a single `attribute: AttributeMetaData<string>`), `DefaultFilterEnum` (union of 11 string literals), and the two runtime/preview interface types. `DatagridTextFilterContainerProps` is the full runtime contract; `DatagridTextFilterPreviewProps` is used only in editor preview contexts.

**Behavioral constraints from this file:**
- `defaultValue` is `DynamicValue<string> | undefined` — it is optional and dynamic (can be an expression result).
- `valueAttribute` is `EditableValue<string> | undefined` — optional two-way binding for persisting the filter value.
- `onChange` is `ActionValue | undefined` — triggered when the filter value or type changes.
- `tabIndex` and `style` are standard widget platform props, not domain-specific.
- Preview props expose `delay` as `number | null` (nullable in preview) vs always `number` at runtime.

**User-facing:** Internal — enforces type safety across the widget codebase.

**New findings:** `DatagridTextFilterPreviewProps` includes `renderMode: "design" | "xray" | "structure"` and a deprecated `className` field (deprecated since v9.18.0 in favor of `class`), matching the same pattern seen in other Mendix widgets.

---

## typings/FilterType.d.ts

**Purpose:** Re-exports `DefaultFilterEnum` under the alias `FilterType` for external consumers that import from this barrel file.

**Logic:** A single re-export: `export { DefaultFilterEnum as FilterType } from "./DatagridTextFilterProps"`.

**Behavioral constraints from this file:** No independent behavior — acts purely as a public API alias.

**User-facing:** No — this is a TypeScript type-level export for downstream consumers.

**New findings:** The existence of this re-export alias suggests that `FilterType` is the intended public name for the filter function enum when this widget's typings are consumed externally (e.g., by other widgets or utilities that need to reference filter types without importing the full props file).

---

## src/components/TextFilterContainer.tsx

**Purpose:** The core rendering component. Wires together the `StringFilterController` (MobX store), external event listeners, and the `InputWithFilters` UI component.

**Logic:** Creates a stable UUID per mount for the `id`. Instantiates a `StringFilterController` via `useSetup` with the filter store, default filter function, adjustability flag, default value, change delay, and a rule that disables the text input when the selected filter is `"empty"` or `"notEmpty"`. Binds `valueAttribute` and `onChange` through `useBasicSync`. Subscribes to `reset.value` and `set.value` external events via `useOnResetValueEvent` / `useOnSetValueEvent`. Renders `InputWithFilters` with all controller outputs and display props.

**Behavioral constraints from this file:**
- When `adjustable` is `false`, the filter-type button is hidden but the controller still respects `defaultFilter`; the user cannot change the comparison operator.
- Selecting `"empty"` or `"notEmpty"` disables the text input field — these operators require no text input.
- External `reset.value` event clears the input and resets to the `defaultValue` (if present) or to empty.
- External `set.value` event updates the input to the provided `stringValue`.
- `useBasicSync` syncs `valueAttribute` writes and fires `onChange` after each debounced filter change.
- The delay from `props.delay` (default 500ms) is the debounce window before the filter is applied and `onChange` fires.

**User-facing:** Yes — this component renders the visible text input and filter-type dropdown.

**New findings:** The component is wrapped in MobX `observer`, so the UI updates reactively when the `StringFilterController`'s state changes. The `parentChannelName` is passed down for proper scoping of external events when multiple filter widgets share a parent channel.

---

## src/components/typings.ts

**Purpose:** Defines the internal `StringFilterProps` interface shared between the container component and the HOCs.

**Logic:** Exports `StringFilterProps` with two fields: `filterStore: String_InputFilterInterface` (the MobX filter store) and `parentChannelName?: string` (optional channel name for external event scoping).

**Behavioral constraints from this file:** No runtime behavior — acts as a shared type contract between HOCs and the rendering component.

**User-facing:** No — internal interface only.

**New findings:** The use of `String_InputFilterInterface` (from `@mendix/widget-plugin-filtering`) shows that the filter store is abstracted behind an interface, allowing different store implementations (e.g., `StringInputFilterStore` in tests, `StringStoreProvider` in linked mode) to be injected polymorphically.

---

## src/DatagridTextFilter.editorConfig.ts

**Purpose:** Configures the widget's property panel behavior in Mendix Studio Pro and renders the structure-mode preview.

**Logic:** `getProperties` hides `screenReaderButtonCaption` when `adjustable` is false, hides the `attributes` list when `attrChoice === "auto"`, and always hides the `linkedDs` internal datasource property. `getPreview` builds a `RowLayout` structure-preview: a bordered container that optionally includes a filter-type icon button (when `adjustable`) and a placeholder text area. `getSvgContent` maps each `DefaultFilterEnum` value to its light/dark SVG icon from `@mendix/widget-plugin-filtering`.

**Behavioral constraints from this file:**
- When `adjustable === false`, the comparison button and its screen-reader caption are hidden in both the property panel and the structure preview.
- When `attrChoice === "auto"`, the `attributes` list is hidden (no manual attribute configuration needed).
- The structure preview faithfully reflects the adjustable/non-adjustable state with or without the icon button.
- All 11 filter types have distinct SVG icons for both light and dark modes.

**User-facing:** Editor-only — affects property panel UX and structure preview; not visible to end users at runtime.

**New findings:** The `linkedDs` property is always hidden from the panel regardless of `attrChoice` — it is managed internally by the widget framework and not exposed directly to developers.

---

## src/DatagridTextFilter.editorPreview.tsx

**Purpose:** Provides a live React-rendered preview of the widget inside Mendix Studio Pro's design canvas.

**Logic:** Calls `enableStaticRendering(true)` (MobX SSR mode) since the preview runs in a non-browser context. Creates two `InputStore` instances via `useMemo` — one initialized with `defaultValue`, one empty (used for range filtering support). Renders `InputWithFiltersComponent` directly (not the observer-wrapped runtime variant) with `filterFnList={[]}` (no dropdown in preview), a no-op `onFilterChange`, and the prop values from the preview props.

**Behavioral constraints from this file:**
- The preview renders the default filter function fixed to whatever `defaultFilter` is set in the property panel — the dropdown is not interactive.
- The placeholder text shown in the preview comes directly from the `placeholder` property.
- `parseStyle` converts the style string (from Studio Pro) into a CSSProperties object.

**User-facing:** Editor-only — visible only in Studio Pro's design view; not rendered at runtime.

**New findings:** The use of two `InputStore` instances mirrors the runtime component's support for range filtering (where two inputs are needed), even though the text filter only uses one input in practice. This future-proofs the preview for potential range mode additions.

---

## src/hocs/withLinkedStringStore.tsx

**Purpose:** HOC that creates a `StringStoreProvider` when the widget is in `"linked"` (Custom) mode, providing the `filterStore` to the wrapped component.

**Logic:** Expects a `filterAPI: FilterAPI` prop (injected by `withFilterAPI`). Creates a `StringStoreProvider` via `useSetup`, passing the `filterAPI`, the mapped `attributes` array (extracted from `props.attributes`), and `props.name` as the `dataKey`. Returns the wrapped component with `filterStore` and `parentChannelName` injected.

**Behavioral constraints from this file:**
- The `dataKey` is the widget's `name` prop — this is the unique identifier used to register the filter within the parent context.
- `attributes` are mapped to `AttributeMetaData<string>` objects; only String-typed attributes are accepted (enforced by the XML definition).
- The `filterAPI.parentChannelName` is forwarded for external event scoping.

**User-facing:** No — this is an invisible integration layer.

**New findings:** `StringStoreProvider` (from `@mendix/widget-plugin-filtering/custom-filter-api`) registers the filter store with the parent datasource context so that the parent widget (e.g., Data Grid 2) can use the filter value when querying data. This is the mechanism by which the widget drives server-side filtering.

---

## src/hocs/withParentProvidedStringStore.tsx

**Purpose:** HOC for `"auto"` mode — retrieves an existing `StringInputFilterStore` from the parent context (injected by Data Grid 2 or similar) and passes it to the wrapped component.

**Logic:** `withParentProvidedStringStore` calls `useStringFilterAPI()` which reads the filter context via `useFilterAPI()`. If the context is missing, it returns an error. If the provider has an error, or if the store is not of type `"direct"`, it returns `EMISSINGSTORE`. If the store type is not `"input"` or the arg1 type is not a string filter, it returns `EStoreTypeMisMatch`. On success, it memoizes `{ filterStore, parentChannelName }` in a ref and returns it. Errors are rendered as a red `Alert` component.

**Behavioral constraints from this file:**
- In `"auto"` mode, the widget depends entirely on a parent widget providing a `StringInputFilterStore` via the filter context; if no parent provides the store, a danger alert is shown.
- Type mismatches (e.g., placing a text filter where a number filter is expected) surface as visible error messages.
- The result is memoized in a ref to avoid re-creating the store binding on every render.

**User-facing:** Partially — error states (missing store, type mismatch) are rendered as visible `Alert` messages.

**New findings:** `EMISSINGSTORE` and `EStoreTypeMisMatch` are typed error constants from `@mendix/widget-plugin-filtering/errors`, providing structured error information for the type mismatch error message shown to the developer in the running app.

---

## src/utils/widget-utils.ts

**Purpose:** Utility function to check whether the widget should delay rendering while the default value is loading.

**Logic:** Exports `isLoadingDefaultValues(props)` which returns `true` if `props.defaultValue?.status === "loading"`.

**Behavioral constraints from this file:**
- The widget renders nothing (via `withPreloader`) until `defaultValue.status` is no longer `"loading"`.
- If `defaultValue` is `undefined` (not configured), the function returns `false` and the widget renders immediately.

**User-facing:** No — internal gating logic for the preloader HOC.

**New findings:** This utility cleanly isolates the loading-state check, making it testable independently and avoiding direct Mendix API coupling in the main component.

---

## e2e/DataGridTextFilter.spec.js

**Purpose:** End-to-end Playwright tests that verify the widget's filtering behavior, default-value initialization, and accessibility compliance in a running Mendix application.

**Logic:** Three test groups: (1) visual snapshot against a known baseline PNG; (2) text filtering — verifies that typing in the filter input narrows rows correctly (e.g., typing "test3" leaves 2 rows with expected cell values) and that data context is passed correctly to a linked text box; (3) default value — navigates to `/p/filter_init_condition` and confirms the filter is pre-applied on load (1 result matching "Betty"); (4) accessibility — runs an axe-core scan against WCAG 2.1 AA rules and expects zero violations.

**Behavioral constraints from this file:**
- The filter is applied client-side (or server-side via the store) as the user types — confirmed by row-count changes.
- When a filter is active and a row is clicked, the selected object's data is available as context for other widgets (e2e-confirmed: clicking a filtered row sets an `AgeTextBox` value).
- The `defaultValue` prop applies the filter immediately on page load, before the user types anything — confirmed by the `filter_init_condition` page test.
- Each test session is cleaned up via `window.mx.session.logout()` to avoid exceeding Mendix's license session limit of 5.
- The widget passes WCAG 2.1 AA accessibility checks (zero axe violations) excluding the navigation tree widget.

**User-facing:** Yes — these tests exercise the complete user-visible filtering workflow.

**New findings:** The e2e test uses `.mx-name-datagrid1` as the selector, confirming the widget is embedded within a standard Data Grid 2 component. The data context passing test (AgeTextBox) confirms that row selection works correctly in the presence of an active text filter.

---

## src/components/__tests__/DatagridTextFilter.spec.tsx

**Purpose:** Unit tests covering the widget's rendering, value synchronization, external event handling, and filter-operator selection.

**Logic:** Sets up a mock `FilterAPI` context with a `StringInputFilterStore`. Tests cover: defaultValue-only-used-as-initial-value (changes after mount are ignored); external `reset.value` event resets to `defaultValue` or empty depending on the event flag; external `set.value` event updates the input to the provided `stringValue`; typing triggers `valueAttribute.setValue` and `onChange.execute` after the debounce delay; selecting `"Empty"` operator disables the text input; multiple instances get unique `aria-controls` IDs; missing context renders a snapshot error message; HashString attributes are accepted alongside String.

**Behavioral constraints from this file:**
- `defaultValue` is used only as an initial value: changes to it after the first render do not update the displayed value (confirmed: `undefined → string`, `string → string`, `string → undefined` all leave the input at its initial value).
- `reset.value` event with `true` flag resets to `defaultValue`; with `false` flag resets to empty and calls `setValue(undefined)`.
- `set.value` event with `{ stringValue: "..." }` sets the input to the provided string and calls `setValue`.
- When `"Empty"` or `"Not empty"` operator is selected, the text input is disabled and focus remains within the filter controls.
- Multiple instances of the widget on the same page get unique `aria-controls` IDs (uses UUID generation).
- When no filter context is present, the widget renders an error state (snapshot-tested).
- Both `String` and `HashString` attribute types work together in a single filter store.

**User-facing:** No — internal test coverage.

**New findings:** The test confirms that `delay: 1000` (1000ms debounce) is used in tests via `jest.runOnlyPendingTimers()`. The `CHANNEL_NAME = "datagrid/1"` format reveals how the parent channel is namespaced in the filter context. `FilterAPI.version: 3` is the current context API version in use.

---

## CHANGELOG.md

### 1. What versions are documented?
Versions from 2.0.0 (2021-09-28) through 3.10.0 (2026-05-06), covering 18 releases.

### 2. What categories of changes are most common?
Primarily bug fixes (filter reset issues, default value sync, onChange not firing, focus management) and internal improvements. A few breaking changes and feature additions.

### 3. What behavioral constraints can be derived from the changelog?
- **v2.0.4 (breaking):** `defaultValue` is now used only as initial value — changes after mount are ignored (v2.4.0).
- **v2.3.0:** Added `empty` and `notEmpty` filter operators (previously missing).
- **v2.0.2:** Added `valueAttribute` (store filter value in attribute) and `onChange` event.
- **v2.5.0:** External events subscription added (Reset_Filter / Set_Filter actions).
- **v2.6.0:** `Set_Filter` action subscription added; `Reset_Filter` updated to support reset-to-default-value.
- **v2.7.0 / v2.8.0 (breaking):** Group key added then removed — replaced by parent widget integration improvements.
- **v2.8.4:** Fixed `onChange` not firing for `empty`/`notEmpty` operators; improved accessibility for filter type select button (enter/space/arrow keys trigger the menu); improved screen reader integration.
- **v2.9.1:** Non-adjustable default filters were being overridden by personalization — fixed.
- **v3.8.1:** Filter selector dropdown now auto-selects best placement based on available space.
- **v3.10.0:** Fixed dropdown placement on small viewports; fixed focus jumping away from filter controls when Empty/Not empty is selected.

### 4. Is this file user-facing?
No — developer-facing release notes.

### 5. What new did you learn?
The widget had a significant behavioral breaking change in v2.4.0 (defaultValue becomes initial-only), which is prominently documented and is also tested in the unit tests. The `empty`/`notEmpty` operators were added in v2.3.0, explaining why they have special handling (disabled input) in the controller. The v2.8.0 removal of "Group key" (added just one version prior in v2.7.0) indicates a short-lived feature that was superseded by a better integration approach.
