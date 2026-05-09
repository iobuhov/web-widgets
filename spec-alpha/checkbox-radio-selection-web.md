# CheckboxRadioSelection

## Purpose

The CheckboxRadioSelection widget renders a group of checkboxes or radio buttons backed by one of three data sources: Context (association, enumeration, or boolean attribute), Database (list value with Mendix Selection API or attribute store), or Static (a developer-configured list of options). It is used when a user must choose one option (radio/single-checkbox) or multiple options (checkboxes) from a bounded set, with the rendering style determined by the data source and attribute type.

## User Scenarios

### [P1] Single selection via radio buttons (enumeration or association)

**Given** the widget is configured with a context source using an enumeration or Reference association attribute  
**When** the user clicks a radio button or its label  
**Then** the selected attribute value is updated to that option; previously selected option is deselected; clicking the label area has the same effect as clicking the radio input

#### Edge Cases

- If no options are available (e.g., the data source yields an empty list), a placeholder message is shown instead of the radio group. The placeholder text defaults to "No options available" if not configured.
- In read-only mode with `readOnlyStyle = "text"`, only the currently selected option is rendered — all other options are hidden entirely.
- In read-only mode with `readOnlyStyle = "bordered"`, all options are rendered as disabled inputs.

### [P2] Multi selection via checkboxes (association ReferenceSet or database Multi)

**Given** the widget is configured with a context source using a ReferenceSet association, or a database source with Multi selection  
**When** the user clicks a checkbox or its label  
**Then** the selected set is updated: checked options are added, unchecked options are removed; the change is committed immediately

#### Edge Cases

- If the ReferenceSet contains objects not present in the current selectable objects list, those objects are silently ignored — only valid options are displayed as selected.
- Clicking the label area triggers the same toggle as clicking the checkbox input (event propagation is stopped to prevent bubbling to parent containers).

### [P3] Boolean attribute rendered as single checkbox or two radio buttons

**Given** the widget is configured with a context source using a boolean attribute  
**When** `controlType = "checkbox"`, the widget renders a single checkbox; when `controlType = "radio"`, two radio buttons (true/false) are rendered  
**Then** toggling the checkbox or selecting a radio button writes the corresponding boolean value to the attribute; the stored value is the string `"true"` or `"false"` (not a native boolean)

#### Edge Cases

- In single-checkbox mode with a validation error, `aria-describedby` and `aria-invalid` are set on the input for accessibility; in multi-checkbox mode these attributes are not set on individual inputs.

### [P4] Custom content per option

**Given** the widget is configured with `optionsSourceCustomContentType = "yes"` for association, database, or static source  
**When** the user views the option list  
**Then** each option renders a widget drop zone (custom content) instead of a plain text caption

#### Edge Cases

- Custom content is not available for static source items individually; the static source uses the `optionsSourceStaticDataSource` list with per-item custom content widgets when enabled.

### [P5] Database source — Selection API mode

**Given** the widget is configured with a database source and no `databaseAttributeString` is set  
**When** the user selects an option  
**Then** the selection is managed through Mendix's `SelectionSingleValue` or `SelectionMultiValue` API; the `onChangeDatabaseEvent` action fires

### [P6] Database source — attribute store mode

**Given** the widget is configured with a database source and `databaseAttributeString` is set  
**When** the user selects an option  
**Then** the selected option's value attribute is written to `databaseAttributeString`; the `onChangeEvent` action fires; the standard `Editability` property controls whether the widget is editable

## Functional Requirements

- FR-001: The widget MUST render radio buttons when the selector type is "single" (enumeration, Reference association, boolean with `controlType=radio`, database Single, or static).
- FR-002: The widget MUST render checkboxes when the selector type is "multi" (ReferenceSet association or database Multi).
- FR-003: The widget MUST render a boolean attribute as a single checkbox when `controlType = "checkbox"`, or as two radio buttons (true/false) when `controlType = "radio"`.
- FR-004: The widget MUST show a placeholder message when the data source status is "unavailable"; the placeholder text defaults to "No options available" if not configured.
- FR-005: The widget MUST apply `role="radiogroup"` to the wrapper div for radio button groups, and `role="group"` for checkbox groups.
- FR-006: The widget MUST link to a Mendix-generated label via `aria-labelledby` when a label element is present in the DOM; if absent, `aria-label` from configuration is used instead. Both are mutually exclusive.
- FR-007: The widget MUST render disabled inputs (not hide them) in read-only mode with `readOnlyStyle = "bordered"`.
- FR-008: The widget MUST hide non-selected options in read-only mode with `readOnlyStyle = "text"`, showing only the current value.
- FR-009: Clicking a label area MUST trigger the same selection change as clicking the input directly; event propagation MUST be stopped to prevent interference with parent container click handlers.
- FR-010: The widget MUST show a `<ValidationAlert>` below the group when `selector.validation` is set.
- FR-011: For ReferenceSet associations, the widget MUST silently ignore any current selection values that are not present in the current selectable objects list.
- FR-012: The database source in Selection API mode MUST hide the standard `Editability` system property in Studio Pro; in attribute store mode, `Editability` applies normally.
- FR-013: The widget MUST support offline apps (`offlineCapable = true`).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `source` | enum: `context \| database \| static` | — | Data source | Determines the backend for options |
| `optionsSourceType` | enum: `association \| enumeration \| boolean` | — | Type | Context source sub-type (context only) |
| `controlType` | enum: `checkbox \| radio` | — | Control type | Boolean type only: renders as single checkbox or two radios |
| `attributeBoolean` | `EditableValue<boolean>` | — | Attribute | Context boolean attribute |
| `attributeEnumeration` | `EditableValue<string>` | — | Attribute | Context enumeration attribute |
| `attributeAssociation` | `ReferenceValue \| ReferenceSetValue` | — | Association | Context association (Reference = single, ReferenceSet = multi) |
| `optionsSourceAssociationDataSource` | `ListValue` | — | Data source | Options list for association source |
| `optionsSourceDatabaseDataSource` | `ListValue` | — | Data source | Options list for database source |
| `optionsSourceDatabaseItemSelection` | `SelectionSingleValue \| SelectionMultiValue` | — | Selection | Selection API for database source |
| `optionsSourceDatabaseValueAttribute` | `ListAttributeValue<string \| Big>` | — | Value attribute | Value attribute on each database list item |
| `databaseAttributeString` | `EditableValue<string \| Big>` | — | Store attribute | Attribute to store selected database value (attribute store mode) |
| `staticAttribute` | `EditableValue<string \| Big \| boolean \| Date>` | — | Attribute | Writable attribute for static source |
| `optionsSourceStaticDataSource` | object list | — | Options | Static list of options with value, caption, customContent |
| `optionsSourceCustomContentType` | enum: `yes \| no` | `no` | Custom content | Enables per-option widget drop zones |
| `readOnlyStyle` | enum: `bordered \| text` | `bordered` | Read-only style | How the widget renders in read-only mode |
| `customEditability` | enum: `default \| never \| conditionally` | `default` | Editability | Database source editability override |
| `onChangeEvent` | `ActionValue` | — | On change | Action for context/static source change |
| `onChangeDatabaseEvent` | `ActionValue` | — | On change | Action for database source change |
| `ariaRequired` | `DynamicValue<boolean>` | — | Required | Sets `aria-required` on the group wrapper |
| `ariaLabel` | `DynamicValue<string>` | — | Accessible label | Explicit ARIA label when no Mendix label is present |
| `groupName` | `ExpressionValue<string>` | — | Group name | `name` attribute on all inputs; groups related controls |
| `noOptionsText` | `DynamicValue<string>` | `"No options available"` | No options text | Placeholder shown when options list is empty |

## Changelog

**v1.1.1 (2026-02-24)**
- Fixed: ReferenceSet selections now skip objects not present in the selectable objects list (previously caused rendering errors).
- Fixed: Long label text now displays correctly.

**v1.1.0 (2025-10-18)**
- Added validation alert shown below the selection group on validation failure.
- Added `ariaLabel` property to support explicit ARIA label when no Mendix label is present.

**v1.0.0 (2025-08-25)**
- Initial release. Supports context (association/enum/boolean), database, and static data sources. Supports offline apps.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] What happens when the `attributeAssociation` type changes at runtime (Reference ↔ ReferenceSet)? The selector is initialized once and not re-created — this scenario may be impossible in Mendix but is not explicitly documented.
- [ ] Is there a maximum number of options that can be rendered before performance degrades? The `hasMore`/`loadMore` protocol exists on `OptionsProvider` but no UI pagination is visible in the draft.
- [ ] The `StaticSingleSelector` is always single — is a static multi-select intentionally not supported?
- [ ] Unit test coverage is very thin (3 render-only tests). Are integration or accessibility tests planned beyond the 5 existing e2e visual snapshots?
