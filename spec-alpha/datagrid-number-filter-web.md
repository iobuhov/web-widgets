# DatagridNumberFilter

## Purpose

The DatagridNumberFilter widget provides a numeric filter control for Mendix Data Grid 2 and Gallery widgets. It allows end users to filter rows by a numeric attribute using one of eight comparison operators (greater than, greater than or equal, equal, not equal, smaller than, smaller than or equal, empty, not empty). The widget supports two attribute source modes — automatic (store provided by the parent column context) and linked (explicit attribute configuration) — and integrates with the Mendix filter channel architecture via JS actions (`Set_Filter`, `Reset_Filter`). All numeric values are handled internally as arbitrary-precision decimals using `big.js`. Supported attribute types: AutoNumber, Decimal, Integer, and Long.

---

## User Scenarios

### [P1] Type a number to filter the data grid
**Given** the widget is placed inside a Data Grid 2 column header with a numeric attribute  
**When** the user types a number into the filter input  
**Then** the data grid updates to show only rows where the attribute value satisfies the selected comparison operator, after a debounce delay (default 500 ms)

#### Edge Cases
- Input is debounced: the filter is not applied until the user pauses typing for `delay` milliseconds (default 500 ms; configurable).
- Locale affects number parsing: decimal separators differ by locale — a number valid in one locale may not parse in another (fixed in v2.9.2).
- Typing a non-numeric value does not crash the widget but may result in no filter being applied.

### [P2] Select a comparison operator
**Given** `adjustable = true` (default)  
**When** the user clicks the filter type button and selects an operator  
**Then** the selected operator is applied to the current input value and the data grid filters accordingly

**Available operators:** Greater than, Greater than or equal, Equal (default), Not equal, Smaller than, Smaller than or equal, Empty, Not empty

#### Edge Cases
- When `adjustable = false`, the filter type button is hidden; only the `defaultFilter` operator is used, and it is protected from personalization overrides.
- The filter type button opens on Enter, Space, or arrow keys (keyboard-accessible).
- The button's `aria-label` reflects the currently selected filter function name (e.g., "Equal").

### [P3] Empty / Not empty filter (input disabled)
**Given** the user selects "Empty" or "Not empty" as the comparison operator  
**When** the operator is active  
**Then** the numeric input field is disabled (not merely hidden), as no numeric value is needed for these comparisons; the data grid filters to rows where the attribute is null or non-null respectively

#### Edge Cases
- `onChange` fires when the "empty" or "not empty" operator is selected, even though no input value is provided (fixed in v2.8.4).
- Keyboard focus does not jump away from the filter controls when "Empty" or "Not empty" is selected (fixed in v3.10.0).

### [P4] Default value initialization
**Given** `defaultValue` is configured with a Mendix expression  
**When** the page loads  
**Then** the widget defers rendering until the expression resolves from `"loading"` state, then initializes the filter input with the resolved value

#### Edge Cases
- `defaultValue` is applied as an **initial value only**. Changing the expression value after widget mount does NOT update the filter — confirmed by unit tests covering both `undefined → value` and `value → undefined` transitions (breaking change introduced in v2.4.0).
- `defaultFilter` sets the initial comparison operator; this is the operator applied on page load when a `defaultValue` is present.

### [P5] Value persistence (Saved attribute)
**Given** `valueAttribute` is configured (EditableValue of type AutoNumber, Decimal, Integer, or Long)  
**When** the user changes the filter value or operator  
**Then** `valueAttribute.setValue()` is called with the current filter value as a `Big` decimal, and the `onChange` action fires

#### Edge Cases
- Both `valueAttribute.setValue()` and `onChange.execute()` are called once per change, after the debounce delay completes.
- On reset with no default: `attribute.setValue(undefined)` is called, clearing the persisted value.
- On reset with default: `attribute.setValue(Big(defaultValue))` is called, restoring the default.

### [P6] JS action integration (Set_Filter / Reset_Filter)
**Given** a Mendix nanoflow or JS action targets this widget  
**When** `Set_Filter` is executed with `{ numberValue: Big(N) }`  
**Then** the filter input updates to the specified value and `valueAttribute.setValue()` is called

**When** `Reset_Filter` is executed  
**Then** the filter input is cleared (reset to default value if the reset flag is `true`, or empty if `false`), and `valueAttribute.setValue()` is called accordingly

#### Edge Cases
- External events are scoped by the widget's Mendix `name` prop (not `parentChannelName`).
- `parentChannelName` enables cross-widget coordination in filter group scenarios.

### [P7] Auto mode — store provided by parent column
**Given** `attrChoice = "auto"` and the widget is placed inside a Data Grid 2 column or Gallery header  
**When** the page renders  
**Then** the widget reads the `Number_InputFilterInterface` store from the parent's React context and renders without constructing its own store

#### Edge Cases
- If placed outside a supported parent widget, the widget renders: "The filter widget must be placed inside the column or header of the Data grid 2.0 or inside header of the Gallery widget."
- If the parent provides a store of the wrong type (e.g., a text filter store), a type mismatch error alert is displayed.

---

## Functional Requirements

- FR-001: The widget MUST support two attribute source modes: `"auto"` (store injected by parent column via React context) and `"linked"` (store constructed from explicitly configured `attributes` list).
- FR-002: In `"linked"` mode the widget MUST apply `withAttributeGuard` to verify the configured attribute supports filtering, and `withFilterAPI` to connect to the filter context before constructing the store.
- FR-003: The widget MUST block rendering until `defaultValue` resolves from `"loading"` state (`withPreloader` / `isLoadingDefaultValues` guard).
- FR-004: `defaultValue` MUST be applied as the initial filter value only. Subsequent changes to the expression after mount MUST be ignored.
- FR-005: The widget MUST support eight comparison operators: `greater`, `greaterEqual`, `equal` (default), `notEqual`, `smaller`, `smallerEqual`, `empty`, `notEmpty`.
- FR-006: When the active operator is `"empty"` or `"notEmpty"`, the numeric input MUST be disabled (not hidden).
- FR-007: Filter application MUST be debounced by `delay` milliseconds (default 500 ms) after the user stops typing.
- FR-008: When `adjustable = false`, the filter type selector button MUST be hidden and the `defaultFilter` operator MUST be applied unconditionally, protected from personalization overrides.
- FR-009: The widget MUST call `valueAttribute.setValue()` and `onChange.execute()` on each filter value change (after debounce), and on each `Set_Filter` / `Reset_Filter` event.
- FR-010: `Set_Filter` MUST update the filter input to the specified `numberValue`. `Reset_Filter` with flag `true` MUST restore `defaultValue`; with flag `false` MUST clear the input.
- FR-011: The filter type button MUST be keyboard-accessible: open on Enter, Space, and arrow keys.
- FR-012: All numeric values MUST be handled as `big.js` arbitrary-precision decimals (`Big`) internally, including `defaultValue`, `valueAttribute`, and JS action payloads.
- FR-013: Supported attribute types for `attributes` (linked mode) and `valueAttribute`: AutoNumber, Decimal, Integer, Long.
- FR-014: The widget MUST support offline Mendix applications (`offlineCapable = true`).
- FR-015: The widget MUST pass WCAG 2.1 AA accessibility requirements (confirmed by automated axe-core scan with zero violations).
- FR-016: The filter selector dropdown MUST automatically select the best placement position based on available viewport space using `position: fixed` strategy (fixed in v3.8.1 / v3.10.0).
- FR-017: Each widget instance MUST generate a stable, unique `aria-controls` ID (UUID-based, generated once per mount via ref) to prevent duplicate ARIA attribute conflicts across multiple instances.

---

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `attrChoice` | Enum (`auto` \| `linked`) | — | Attribute source | `"auto"`: reads store from parent column context. `"linked"`: uses explicit `attributes` list and `linkedDs`. |
| `attributes` | List (AttributeMetaData\<Big\>) | — | Attributes | Numeric attributes to filter on. Accepts AutoNumber, Decimal, Integer, Long. Relevant only in `"linked"` mode. |
| `linkedDs` | ListValue | — | Data source | Datasource for linked attribute mode. Hidden in `"auto"` mode. |
| `defaultFilter` | Enum | `"equal"` | Default filter | Initial comparison operator. Options: `greater`, `greaterEqual`, `equal`, `notEqual`, `smaller`, `smallerEqual`, `empty`, `notEmpty`. |
| `defaultValue` | DynamicValue (Decimal / Big) | — | Default value | Initial filter value expression. Applied once on mount; changes after mount are ignored. |
| `placeholder` | DynamicValue (String) | — | Placeholder | Input field placeholder text. |
| `adjustable` | Boolean | `true` | Adjustable | When `true`, the end user can switch the comparison operator at runtime. When `false`, `defaultFilter` is fixed and protected from personalization. |
| `delay` | Integer | `500` | Input delay (ms) | Debounce delay in milliseconds before the filter is applied after the user stops typing. |
| `valueAttribute` | EditableValue (AutoNumber \| Decimal \| Integer \| Long) | — | Saved attribute | Stores the current filter value for personalization/persistence. `setValue` is called on each change and on reset. |
| `onChange` | ActionValue | — | On change | Action executed when the filter value or operator changes. |
| `screenReaderButtonCaption` | DynamicValue (String) | — | Filter selector caption | Accessible label for the comparison operator button. Hidden when `adjustable = false`. |
| `screenReaderInputCaption` | DynamicValue (String) | `"Search"` (en) | Input caption | Accessible label for the numeric input element. Built-in translations: en_US "Search", de_DE "Suche", nl_NL "Zoeken". |

---

## Changelog

- **v3.10.0 (2026-05-06):** Fixed filter selector dropdown placement on small viewports. Fixed keyboard focus jumping away from filter controls when "Empty" or "Not empty" is selected.
- **v3.9.0 (2026-03-23):** Fixed crash when `valueAttribute` (Saved attribute) is configured.
- **v2.9.2 (2025-05-19):** Fixed number filter not working in some locales — number parsing is locale-sensitive.

---

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] When `attrChoice = "linked"` and multiple attributes are configured in the `attributes` list, how does the filter condition combine them? The `NumberStoreProvider` accepts an array, but the multi-attribute behavior is not exercised in the available unit tests.
- [ ] What is the exact behavior when a non-numeric string is entered in the input? The source confirms no crash, but whether the filter is cleared, ignored, or treated as zero is not specified.
- [ ] The `editorPreview.tsx` creates two `InputStore` instances per render — does this dual-store model serve any purpose at runtime beyond matching the `InputWithFiltersComponent` API? A second store's purpose (if any) for the number filter is not documented.
- [ ] Is locale-sensitive number parsing applied to `valueAttribute` on read (restoring persisted values), or only on user input? The v2.9.2 fix is documented for user input; persistence restoration behavior is unclear.
