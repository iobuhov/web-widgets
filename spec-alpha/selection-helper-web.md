# SelectionHelper

## Purpose

The Selection Helper widget provides a visual select-all / deselect-all control for Mendix Data Grid 2 and Gallery widgets that have multi-selection enabled. It reads the parent widget's selection state through the `useSelectionContextValue()` hook and renders either a three-state checkbox (checkbox mode) or developer-supplied widget slots (custom mode). Because all selection state lives in the parent widget's context, the Selection Helper has no independent state — it is a pure adapter between the selection context and the UI.

## User Scenarios

### [P1] Toggle all rows via three-state checkbox
**Given** a Data Grid 2 with multi-selection enabled and a Selection Helper in checkbox mode placed in the grid header  
**When** the user clicks the checkbox  
**Then** `togglePageSelection()` is called on the selection context, cycling between all-selected and none-selected states  

#### Edge Cases
- When placed outside a Data Grid 2 or Gallery, or when multi-selection is not enabled, an Alert is rendered explaining the usage error instead of the checkbox.
- Pressing Enter or Space activates the toggle (keyboard accessible).

### [P2] Custom selection state UI
**Given** a Selection Helper in custom mode with three widget slots configured  
**When** the selection context reports `selectionStatus = "all"`, `"some"`, or `"none"`  
**Then** exactly the matching slot (`customAllSelected`, `customSomeSelected`, or `customNoneSelected`) is rendered inside a clickable container  

#### Edge Cases
- Clicking anywhere in the custom container calls `togglePageSelection()`.
- Keyboard Enter/Space activates the container.
- When the custom container is clicked, the click and keyboard handlers are both wired — no special configuration is needed.

### [P3] Indeterminate state
**Given** a multi-selection grid where some (but not all) rows are selected  
**When** the Selection Helper renders in checkbox mode  
**Then** the `ThreeStateCheckBox` renders in indeterminate state (`"some"` status) showing a half-checked visual  

## Functional Requirements

- FR-001: System MUST retrieve selection state exclusively from `useSelectionContextValue()` provided by the parent Data Grid 2 or Gallery widget.
- FR-002: System MUST render an error Alert when placed outside a compatible parent widget or when the parent does not have multi-selection enabled.
- FR-003: System MUST support two render modes: `checkbox` (renders `ThreeStateCheckBox` with optional label) and `custom` (renders developer-supplied widget slots based on selection status).
- FR-004: In checkbox mode, the `ThreeStateCheckBox` MUST reflect `"all"` (checked), `"some"` (indeterminate), and `"none"` (unchecked) selection states.
- FR-005: In custom mode, the system MUST render exactly one of `customAllSelected`, `customSomeSelected`, or `customNoneSelected` based on the current `selectionStatus`.
- FR-006: Clicking or pressing Enter/Space on the control MUST call `togglePageSelection()` on the selection context.
- FR-007: The component MUST be wrapped in MobX `observer` to reactively re-render on selection state changes.
- FR-008: System MUST NOT expose configurable event actions — the only interaction is `togglePageSelection()`.
- FR-009: `checkboxCaption` MUST only be shown in checkbox mode; custom slots MUST only be available in custom mode.
- FR-010: The widget MUST be web-only (`supportedPlatform="Web"`).

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `renderStyle` | `"checkbox"` \| `"custom"` | `"checkbox"` | Render style | Switches between the built-in ThreeStateCheckBox and developer-supplied widget slots |
| `checkboxCaption` | `DynamicValue<string>` (optional) | — | Checkbox caption | Label text displayed next to the checkbox; dynamic expression, can reference context data; only relevant in checkbox mode |
| `customAllSelected` | `ReactNode` | — | All selected content | Widget slot rendered when selection status is "all"; only relevant in custom mode |
| `customSomeSelected` | `ReactNode` | — | Some selected content | Widget slot rendered when selection status is "some" (partial selection); only relevant in custom mode |
| `customNoneSelected` | `ReactNode` | — | None selected content | Widget slot rendered when selection status is "none"; only relevant in custom mode |

System properties supported: Name, TabIndex.

## Changelog

- **v3.6.1 (2025-10-14)**: Fixed checkbox state sync with selection context — addressed edge cases in MobX observable state initialization order.
- **v1.0.4 (2025-05-26)**: Internal improvements.
- **v1.0.3 (2023-10-13)**: Removed redundant code (performance improvement).
- **v1.0.1 (2023-05-02)**: Added explicit support for placement in Data Grid 2 header.
- **v1.0.0 (2023-03-29)**: Initial release.

Note: The jump from v1.0.4 to v3.6.1 reflects synchronization with a parent package release cycle (e.g., the Gallery or widget bundle versioning scheme).

## Open Questions

> Could not be determined from source code alone — requires human review
- [ ] What is the exact `togglePageSelection()` cycle when starting from partial selection ("some")? The source confirms it cycles between states but the direction (some → all vs some → none) is not visible from this widget's code.
- [ ] Is there a maximum number of items that `togglePageSelection()` selects in "select all" state, or does it select across all pages?
