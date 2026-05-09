# DropdownSort

## Purpose

The Dropdown Sort widget provides a user-facing control for sorting a Gallery widget by a developer-configured set of attributes. It renders a read-only text input that opens a portal-based dropdown list of sort options, paired with a toggle button to switch between ascending and descending order. The widget must be placed inside a Gallery widget's header area, where it receives the Gallery's sort context via React context. It is intended for use in list or gallery views where end users need to interactively control sort order without exposing a form field or requiring a microflow.

## User Scenarios

### [P1] Select a sort attribute from the dropdown
**Given** the widget is placed in a Gallery header with at least one sortable attribute configured  
**When** the user clicks the text input to open the dropdown  
**Then** a portal-rendered list of sort options appears below the input, with the currently selected option highlighted; focus advances to the selected option (or the first option if none is selected) after a 10 ms delay

#### Edge Cases
- The dropdown list width matches the text input width, measured at render time.
- If `emptyOptionCaption` is not configured, the input shows "Select an attribute" as the placeholder (controller default).
- Clicking outside the dropdown closes it without changing the selection.

### [P2] Toggle sort direction
**Given** the user has selected a sort attribute  
**When** the user clicks the sort-direction toggle button  
**Then** the sort order switches between ascending and descending; the Gallery updates its data accordingly

#### Edge Cases
- Toggling direction when no attribute is selected updates the local direction observable but does NOT write to the Gallery's sort store.
- If the Gallery or another sort widget changes the sort order externally, the widget's direction state is synced via a MobX reaction.

### [P3] Use keyboard to navigate and select
**Given** the dropdown is open  
**When** the user presses Enter or Space on a focused menu item  
**Then** that option is selected and the dropdown closes, returning focus to the input

#### Edge Cases
- Tab on the last menu item moves focus to the sort direction button.
- Shift+Tab on the first menu item returns focus to the text input.
- Escape from any menu item returns focus to the text input and closes the dropdown.
- Selecting the empty option ("none") clears the current sort entirely (`setSortOrder()` called with no arguments).

### [P4] Widget placed outside Gallery context
**Given** the widget is placed on a page or inside a container that does not provide a Gallery sort context  
**When** the page renders  
**Then** the widget renders a danger alert with the exact message: "Error: widget is out of context. Please place the widget inside the Gallery header."

#### Edge Cases
- This error is non-fatal — the host page continues to render.
- No dropdown or sort controls are shown in this state.

### [P5] Multiple instances on the same page
**Given** two or more Gallery widgets each have a DropdownSort widget in their headers  
**When** the page renders  
**Then** each instance maintains independent sort state linked to its own Gallery, and each has a unique `aria-controls` attribute to avoid ARIA ID collisions

#### Edge Cases
- Each instance gets a UUID-based ID (`DropdownSort{uuid}`) generated at mount.
- Attributes from one Gallery's sort store do not affect another Gallery's sort store.

## Functional Requirements

- FR-001: The widget MUST require placement inside a Gallery widget header that provides a sort context via React context; if no context is present, the widget MUST render an error alert.
- FR-002: The widget MUST render a read-only text input that opens a portal-based dropdown list rendered into `document.body` to avoid z-index and overflow clipping issues.
- FR-003: The dropdown list MUST have `role="menu"`; each option MUST have `role="menuitem"`.
- FR-004: The widget MUST support full keyboard navigation: Enter/Space to select; Tab on last item → sort button; Shift+Tab on first item → input; Escape → input.
- FR-005: On dropdown open, focus MUST be moved to the selected option, or to the first option if none is selected, after a 10 ms delay.
- FR-006: The widget MUST generate a unique UUID-based ID per instance to ensure distinct `aria-controls`, `aria-label`, and related ARIA attributes across multiple instances on the same page.
- FR-007: The widget MUST support the following attribute types for sorting: AutoNumber, Decimal, Integer, Long, String, DateTime, Boolean, Enum.
- FR-008: Selecting the empty option MUST clear the Gallery sort order entirely.
- FR-009: Toggling the sort direction MUST update the Gallery's sort store only when an attribute is currently selected.
- FR-010: When the Gallery or another widget changes the sort order externally, the widget's local direction state MUST be synced via a MobX reaction.
- FR-011: The widget MUST support dynamic attribute lists — if the `attributes` prop changes at runtime, the available dropdown options MUST update without remounting.
- FR-012: The widget MUST accept custom `screenReaderButtonCaption` and `screenReaderInputCaption` props, rendered as `aria-label` on the direction button and the text input respectively.
- FR-013: The widget MUST pass WCAG 2.1 AA accessibility audit (as confirmed by axe-core).
- FR-014: The widget MUST support offline Mendix applications (`offlineCapable="true"`).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `linkedDs` | Datasource (linked) | — | Gallery data source | The datasource exposed by the parent Gallery widget. Required; must be placed inside Gallery header. |
| `attributes` | List | — | Sort attributes | List of sortable attributes. Each entry defines one sort option shown in the dropdown. |
| `attributes.attribute` | AttributeMetaData | — | Attribute | Attribute to sort by. Accepted types: AutoNumber, Decimal, Integer, Long, String, DateTime, Boolean, Enum. Metadata-only — not bound to runtime values. |
| `attributes.caption` | String | — | Caption | Display label for this sort option shown in the dropdown list. |
| `emptyOptionCaption` | DynamicValue\<string\> | — | Empty option caption | Placeholder text shown in the input when no attribute is selected. Defaults to "Select an attribute" if not configured. Optional. |
| `screenReaderButtonCaption` | DynamicValue\<string\> | — | Sort button caption | `aria-label` for the sort direction toggle button. Optional. |
| `screenReaderInputCaption` | DynamicValue\<string\> | — | Input caption | `aria-label` for the text input trigger. Optional. |

## Changelog

- **v3.4.0 (2025-10-01):** Fixed Gallery errors when the app is used inside an iframe. Fixed attribute captions not showing correctly in the dropdown list.
- **v1.2.2 (2025-03-31):** Removed an incorrectly applied read-only style that caused the widget to appear visually disabled even when it was not.
- **v1.2.0 (2024-09-13):** Reworked integration with Gallery widget, aligning with the shared `widget-plugin-sorting` package architecture.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] The version jump from v1.2.2 to v3.4.0 is not explained in the CHANGELOG — were there intermediate versions (2.x, 3.0–3.3) with undocumented changes?
- [ ] Is there a maximum number of entries allowed in the `attributes` list? No limit is declared in the widget XML.
- [ ] The `emptyOptionCaption` fallback ("Select an attribute") is defined in `SingleSortController`, not in the widget XML. Is this fallback string localizable or translatable?
- [ ] The widget uses `data-focusindex` on the outer container and the `ul` list — is this a Mendix focus management convention, and does it interact with any platform-level focus trap behavior?
