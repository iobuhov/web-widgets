## 1. Spec Review and Validation

- [ ] 1.1 Read `specs/dropdown-sort/spec.md` end-to-end and verify all requirements are accurate against `SortComponent.tsx`
- [ ] 1.2 Verify ARIA attribute requirements match the rendered JSX in `SortComponent.tsx`
- [ ] 1.3 Verify keyboard navigation requirements match the `onKeyDown` handlers in `SortComponent.tsx`
- [ ] 1.4 Resolve the open question in `design.md`: confirm whether selecting an already-active attribute clears the sort or is a no-op (trace `onSelect` → `useSortSelect` → sort store)

## 2. Archive Spec to Canonical Location

- [ ] 2.1 Create `openspec/specs/dropdown-sort/` directory
- [ ] 2.2 Copy `specs/dropdown-sort/spec.md` from this change into `openspec/specs/dropdown-sort/spec.md`

## 3. Test Coverage Mapping

- [ ] 3.1 Audit `src/components/__test__/` — list which spec scenarios already have test coverage
- [ ] 3.2 Add a test for the portal-rendering requirement (dropdown `<ul>` is child of `document.body`)
- [ ] 3.3 Add a test for `aria-expanded` toggling on open/close
- [ ] 3.4 Add a test for `Tab` from last item — focus moves to direction button and dropdown closes
- [ ] 3.5 Add a test for `Shift+Tab` from first item — focus returns to input and dropdown closes
