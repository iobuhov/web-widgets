# DatagridTextFilter

## Purpose

The Datagrid Text Filter widget provides a configurable text input with an optional comparison-operator selector, enabling users to filter string columns in Data Grid 2 (or compatible grids) by substring, exact match, starts/ends-with, empty/not-empty, or other string predicates. It operates in two modes: **auto** (the widget wires itself into a parent Data Grid 2's filter context with no explicit datasource configuration) and **linked** (the developer configures an explicit datasource and attribute list for standalone or cross-widget filtering). The widget is offline-capable and supports WCAG 2.1 AA accessibility requirements.

---

## User Scenarios

### [P1] Filter a text column by substring while typing

**Given** a Data Grid 2 containing a text column, with a Datagrid Text Filter widget placed in that column's filter slot in `attrChoice = "auto"` mode  
**When** the user types a substring into the filter input (e.g., "test3")  
**Then** after the configured debounce delay (default 500 ms), the grid narrows its rows to only those whose column value contains the typed substring; rows not matching the substring are hidden

#### Edge Cases
- If the user clears the input, all rows are restored (equivalent to no active filter).
- If the debounce delay has not elapsed, rows are not yet filtered; the filter is applied only after the delay expires.
- Typing a new character resets the debounce timer.

---

### [P2] Select "Empty" or "Not empty" operator to filter without typing

**Given** an adjustable Datagrid Text Filter (`adjustable = true`)  
**When** the user opens the comparison-operator dropdown and selects "Empty" or "Not empty"  
**Then** the text input field is disabled (no text entry required); the grid immediately filters to rows where the column value is empty or non-empty respectively

#### Edge Cases
- The text input remains disabled until the user switches back to an operator that requires a value (e.g., "Contains").
- If `adjustable = false`, the operator dropdown is hidden and `defaultFilter` determines the fixed operator for the lifetime of the page.

---

### [P3] Developer configures a default value applied on page load

**Given** a Datagrid Text Filter with `defaultValue` set to an expression (e.g., `"Betty"`)  
**When** the page loads  
**Then** the filter input is pre-populated with the default value and the grid is filtered accordingly before the user interacts

#### Edge Cases
- The widget defers rendering until `defaultValue.status` is no longer `"loading"` (shows nothing during load).
- `defaultValue` is used only as an initial value; changes to the expression after mount do not update the displayed value or reapply the filter.
- An external `reset.value` event with flag `true` resets the input back to `defaultValue`; with flag `false` it clears the input and stores `undefined`.

---

### [P4] Persist filter value in a Mendix attribute

**Given** a `valueAttribute` (String or HashString) configured on the widget  
**When** the user changes the filter value  
**Then** the current filter string is written to the bound attribute after the debounce delay, allowing the value to be read by other widgets or stored across sessions

#### Edge Cases
- `HashString` type is accepted, enabling the filter value to be stored in hashed form.
- An external `set.value` event with a `stringValue` payload can programmatically update both the input and the bound attribute.

---

### [P5] Linked mode: filter drives an explicit datasource

**Given** `attrChoice = "linked"` with a `linkedDs` datasource and one or more `attributes` of type String  
**When** the user changes the filter value  
**Then** the linked datasource is re-queried using the current filter value and selected operator; only rows matching the filter are returned

#### Edge Cases
- If no parent provides a filter context and `attrChoice = "auto"`, a visible error alert is shown.
- Type mismatches (e.g., text filter placed on a number column) surface as a visible error message.

---

## Functional Requirements

- **FR-001:** The system MUST support two wiring modes: `"auto"` (reads filter store from parent Data Grid 2 context) and `"linked"` (manages its own filter store against an explicit datasource and attribute list).
- **FR-002:** The system MUST apply the filter value after a configurable debounce delay (default 500 ms, configured via `delay` property).
- **FR-003:** The system MUST support 11 comparison operators: `contains`, `startsWith`, `endsWith`, `greater`, `greaterEqual`, `equal`, `notEqual`, `smaller`, `smallerEqual`, `empty`, `notEmpty`.
- **FR-004:** The system MUST disable the text input field when the selected operator is `"empty"` or `"notEmpty"`, as those operators require no text input.
- **FR-005:** The system MUST use `defaultValue` only as an initial value; subsequent changes to the `defaultValue` expression after mount MUST NOT update the displayed or applied filter value.
- **FR-006:** The system MUST defer rendering until `defaultValue.status` is resolved (not `"loading"`).
- **FR-007:** The system MUST accept external `reset.value` events: flag `true` resets to `defaultValue`; flag `false` clears the input and calls `setValue(undefined)`.
- **FR-008:** The system MUST accept external `set.value` events with a `stringValue` payload and update the input accordingly.
- **FR-009:** The system MUST write the current filter value to `valueAttribute` (if configured) after each debounced change; `valueAttribute` MUST accept both `String` and `HashString` types.
- **FR-010:** The system MUST fire the `onChange` action after each debounced filter change (value or operator).
- **FR-011:** When `adjustable = false`, the operator selector button MUST be hidden and the operator MUST be fixed to `defaultFilter` for the lifetime of the widget.
- **FR-012:** Each widget instance MUST generate a unique `aria-controls` ID to support multiple filter instances on the same page.
- **FR-013:** The system MUST be operable in offline Mendix apps (`offlineCapable = true`).
- **FR-014:** In `"auto"` mode, if no parent provides a `StringInputFilterStore`, the widget MUST render a visible error alert.
- **FR-015:** The system MUST pass WCAG 2.1 AA accessibility requirements (zero axe-core violations).

---

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `attrChoice` | `"auto" \| "linked"` | `"auto"` | Attribute type | Selects wiring mode. `"auto"`: reads filter store from parent Data Grid 2 context. `"linked"`: developer configures explicit datasource and attributes. |
| `linkedDs` | Datasource (list, linked) | — | — | Internal datasource used in `"linked"` mode. Not exposed in the property panel; managed by the widget framework. |
| `attributes` | List of `AttributeMetaData<string>` | — | Attributes | List of String-typed column attributes targeted by this filter. Visible only when `attrChoice = "linked"`. |
| `defaultValue` | `DynamicValue<string> \| undefined` | — | Default value | Initial filter value (expression). Used only on first render; subsequent expression changes are ignored. Widget defers rendering until this value resolves. |
| `defaultFilter` | `DefaultFilterEnum` | `"contains"` | Default filter | Initial comparison operator. One of 11 operators. Fixed operator when `adjustable = false`. |
| `placeholder` | Text template | — | Placeholder | Placeholder text shown in the input when empty. |
| `adjustable` | Boolean | `true` | Adjustable | When `true`, the operator selector button is visible and the user can change the operator at runtime. When `false`, the operator is fixed to `defaultFilter`. |
| `delay` | Integer (ms) | `500` | Delay | Debounce period in milliseconds before the filter is applied after user input and before `onChange` fires. |
| `valueAttribute` | `EditableValue<string> \| undefined` | — | Value attribute | Optional two-way binding. Writes current filter string to attribute after each debounced change. Accepts `String` or `HashString`. |
| `onChange` | `ActionValue \| undefined` | — | On change | Mendix action fired after each debounced filter change (value or operator). |
| `screenReaderButtonCaption` | Text template | — | Screen reader button caption | Screen reader label for the operator selector button. Hidden from panel when `adjustable = false`. |
| `screenReaderInputCaption` | Text template | — | Screen reader input caption | Screen reader label for the text input field. |

### `DefaultFilterEnum` values

| Value | Operator |
|-------|----------|
| `contains` | Contains |
| `startsWith` | Starts with |
| `endsWith` | Ends with |
| `greater` | Greater than |
| `greaterEqual` | Greater than or equal |
| `equal` | Equal |
| `notEqual` | Not equal |
| `smaller` | Smaller than |
| `smallerEqual` | Smaller than or equal |
| `empty` | Empty |
| `notEmpty` | Not empty |

---

## Changelog

### 3.10.0 (2026-05-06)
- Fixed dropdown placement on small viewports.
- Fixed focus jumping away from filter controls when "Empty" or "Not empty" operator is selected.

### 3.8.1 (2026-02-19)
- `onChange` now fires correctly for `empty` and `notEmpty` operators.
- Improved keyboard accessibility for the filter-type select button (Enter/Space/arrow keys open the menu).
- Improved screen reader integration for filter controls.

### 2.9.1
- Fixed non-adjustable default filters being overridden by personalization.

---

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] What is the maximum supported value of `delay`? The XML defines it as an integer with a default of 500 ms but no documented upper bound.
- [ ] Is there a documented API contract for the `FilterAPI.version: 3` context? Breaking changes between versions would affect compatibility with other filter widgets.
- [ ] The `withPreloader` HOC suppresses all rendering until `defaultValue` resolves — is there an intentional timeout or user-visible loading indicator, or does the widget simply remain invisible?
