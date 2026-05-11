## Why

The DropdownSort widget (`@mendix/dropdown-sort-web`) provides sorting functionality for the Gallery widget but lacks formal specification documentation. Without a spec, expected behavior, accessibility requirements, and edge cases are only implicit in the implementation — making it harder to review, test, or extend the widget consistently.

## What Changes

- Create a capability spec for `dropdown-sort` documenting the widget's requirements, behavior contract, and accessibility guarantees
- Cover the dropdown interaction model (open/close, selection, direction toggle)
- Specify keyboard navigation and ARIA requirements
- Define the sort state contract between the widget and the Gallery datasource

## Capabilities

### New Capabilities

- `dropdown-sort`: Sorting dropdown widget that integrates with the Gallery widget datasource — covers attribute selection, direction toggling (asc/desc), keyboard navigation, accessibility (ARIA), and portal-rendered dropdown menu behavior

### Modified Capabilities

## Impact

- Spec documents only — no production code changes
- Affects: `packages/pluggableWidgets/dropdown-sort-web/src/`
- Relevant source files: `DropdownSort.tsx`, `components/SortComponent.tsx`, `DropdownSort.xml`
- Consumers: Mendix apps using the Drop-down Sort widget with Gallery (min platform version 9.17.0)
