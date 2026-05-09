# checkbox-radio-selection-web — Source Extraction Draft

Widget: `@mendix/checkbox-radio-selection-web` v1.1.1  
Package: `com.mendix.widget.web.CheckboxRadioSelection.mpk`  
Min Mendix: 10.7.0, reactReady: true, appNumber: -1 (not yet published to Marketplace)

---

## src/CheckboxRadioSelection.tsx

**1. Purpose:** Root container component — entry point for the Mendix runtime. Initializes the selector and routes rendering to `RadioSelection` or `CheckboxSelection` based on selector type.

**2. Logic:**
- Calls `useGetSelector(props)` to get the appropriate selector instance
- Builds `commonProps` shared by both child components (tabIndex, inputId, labelId, readOnlyStyle, ariaRequired, groupName, ariaLabel, noOptionsText)
- Three render paths:
  1. `selector.status === "unavailable"` → `<Placeholder>` with noOptionsText
  2. `selector.type === "single"` → `<RadioSelection selector={selector} ...commonProps/>`
  3. else (multi) → `<CheckboxSelection selector={selector} ...commonProps/>`
- `noOptionsText` falls back to hardcoded `"No options available"` if not configured

**3. Behavioral documentation:** The widget renders either a group of radio buttons or checkboxes depending on the data source and configuration. When data is unavailable (loading or error), a placeholder message is shown. The "single" vs "multi" distinction determines radio vs checkbox rendering.

**4. User-facing?** Yes — container component.

**5. New learnings:** `selector.status === "unavailable"` is a tristate distinct from Mendix's `ValueStatus` — it's a widget-internal state computed by the selector. The `Placeholder` component is shown for data unavailability, not loading.

---

## src/CheckboxRadioSelection.xml

**1. Purpose:** Widget descriptor with extensive property tree supporting 3 data sources and 3 option types.

**2. Logic:** Key properties:
- `source` (context/database/static) — determines the data backend
- `optionsSourceType` (association/enumeration/boolean) — for context source only
- `controlType` (checkbox/radio) — only surfaced for boolean type
- `attributeEnumeration` / `attributeBoolean` — `EditableValue` for context enum/boolean
- `attributeAssociation` — Reference or ReferenceSet association
- `optionsSourceAssociationDataSource` / `optionsSourceDatabaseDataSource` — `ListValue` for options
- `optionsSourceDatabaseItemSelection` — `SelectionSingleValue | SelectionMultiValue` (drives single vs multi for database source)
- `optionsSourceDatabaseValueAttribute` — `ListAttributeValue<string|Big>` for value-to-store
- `databaseAttributeString` — `EditableValue<string|Big>` for storing selected database value
- `staticAttribute` — writable attribute for static source (any type)
- `optionsSourceStaticDataSource` — list of objects with `value`, `caption`, `customContent`
- `optionsSourceCustomContentType` (yes/no) — enables per-option widget drop zones
- `readOnlyStyle` (bordered/text) — control vs content-only when read-only
- `customEditability` (default/never/conditionally) — database-only editability override
- Two onChange actions: `onChangeEvent` (context/static) and `onChangeDatabaseEvent` (database)
- Accessibility: `ariaRequired`, `ariaLabel`
- `groupName` — expression for the `name` attribute on inputs (groups related inputs)
- `noOptionsText` — translated text (en_US/nl_NL)

**3. Behavioral documentation:** The XML reveals the complexity: database source has different onChange (selection-based), database with `databaseAttributeString` behaves like association/enum (single string stored), while database without it uses Mendix's Selection API. Custom content slots per option available for association, database, and static sources.

**4. User-facing?** Yes — Studio Pro property panel.

**5. New learnings:** `offlineCapable=true` — works in offline apps. `appNumber: -1` in package.json signals this is a new widget not yet published. The database source has two modes: (1) store selected object's attribute to a string attribute, (2) use Mendix's Selection API. These have different property sets and on-change events.

---

## typings/CheckboxRadioSelectionProps.d.ts

**1. Purpose:** Auto-generated TypeScript types from the XML.

**2. Logic:** Confirms the full property type surface:
- `OptionsSourceStaticDataSourceType` has `staticDataSourceValue: DynamicValue<string | Big | boolean | Date>` — static source supports any primitive type
- `attributeAssociation: ReferenceValue | ReferenceSetValue` — union type, single vs multi association
- `optionsSourceDatabaseItemSelection?: SelectionSingleValue | SelectionMultiValue` — optional, undefined when not database source
- `databaseAttributeString?: EditableValue<string | Big>` — optional, when used enables attribute-mode for database source
- `staticAttribute: EditableValue<string | Big | boolean | Date>` — writable, any type

**3. Behavioral documentation:** The type union `ReferenceValue | ReferenceSetValue` requires runtime type discrimination (done in `getSelector.ts`). Similarly `SelectionSingleValue | SelectionMultiValue` drives single vs multi selector for database source.

**4. User-facing?** No — TypeScript types.

**5. New learnings:** `Big` (from big.js) appears in attribute types for integer/long/decimal values — confirming arbitrary-precision numeric handling. `ListWidgetValue` for custom content per option (association and database only — static uses `ReactNode` directly).

---

## src/helpers/types.ts

**1. Purpose:** Core TypeScript interfaces for the selector abstraction layer.

**2. Logic:**
- `Status`: `"unavailable" | "loading" | "available"` — widget-internal state
- `SelectionType`: `"single" | "multi"`
- `CaptionsProvider`: `get(value)` for string captions, `render(value)` for ReactNode (supports custom content)
- `OptionsProvider`: generic interface for options list with search, lazy loading (`hasMore`, `loadMore`), datasource filter, and internal `_updateProps`/`_optionToValue`/`_valueToOption` methods
- `SelectorBase`: abstract interface for all selectors — `updateProps()`, `status`, `type`, `readOnly`, `validation`, `options`, `caption`, `clearable`, `currentId`, `setValue()`, `customContentType`
- `SingleSelector`: extends SelectorBase with `type: "single"`, `currentId: string | null`, `controlType: "checkbox" | "radio"`
- `MultiSelector`: extends SelectorBase with `type: "multi"`, `currentId: string[] | null`, `getOptions()`
- `SelectionBaseProps`: props shared between RadioSelection and CheckboxSelection

**3. Behavioral documentation:** The selector abstraction decouples the UI components from data source details. Both RadioSelection and CheckboxSelection program against these interfaces, not concrete classes. `controlType` on `SingleSelector` enables the "single checkbox for boolean" special case.

**4. User-facing?** No — TypeScript types.

**5. New learnings:** `MultiSelector.getOptions()` is distinct from `SingleSelector.options.getAll()` — multi uses a different method for listing options. `CaptionsProvider.render()` takes `string | null | number | null` and returns `ReactNode`, enabling custom widget content rendering via a unified interface.

---

## src/helpers/getSelector.ts

**1. Purpose:** Factory function — creates the appropriate selector class based on widget configuration.

**2. Logic:**
```
source === "context":
  optionsSourceType in ["enumeration", "boolean"] → EnumBooleanSingleSelector
  optionsSourceType === "association":
    attributeAssociation.type === "Reference" → AssociationSingleSelector
    else → AssociationMultiSelector

source === "database":
  optionsSourceDatabaseItemSelection?.type === "Multi" → DatabaseMultiSelector
  else → DatabaseSingleSelector

source === "static" → StaticSingleSelector
```

**3. Behavioral documentation:** Selector type is determined at widget initialization and never changes (selector is cached in a ref by `useGetSelector`). For association source, `ReferenceValue` maps to single selector, `ReferenceSetValue` maps to multi. Database single vs multi is driven by the Selection API type.

**4. User-facing?** No — internal factory.

**5. New learnings:** There are 6 distinct selector implementations. The factory covers all combinations at initialization; if the association type changes (which Mendix doesn't allow at runtime), the widget would not re-initialize. Note: `StaticSingleSelector` is always single — there's no `StaticMultiSelector`.

---

## src/components/CheckboxSelection/CheckboxSelection.tsx

**1. Purpose:** Multi-select UI component — renders a list of checkboxes.

**2. Logic:**
- Gets all options from `selector.getOptions()` and current selection from `selector.currentId` (string[])
- `handleChange(optionId, checked)`: builds new selection array by adding or removing the toggled optionId, calls `selector.setValue(newSelection)`
- Guards against readonly: `if (!isReadOnly)` before calling setValue
- Each checkbox gets `id={checkboxId}` = `${inputId}-checkbox-${index}`
- `aria-describedby` and `aria-invalid` only set when `isSingleCheckbox && validation` (single-checkbox case with validation)
- `CaptionContent onClick` prevents event propagation and calls `handleChange` directly — allows clicking the label area to toggle
- Shows `<Placeholder>` when options array is empty
- Shows `<ValidationAlert>` when `selector.validation` is set

**3. Behavioral documentation:** Renders as a list of checkboxes with associated labels. Clicking the label area works the same as clicking the checkbox itself. In read-only mode, checkboxes are rendered as `disabled` inputs (style depends on `readOnlyStyle`). Validation error message shown below the group.

**4. User-facing?** Yes — rendered UI.

**5. New learnings:** `CaptionContent` has a click handler that stops propagation (`e.stopPropagation()` + `e.nativeEvent.stopImmediatePropagation()`) — this is to prevent bubbling to parent containers that might react to clicks (e.g., in-form click handling). The `aria-describedby`/`aria-invalid` only applies to single-checkbox mode, not multi-checkbox groups.

---

## src/components/RadioSelection/RadioSelection.tsx

**1. Purpose:** Single-select UI component — renders radio buttons or a single boolean checkbox.

**2. Logic:**
- `asSingleCheckbox = selector.controlType === "checkbox"` — boolean type with `controlType: "checkbox"` renders as one checkbox
- For single-checkbox mode: finds the "true" option in `allOptions`, or falls back to `allOptions[0]`; renders only that one option
- `handleChange`: if `asSingleCheckbox`, toggles "true"/"false" string; else sets to the radio's value
- In read-only mode with `readOnlyStyle === "text"`: non-selected options are not rendered at all (returns `null`)
- The input `type` attribute is `selector.controlType` — either "radio" or "checkbox"
- Caption area click handler stops propagation and directly calls `selector.setValue()`
- Shows `<Placeholder>` when options array is empty

**3. Behavioral documentation:** Handles two visually distinct modes under one component: (1) radio buttons for enum/association single select, (2) single checkbox for boolean type. In "text" read-only mode, only the currently selected option is shown without control elements. Clicking label area triggers selection same as clicking the input.

**4. User-facing?** Yes — rendered UI.

**5. New learnings:** The boolean-as-single-checkbox is implemented by detecting `controlType === "checkbox"` on a `SingleSelector`. The value stored is the string `"true"` or `"false"` (not native boolean) — consistent with the `EditableValue<string>` abstraction used by `EnumBooleanSingleSelector`. The `<If>` component from `widget-plugin-component-kit` is used to conditionally render the input element in read-only text mode.

---

## src/hooks/useGetSelector.ts

**1. Purpose:** React hook — manages the lifecycle of the selector instance.

**2. Logic:**
- Uses `useRef<Selector>` to persist the selector across re-renders
- Initializes selector once via `getSelector(props)` on first render
- Subscribes to `onAfterSearchTermChange` to force re-render when search results change (via `setInput({})` state update)
- Calls `selector.updateProps(props)` on every render to sync latest props into the cached selector instance

**3. Behavioral documentation:** The selector is a stable object that gets props injected each render rather than being recreated. Search term changes trigger a forced re-render to update the displayed options list. This pattern avoids re-running the factory function on every render.

**4. User-facing?** No — React hook.

**5. New learnings:** The `setInput({})` trick (updating state with a new empty object reference) forces a re-render without carrying any actual state — it's a pure re-render trigger. This is only needed for search functionality. The `useRef` + `updateProps` pattern is the standard Mendix widget pattern for stateful selectors.

---

## src/hooks/useWrapperProps.ts

**1. Purpose:** Hook that computes ARIA and CSS props for the selection group wrapper div.

**2. Logic:**
- `role: isCheckbox ? "group" : "radiogroup"` — WCAG-required roles for checkbox groups vs radio groups
- `aria-labelledby: hasLabel ? ${inputId}-label : undefined` — links to Mendix-generated label if present
- `aria-required: ariaRequired?.value` — passes through configured required state
- `aria-label: hasAriaLabel ? ariaLabel?.value : undefined` — explicit aria-label if no Mendix label
- CSS classes: `widget-checkbox-radio-selection-list`, with conditional `widget-checkbox-radio-selection-readonly` and `widget-checkbox-radio-selection-readonly-{readOnlyStyle}` when read-only
- `getInputLabel(inputId)` — DOM query to find a label element for the input

**3. Behavioral documentation:** The wrapper div gets ARIA roles to make the group accessible as a whole. `aria-labelledby` points to the Mendix-generated label element. When read-only, the CSS class suffix (`bordered` or `text`) drives the visual style.

**4. User-facing?** No — hook.

**5. New learnings:** `getInputLabel(inputId)` performs a DOM query — this is how the widget detects whether a Mendix label has been configured for the widget. If no label is found, `aria-labelledby` is omitted and only `aria-label` (if configured) applies. Both label mechanisms are mutually exclusive in practice.

---

## src/CheckboxRadioSelection.editorConfig.ts

**1. Purpose:** Studio Pro property panel configuration — complex conditional property hiding.

**2. Logic:**
- Source-specific hiding: each source (context/database/static) hides the properties of the other two
- `optionsSourceType === "boolean"` is the only case where `controlType` is shown
- Database source has extra conditional logic:
  - Hides `optionsSourceDatabaseValueAttribute` and `databaseAttributeString` for Multi selection
  - Hides `Editability` when `databaseAttributeString` is empty (Selection API mode — editability not applicable)
  - Hides `customEditabilityExpression` unless `customEditability === "conditionally"`
- `optionsSourceCustomContentType === "no"` hides custom content widgets for all sources
- Static source: hides `staticDataSourceCustomContent` per item when custom content is off
- `getPreview()` renders structure preview with checkbox/radio icon and caption, or dropzone(s) for custom content
- `getIconPreview(isMultiSelect)`: renders checkbox SVG for multi/boolean-checkbox, radio SVG otherwise

**3. Behavioral documentation:** The property panel adapts heavily based on source selection — most properties are hidden when irrelevant. Custom content dropzones are shown in the preview when enabled. Database source in Selection API mode (no `databaseAttributeString`) hides the standard `Editability` system property.

**4. User-facing?** No — Studio Pro editor.

**5. New learnings:** The distinction between database-attribute-mode and database-selection-mode changes which system properties (`Editability`, `customEditability`) are shown. This is an advanced pattern where the widget bypasses Mendix's standard editability system for the selection-mode case.

---

## src/CheckboxRadioSelection.editorPreview.tsx

**1. Purpose:** Studio Pro canvas preview — renders a working preview using preview selectors.

**2. Logic:**
- Creates selector based on source: `StaticPreviewSelector`, `DatabasePreviewSelector` / `DatabaseMultiPreviewSelector`, or `AssociationPreviewSelector`
- `useMemo` on `[props]` — recreates selector when props change
- Renders same `RadioSelection`/`CheckboxSelection` components as runtime, but with preview selectors
- Uses `dynamic(false)` / `dynamic("")` from `@mendix/widget-plugin-test-utils` for stable preview ARIA props

**3. Behavioral documentation:** Canvas preview shows realistic radio/checkbox rendering based on the current property configuration. Preview selectors return static/mock data rather than live Mendix data.

**4. User-facing?** No — Studio Pro canvas.

**5. New learnings:** Preview selectors are separate classes (`StaticPreviewSelector`, `DatabasePreviewSelector`, etc.) that implement the same selector interfaces but with static data. The `dynamic()` helper from test-utils creates stable `DynamicValue` mocks for use in preview context.

---

## src/__tests__/CheckboxRadioSelection.spec.tsx

**1. Purpose:** Unit tests for the root container component.

**2. Logic:** Mocks `getSelector` to return a static single selector with 3 options. 3 basic tests:
1. Renders without crashing — checks for `.widget-checkbox-radio-selection` class
2. Renders when selector status is unavailable (test description promises more than it delivers — no status override)
3. Applies correct CSS class

**3. Behavioral documentation:** Minimal tests — only verify that the component renders and has the correct CSS class. No behavior testing (selection changes, read-only mode, etc.).

**4. User-facing?** No — test file.

**5. New learnings:** Test coverage is very thin for this widget's complexity. The mock injects a single-selector with `controlType: "radio"` and `status: "available"` — only the happy-path single-select case is tested at the unit level. The second test is misleading — it doesn't actually test unavailable status.

---

## e2e/SelectionControls.spec.js

**1. Purpose:** Playwright e2e tests — visual screenshot tests for all 5 data source variants and basic interaction tests.

**2. Logic:**
- Navigates to `/p/checkboxradioselection`, clicks `actionButton1` to load data, waits for `networkidle`
- 5 screenshot tests: association (#1), enum (#2), boolean (#3), static (#4, Page 2 tab), database (#5, Page 2 tab)
- 2 interaction tests: radio button click (database widget), checkbox click (association widget)
- Session logout after each test to avoid Mendix 5-session license limit

**3. Behavioral documentation:** Tests verify visual appearance against stored screenshots for all source types. Interaction tests verify that clicking inputs marks them as checked. Database and static widget tests require navigating to "Page 2" tab.

**4. User-facing?** No — test file.

**5. New learnings:** 5 snapshot images are stored in the repo (association, enum, boolean, static, database). The test project has two pages: Page 1 for association/enum/boolean, Page 2 for static/database. `networkidle` wait + explicit button click suggests the page requires user interaction to load entity context data.

---

## Helper Files Summary

**helpers/utils.ts**: Utility functions — `_valuesIsEqual()` (compares Big/boolean/Date/string), `getCustomCaption()` (editor caption generation), `getValidationErrorId()` (id for validation alert), `getInputLabel()` (DOM query for label element).

**helpers/BaseOptionsProvider.ts**: Abstract base class implementing `OptionsProvider` — manages options array, search term, callback registration. Default: `hasMore=false`, `isLoading=false`, `status="available"`.

**helpers/EnumBool/EnumBoolOptionsProvider.ts**: Handles enum/boolean `EditableValue.universe` as options list. Converts `"true"`/`"false"` strings ↔ boolean values.

**helpers/Database/DatabaseOptionsProvider.ts**: Maps `ListValue` ObjectItems to string keys (item IDs). Rebuilds map on each `_updateProps` call.

**helpers/Association/AssociationOptionsProvider.ts**: Same pattern as DatabaseOptionsProvider but for association datasources.

**helpers/Static/StaticOptionsProvider.ts**: Maps static config entries to string index keys (`"0"`, `"1"`, ...). Reverse lookup finds index for a given static option object.

---

## Summary

**Widget:** checkbox-radio-selection-web v1.1.1 — a new widget (released 2025-08-25, not yet Marketplace-published).

**Purpose:** Renders checkbox or radio button selection backed by one of 3 data sources: Context (association/enum/boolean), Database (ListValue + Selection API or attribute), or Static (configured list).

**Architecture:**

```
CheckboxRadioSelection (container)
  └─ useGetSelector → getSelector() → one of 6 selector classes
  └─ CheckboxSelection (multi: association-ReferenceSet, database-Multi)
  └─ RadioSelection   (single: association-Reference, enum, boolean, database-Single, static)
  └─ Placeholder      (when status unavailable)
```

**6 selector classes:**
| Selector | Source | Type |
|----------|--------|------|
| EnumBooleanSingleSelector | context enum/boolean | single |
| AssociationSingleSelector | context Reference | single |
| AssociationMultiSelector | context ReferenceSet | multi |
| DatabaseSingleSelector | database, Single selection | single |
| DatabaseMultiSelector | database, Multi selection | multi |
| StaticSingleSelector | static | single |

**Key behaviors:**
- Boolean type: renders as single checkbox (if `controlType=checkbox`) or two radio buttons (true/false)
- Read-only style "text": hides non-selected options entirely (only shows current value as text)
- Read-only style "bordered": renders disabled controls
- Custom content: per-option widget drop zones available for association, database, and static sources
- Database source has two modes: Selection API (no `databaseAttributeString`) or attribute store (with `databaseAttributeString`)
- Validation alerts shown below the group; `aria-invalid`/`aria-describedby` for single-input cases
- ARIA: `role="radiogroup"` (radio), `role="group"` (checkbox), `aria-labelledby` links to Mendix label

**Tests:** Thin unit tests (3 render checks), 5 e2e visual snapshots + 2 interaction tests.

---

## CHANGELOG.md

**1. Purpose:** Tracks notable changes across all released versions of the widget.

**2. Logic:** Three versions released: v1.0.0 (2025-08-25 initial release), v1.1.0 (2025-10-18 added validation alert and aria-label), v1.1.1 (2026-02-24 fixed association selection and long label display).

**3. Behavioral documentation:** v1.1.1 introduces a behavioral constraint fix: when the reference set backing an association contains objects that are not present in the configured selectable objects list, the widget now correctly skips (ignores) those association objects rather than attempting to render them. This means the displayed selection always reflects only options that are valid according to the current selectable objects list. The same release fixed a visual defect where long label text was not displaying correctly. v1.1.0 added two user-facing capabilities: a validation alert (shown below the group when validation fails) and the `aria-label` property (allowing explicit accessible name override when no Mendix label is present). These are reflected in the existing `CheckboxSelection.tsx` (shows `<ValidationAlert>`) and `useWrapperProps.ts` (`aria-label` passed through) sections. v1.0.0 was the initial release on 2025-08-25 — confirming the widget is new and not yet Marketplace-published (`appNumber: -1`).

**4. User-facing?** No — developer/operator changelog. The behaviors documented are user-facing, but the file itself targets developers.

**5. New learnings:** The v1.1.1 fix reveals a meaningful behavioral constraint: the AssociationMultiSelector must reconcile the current `currentId` (reference set objects) against the options list (selectable objects) and silently drop any IDs not present in the options — ensuring the UI never displays a selected state for an object that isn't a valid choice. Long label fix implies the CSS for label text was width-constrained and has been corrected.
