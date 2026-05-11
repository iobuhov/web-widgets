## ADDED Requirements

### Requirement: Attribute selection via dropdown
The widget SHALL render a read-only input field that opens a dropdown list of configurable sort attributes. When the user selects an attribute, the widget SHALL notify the linked Gallery datasource to sort by that attribute and SHALL display the selected attribute's caption in the input field.

#### Scenario: Opening the dropdown
- **WHEN** the user clicks the input field
- **THEN** a dropdown list SHALL appear showing all configured sort attributes

#### Scenario: Selecting an attribute
- **WHEN** the user clicks a list item in the open dropdown
- **THEN** the dropdown SHALL close, the selected attribute's caption SHALL appear in the input field, and the Gallery datasource SHALL be sorted by that attribute

#### Scenario: Already-selected attribute is visually marked
- **WHEN** the dropdown is open and an attribute is currently selected
- **THEN** the corresponding list item SHALL have the `filter-selected` CSS class applied

#### Scenario: Empty/placeholder state
- **WHEN** no attribute is selected
- **THEN** the input field SHALL display the configured `emptyOptionCaption` as placeholder text

### Requirement: Sort direction toggle
The widget SHALL render a button adjacent to the input field that toggles the sort direction between ascending (`asc`) and descending (`desc`). The button SHALL reflect the current direction via its CSS class.

#### Scenario: Ascending direction indicator
- **WHEN** the current sort direction is `asc`
- **THEN** the direction button SHALL have the CSS class `icon-asc` applied

#### Scenario: Descending direction indicator
- **WHEN** the current sort direction is `desc`
- **THEN** the direction button SHALL have the CSS class `icon-desc` applied

#### Scenario: Toggling direction
- **WHEN** the user clicks the direction button
- **THEN** the sort direction SHALL toggle to the opposite value

### Requirement: Supported attribute types
The widget SHALL accept sort attributes of the following Mendix attribute types: `AutoNumber`, `Decimal`, `Integer`, `Long`, `String`, `DateTime`, `Boolean`, `Enum`.

#### Scenario: String attribute sort
- **WHEN** a `String` attribute is configured and selected by the user
- **THEN** the Gallery datasource SHALL sort by that string attribute

#### Scenario: Numeric attribute sort
- **WHEN** a `Decimal`, `Integer`, `Long`, or `AutoNumber` attribute is configured and selected
- **THEN** the Gallery datasource SHALL sort by that numeric attribute

### Requirement: Portal-rendered dropdown menu
The widget SHALL render the dropdown list by appending it to `document.body` via a React portal. The dropdown SHALL be positioned immediately below the input field, left-aligned to it, and SHALL match the input field's width.

#### Scenario: Dropdown attaches to document.body
- **WHEN** the dropdown is open
- **THEN** the dropdown `<ul>` element SHALL be a direct child of `document.body`

#### Scenario: Dropdown width matches input
- **WHEN** the dropdown is open
- **THEN** the dropdown list width SHALL equal the pixel width of the input field

#### Scenario: Dropdown positioned below input
- **WHEN** the dropdown is open
- **THEN** the dropdown top edge SHALL align with the bottom edge of the input field (`position: fixed`, `top: inputBottom`, `left: inputLeft`)

### Requirement: Click-outside dismissal
The widget SHALL close the open dropdown when the user clicks anywhere outside the dropdown container or the dropdown list.

#### Scenario: Click outside closes dropdown
- **WHEN** the dropdown is open and the user clicks outside both the input wrapper and the dropdown list
- **THEN** the dropdown SHALL close

#### Scenario: Click inside does not close dropdown
- **WHEN** the dropdown is open and the user clicks on a list item
- **THEN** the dropdown closes only after selection (not before)

### Requirement: Keyboard navigation — input field
The input field SHALL support keyboard interaction for opening and closing the dropdown.

#### Scenario: Enter key opens dropdown
- **WHEN** the input field has focus and the user presses `Enter`
- **THEN** the dropdown SHALL open (or close if already open)

#### Scenario: Space key opens dropdown
- **WHEN** the input field has focus and the user presses `Space`
- **THEN** the dropdown SHALL open (or close if already open)

#### Scenario: Focus on open — selected item focused first
- **WHEN** the dropdown opens
- **THEN** focus SHALL move to the currently selected list item, or to the first list item if none is selected

### Requirement: Keyboard navigation — dropdown list items
Each list item in the open dropdown SHALL support keyboard interaction for selection and navigation.

#### Scenario: Enter key selects item
- **WHEN** a list item has focus and the user presses `Enter`
- **THEN** that attribute SHALL be selected and the dropdown SHALL close

#### Scenario: Space key selects item
- **WHEN** a list item has focus and the user presses `Space`
- **THEN** that attribute SHALL be selected and the dropdown SHALL close

#### Scenario: Escape key closes dropdown
- **WHEN** a list item has focus and the user presses `Escape`
- **THEN** the dropdown SHALL close and focus SHALL return to the input field

#### Scenario: Shift+Tab from first item closes dropdown
- **WHEN** the first list item has focus and the user presses `Shift+Tab`
- **THEN** the dropdown SHALL close and focus SHALL return to the input field

#### Scenario: Tab from last item closes dropdown and moves to button
- **WHEN** the last list item has focus and the user presses `Tab`
- **THEN** the dropdown SHALL close and focus SHALL move to the direction toggle button

### Requirement: Accessibility — ARIA attributes
The widget SHALL provide ARIA attributes on its interactive elements to support assistive technologies.

#### Scenario: Input aria-haspopup
- **WHEN** the widget renders
- **THEN** the input element SHALL have `aria-haspopup` set to `true`

#### Scenario: Input aria-expanded reflects dropdown state
- **WHEN** the dropdown is closed
- **THEN** the input element SHALL have `aria-expanded="false"`
- **WHEN** the dropdown is open
- **THEN** the input element SHALL have `aria-expanded="true"`

#### Scenario: Input aria-controls links to dropdown list
- **WHEN** the widget renders
- **THEN** the input element SHALL have `aria-controls` pointing to the `id` of the dropdown `<ul>` element

#### Scenario: Input aria-label from configuration
- **WHEN** `screenReaderInputCaption` is configured
- **THEN** the input element SHALL have `aria-label` set to that value

#### Scenario: Direction button aria-label from configuration
- **WHEN** `screenReaderButtonCaption` is configured
- **THEN** the direction toggle button SHALL have `aria-label` set to that value

#### Scenario: Dropdown list role
- **WHEN** the dropdown is open
- **THEN** the `<ul>` element SHALL have `role="menu"`

#### Scenario: List item role
- **WHEN** the dropdown is open
- **THEN** each `<li>` element SHALL have `role="menuitem"` and `tabIndex={0}`
