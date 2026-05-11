## Context

The DropdownSort widget (`@mendix/dropdown-sort-web` v3.10.0) is an existing Mendix pluggable widget. It provides sorting for the Gallery widget via a custom dropdown UI built with React 18 + TypeScript + MobX. The widget is composed of a container (`DropdownSort.tsx`) that wires to the Gallery datasource via `@mendix/widget-plugin-sorting`, and a presentational component (`SortComponent.tsx`) that renders the dropdown UI.

This change produces a spec-only artifact (`openspec/specs/dropdown-sort/spec.md`). No production code changes.

Key source files:
- `src/DropdownSort.tsx` — container, HOC wiring, datasource integration
- `src/components/SortComponent.tsx` — dropdown UI, keyboard handling, portal rendering
- `src/DropdownSort.xml` — property schema (attribute types, captions, accessibility props)

## Goals / Non-Goals

**Goals:**
- Capture observable external behavior as testable requirements and scenarios
- Cover: attribute selection, direction toggle, keyboard navigation, accessibility attributes, empty/placeholder state, dropdown positioning and portal rendering, click-outside dismissal
- Align with existing implementation so spec is immediately valid (no fiction)

**Non-Goals:**
- Internal implementation details (HOC chain, MobX store internals, CSS class names)
- Performance requirements
- Server-side sort mechanics (Gallery datasource responsibility)
- Mobile / touch-specific behavior

## Decisions

**Spec external behavior only, not internal implementation.**
Rationale: the component's internal HOC wiring and MobX store are implementation details. The spec should describe what a user (or test) can observe — inputs, rendered output, UI interactions.

**Use the widget XML property schema as the authoritative source for supported attribute types.**
Rationale: the XML schema (`DropdownSort.xml`) is the contract between the widget and the Mendix platform. The implementation can only receive attribute types the platform validates against this schema.

**Treat portal rendering as an observable behavior (dropdown attaches to `document.body`).**
Rationale: portal rendering is observable and affects DOM structure, z-index layering, and testing. It is a behavior-visible design choice.

**Derive keyboard navigation contract from `SortComponent.tsx` — spec only what the code implements.**
Rationale: no external keyboard spec exists; the implementation is the ground truth. Speccing unimplemented behaviors would create false requirements.

## Risks / Trade-offs

- [Risk] Spec reflects current implementation which may have bugs → Mitigation: flag any observed inconsistencies as open questions rather than speccing the buggy behavior as correct
- [Risk] The `useSortSelect` hook behavior is not specced here (it lives in `@mendix/widget-plugin-sorting`) → Mitigation: spec only the props surface that `SortComponent` exposes; the sorting plugin is an internal dependency

## Open Questions

- Should empty-option selection (clicking an already-selected item or the placeholder) clear the sort, or is that responsibility of the sort store? (Current code delegates to `onSelect` callback — sort store decides.)
