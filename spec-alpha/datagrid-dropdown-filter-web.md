# DatagridDropdownFilter

## Purpose

The DatagridDropdownFilter widget provides a configurable dropdown filter for Mendix Data Grid 2 and Gallery widgets. It supports two orthogonal filtering modes: attribute-based filtering (Enum and Boolean attribute types) and association-based filtering (reference entity selection). Within each mode the widget renders one of three UI variants — a plain select dropdown, a text-searchable combobox, or a multi-value tag picker — determined automatically by the `filterable` and `multiSelect` property combination. The widget integrates with the Mendix filter channel architecture, enabling coordination with other filter widgets on the same page via JS actions (`Set_Filter`, `Reset_Filter`).

---

## User Scenarios

### [P1] Filter a data grid by enum attribute (simple dropdown)
**Given** the widget is placed inside a Data Grid 2 column header with `baseType = "attr"`, `filterable = false`, and a bound enum or Boolean attribute  
**When** the user opens the dropdown and selects an option  
**Then** the data grid immediately filters to rows matching the selected value; multi-select (OR logic) is available when `multiSelect = true`

#### Edge Cases
- When `multiSelect = false`, selecting a new option replaces the previous selection.
- When `multiSelect = true`, options are presented with checkboxes; all selected values are applied as an OR filter.
- Selecting the empty option ("None", configured via `emptyOptionCaption`) clears the filter and shows all rows.
- When `defaultValue` is configured, the filter initializes with that value on page load before any user interaction.

### [P2] Filter by association (reference entity selection)
**Given** the widget is placed with `baseType = "ref"`, `refEntity` and `refOptions` datasource configured  
**When** the user opens the dropdown and selects a reference object  
**Then** the data grid filters to rows associated with the selected entity; the empty option ("None") is always displayed first in the list

#### Edge Cases
- Association mode always requires explicit `refEntity` and `refOptions` configuration — there is no "auto" path for associations.
- When `fetchOptionsLazy = true`, options load on scroll (100 px from bottom); personalization value restoration is limited in lazy mode.
- The selected option text is shown in the toggle button after selection.
- Multiple selections persist: checked options remain checked after the dropdown is closed and reopened.

### [P3] Searchable filterable mode (combobox / tag picker)
**Given** `filterable = true` and `multiSelect = false`  
**When** the user types in the text input  
**Then** the option list narrows to matching options (combobox mode); selecting one option applies the filter

**Given** `filterable = true` and `multiSelect = true`  
**When** the user types and selects options  
**Then** selected values appear as tags in the input (tag picker mode); all selected tags apply as an OR filter

#### Edge Cases
- In filterable mode, `clearable` is implicitly `true` and cannot be disabled.
- `emptyOptionCaption` is hidden in filterable mode (no "None" option in the filtered list).
- `filterInputPlaceholderCaption` is shown as the input placeholder only when `filterable = true`.
- `refSearchAttr` is required for association filterable mode; it determines the attribute used for server-side text search.
- `selectionMethod` (`"checkbox"` | `"rowClick"`) and `selectedItemsStyle` (`"text"` | `"boxes"`) are only relevant in filterable + multiSelect mode and `selectedItemsStyle = "boxes"`.

### [P4] Default value initialization
**Given** `defaultValue` is configured with a Mendix expression  
**When** the page loads  
**Then** the widget defers rendering until the expression resolves (`withPreloader` guard), then initializes the filter with the resolved value

#### Edge Cases
- `defaultValue` is used as an **initial value only**. Changes to the expression after widget mount are ignored (breaking change introduced in v2.4.0).
- Multi-select default: comma-separated string values (e.g., `"Blue,Cyan,Red"`) are supported for enum multi-select initialization.
- The loading guard applies only to `defaultValue`; association `refOptions` may render before options are fully available.

### [P5] JS action integration (Set_Filter / Reset_Filter)
**Given** a Mendix nanoflow or JS action calls `Set_Filter` or `Reset_Filter` for this widget  
**When** the action is executed  
**Then** `Set_Filter` applies the specified filter value; `Reset_Filter` clears the selection or restores the configured `defaultValue` when reset-to-default is requested

#### Edge Cases
- JS action events are scoped to `parentChannelName` when filter channel coordination is configured.

### [P6] Attribute guard — non-filterable attribute
**Given** the widget is bound to an attribute that does not have filtering enabled in the Mendix data model  
**When** the page renders  
**Then** the widget displays a `danger` alert instead of the dropdown, informing the developer that filtering is not enabled for the selected attribute

#### Edge Cases
- This guard applies to both "linked" attribute mode and association mode, but NOT to "auto" mode (the parent column handles its own guard in that case).

---

## Functional Requirements

- FR-001: The widget MUST support two base filtering modes, selected by `baseType`: `"attr"` (Enum and Boolean attributes only) and `"ref"` (association/reference entities).
- FR-002: The widget MUST automatically select the UI frontend type based on `filterable` and `multiSelect`: `filterable=false` → Select; `filterable=true, multiSelect=false` → Combobox; `filterable=true, multiSelect=true` → TagPicker.
- FR-003: In `"auto"` attribute mode the widget MUST read the filter store from the parent Data Grid column context. If the parent does not provide a `"direct"` / `"select"` store, the widget MUST render an error alert.
- FR-004: In `"linked"` attribute mode the widget MUST construct its own `EnumStoreProvider` using the explicitly configured `linkedDs` and `attr`, keyed by the widget's `name`.
- FR-005: In association mode the widget MUST always use the "linked" path, constructing a `RefFilterStore` from `refEntity` and `refOptions`. If these are missing at runtime the widget MUST throw a runtime error.
- FR-006: The widget MUST block rendering until `defaultValue` resolves from `"loading"` state (`withPreloader` HOC).
- FR-007: `defaultValue` MUST be applied as the initial filter value only; subsequent expression changes after mount MUST be ignored.
- FR-008: Custom `filterOptions` MUST be validated against the enum store on mount. Any invalid option value MUST cause the widget to render an error alert instead of the dropdown.
- FR-009: In association mode, scroll-triggered lazy loading MUST fire when the user scrolls within 100 px of the bottom of the option list. In `fetchOptionsLazy = true` mode, options are deferred until scroll or focus.
- FR-010: When `filterable = true`, `clearable` MUST be treated as `true` regardless of the configured value, and the `emptyOptionCaption` property MUST be hidden.
- FR-011: `Set_Filter` and `Reset_Filter` JS actions MUST be supported in all three frontend types (Select, Combobox, TagPicker).
- FR-012: The widget MUST support offline Mendix applications (`offlineCapable = true`).
- FR-013: The widget MUST pass WCAG 2.1 AA accessibility requirements (confirmed by automated axe-core scan with zero violations).
- FR-014: Multi-select enum filtering MUST apply an OR logical condition across all selected values.
- FR-015: Filter selector dropdown placement MUST automatically select the best position based on available viewport space (fixed since v3.10.0).

---

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `baseType` | Enum (`attr` \| `ref`) | — | Filter type | Selects attribute-based or association-based filtering mode. Determines the entire downstream rendering path. |
| `attrChoice` | Enum (`auto` \| `linked`) | — | Attribute source | `"auto"`: reads store from parent column context. `"linked"`: uses explicit `linkedDs` and `attr`. Only relevant when `baseType = "attr"`. |
| `attr` | AttributeMetaData (Enum \| Boolean) | — | Attribute | The filterable attribute. Only Enum and Boolean types are supported. Shown only in `"linked"` mode. |
| `linkedDs` | ListValue | — | Data source | Datasource for linked attribute mode. Hidden in `"auto"` mode. |
| `auto` | Boolean | — | Auto options | When `true`, options are derived automatically from the enum/boolean universe. When `false`, `filterOptions` must be provided. |
| `filterOptions` | List | — | Filter options | Custom caption/value pairs for the dropdown. Validated against enum store on mount. Hidden when `auto = true`. |
| `refEntity` | EntityRef | — | Entity | The reference entity to filter on. Association mode only. |
| `refOptions` | ListValue | — | Options datasource | Datasource for association options. Association mode only. |
| `refCaptionSource` | Enum (`attr` \| `exp`) | — | Caption source | `"attr"`: caption from a Mendix attribute; `"exp"`: caption from an expression. Determines search attribute behavior. |
| `refCaption` | AttributeMetaData | — | Caption attribute | Used as both display caption and search attribute when `refCaptionSource = "attr"`. |
| `refSearchAttr` | AttributeMetaData | — | Search attribute | Attribute used for server-side text search. Required when `filterable = true` and `refCaptionSource = "exp"`. Hidden otherwise. |
| `fetchOptionsLazy` | Boolean | `false` | Use lazy load | Defers loading of reference options until scroll or focus. Association mode only. Note: personalization restoration is limited in lazy mode. |
| `defaultValue` | DynamicValue (String) | — | Default value | Initial filter value expression. Applied once on mount; ignored after mount. |
| `emptyOptionCaption` | DynamicValue (String) | — | Empty option caption | Label for the "None" (no filter) option. Hidden when `filterable = true` or `multiSelect = true`. |
| `emptySelectionCaption` | DynamicValue (String) | — | Empty selection caption | Placeholder shown in the toggle button when no selection is made. |
| `filterInputPlaceholderCaption` | DynamicValue (String) | — | Input placeholder | Text shown in the filter input field. Only relevant when `filterable = true`. |
| `filterable` | Boolean | `false` | Filterable | Enables text-input filtering of options (combobox / tag picker mode). |
| `multiSelect` | Boolean | `false` | Multi-select | Allows selecting multiple values. Requires `filterable = true` to unlock `selectionMethod` and `selectedItemsStyle`. |
| `clearable` | Boolean | `true` | Clearable | Allows clearing the selection. Hidden (and implicitly `true`) when `filterable = true`. |
| `selectedItemsStyle` | Enum (`text` \| `boxes`) | — | Selected items style | `"text"`: comma-separated; `"boxes"`: tag chips. Only shown when `filterable = true` and `multiSelect = true`. |
| `selectionMethod` | Enum (`checkbox` \| `rowClick`) | — | Selection method | How options are selected in tag picker mode. Only shown when `selectedItemsStyle = "boxes"`. |
| `valueAttribute` | EditableValue (String) | — | Saved attribute | Stores the current filter selection as a string for personalization/persistence. String type only; associations not supported. |
| `onChange` | ActionValue | — | On change | Action executed when the filter selection changes. |
| `ariaLabel` | DynamicValue (String) | — | ARIA label | Accessible label for the dropdown. If not set, screen readers receive an empty label. |

---

## Changelog

- **v3.10.0 (2026-05-06):** Fixed filter selector dropdown placement on small viewports — now selects best position based on available viewport space.
- **v3.0.4 (2025-08-05) [BREAKING]:** Split text configuration into three distinct properties: `emptyOptionCaption` (the "None" list item), `emptySelectionCaption` (toggle button placeholder), and `filterInputPlaceholderCaption` (input placeholder). Prior configurations must be reconfigured. Also added enhanced keyboard navigation.
- **v3.0.3 (2025-07-24):** Fixed `fetchOptionsLazy` flag not being read correctly — the widget was always loading lazily in association mode regardless of the setting.

---

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] When `ariaLabel` is not configured, the combobox receives an empty `aria-label`. Is there a intended fallback label, or is configuration required for accessibility compliance?
- [ ] Does `filterOptions` support dynamic (expression-driven) values, or only static configured values? The `withCustomOptionsGuard` validates on mount only (not reactive), so dynamic changes after mount would not be re-validated.
- [ ] In association multi-select mode with `fetchOptionsLazy = true`, what is the behavior when personalization restores a previously selected value that has not yet been loaded? The draft notes this is "limited" but does not specify the exact behavior.
- [ ] The `check` function in `editorConfig.ts` returns no static validation errors. Is runtime-only validation (via HOC guards) intentional for all constraint enforcement, or are design-time checks planned?
