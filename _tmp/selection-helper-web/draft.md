# selection-helper-web — Draft Spec

Widget: `selection-helper-web`
Package: `packages/pluggableWidgets/selection-helper-web/`
Agent: worker
Date: 2026-05-09

---

## src/SelectionHelper.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. It connects to the parent Data Grid 2 or Gallery widget's selection context via `useSelectionContextValue()` and delegates rendering to `SelectionHelperComponent`.

**2. What kind of logic is described in this file?**
- `observer` from `mobx-react-lite` wraps the component, making it reactive to MobX observable state in the selection context.
- `useSelectionContextValue()` retrieves the selection context from the parent widget. If no context is found or context has an error (`contextValue.hasError`), renders an `Alert` explaining correct usage.
- The `selection.selectionStatus` drives which custom widget slot is rendered as children: `"all"` → `customAllSelected`, `"some"` → `customSomeSelected`, `"none"` → `customNoneSelected`.
- `checkboxCaption?.value ?? ""` is passed as the last child (text label for checkbox mode).
- `onClick` calls `selection.togglePageSelection()`.

**3. What part of behavior can be documented from this file?**
- The widget shows an error `Alert` when placed outside a Data Grid 2 / Gallery context or when multi-selection is not enabled.
- `togglePageSelection()` cycles the selection state: none → all → none (or similar); the exact cycle is defined by the selection context (not this file).
- All three custom slots are passed as children — `SelectionHelperComponent` selects which to display based on `selectionStatus`.
- The component is MobX-observable: re-renders automatically when selection state changes.

**4. Is it user-facing?**
Partially — the error Alert is user-facing (developer error). The rendered output is user-facing.

**5. What new did you learn from this file?**
This widget is a thin adapter — it has no independent state. All logic is in the selection context (provided by Data Grid 2 or Gallery). The widget itself only translates context state into rendered UI.

---

## src/SelectionHelper.xml

**1. What is the purpose of this file?**
Mendix widget descriptor declaring widget identity, category, and properties.

**2. What kind of logic is described in this file?**
Properties in the "General" group:
- `renderStyle` (enum: checkbox/custom, default checkbox) — switches between checkbox UI and custom widget slots.
- `checkboxCaption` (textTemplate, optional) — label shown next to the checkbox.
- `customAllSelected` (widgets slot) — widget content shown when all items are selected.
- `customSomeSelected` (widgets slot) — widget content shown when some items are selected.
- `customNoneSelected` (widgets slot) — widget content shown when no items are selected.

Widget-level flags: `needsEntityContext="true"`, `offlineCapable="true"`, `supportedPlatform="Web"` (explicit web-only).

**3. What part of behavior can be documented from this file?**
- The widget explicitly declares `supportedPlatform="Web"` — not for Mendix native mobile.
- Only the checkbox mode uses `checkboxCaption`; only the custom mode uses the three widget slots. Studio Pro hides the irrelevant properties via `editorConfig.ts`.
- No `onClick` action property — the click behavior is hardwired to `togglePageSelection()` on the selection context.
- No `onChange`, `onFocus`, or other event props — the widget is purely selection-state-driven.

**4. Is it user-facing?**
No — Studio Pro configuration descriptor.

**5. What new did you learn from this file?**
The widget has no configurable event actions — all interaction is implicit through the selection context. This is architecturally different from other widgets; the "action" is baked into the widget's purpose (toggle page selection) rather than configurable by the developer.

---

## typings/SelectionHelperProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript types for the widget's container and preview props.

**2. What kind of logic is described in this file?**
- `SelectionHelperContainerProps` (runtime): `renderStyle: RenderStyleEnum`, `checkboxCaption?: DynamicValue<string>`, `customAllSelected: ReactNode`, `customSomeSelected: ReactNode`, `customNoneSelected: ReactNode`.
- `SelectionHelperPreviewProps` (Studio Pro preview): widget slots typed as `{ widgetCount: number; renderer: ComponentType<...> }`.

**3. What part of behavior can be documented from this file?**
- The three custom slots are `ReactNode` at runtime (already rendered widget content from Mendix framework).
- `checkboxCaption` is `DynamicValue<string>` — can be a dynamic expression referencing context data.

**4. Is it user-facing?**
No — TypeScript types only.

**5. What new did you learn from this file?**
The custom slot children (`ReactNode`) are passed directly into `SelectionHelperComponent` as children and conditionally rendered based on `selectionStatus`. This means the developer can put any Mendix widget(s) into each state slot — buttons, icons, text, etc.

---

## src/components/SelectionHelperComponent.tsx

**1. What is the purpose of this file?**
The presentation component that renders either a three-state checkbox or a custom clickable container based on `type`.

**2. What kind of logic is described in this file?**
- `type === "custom"`: renders `<div class="selection-helper-custom">` with click handler and keyboard handler (Enter/Space calls `onClick`). Children are rendered as-is.
- `type === "checkbox"`: renders `<ThreeStateCheckBox id={id} value={status} onChange={onClick}>` wrapped in Mendix bootstrap checkbox markup (`mx-checkbox form-group label-after`), with `<label>` referencing the checkbox `id`.
- `id` is computed once via `useMemo` using `Date.now().toString()` — a TODO comment suggests replacing with React `useId()`.

**3. What part of behavior can be documented from this file?**
- **ThreeStateCheckBox**: a Mendix component kit checkbox that supports `"all"`, `"some"` (indeterminate), `"none"` states.
- **Checkbox mode**: `"some"` status renders as indeterminate (half-checked). The label is the checkbox caption text.
- **Custom mode**: any widgets placed in the slot are rendered inside a `div` that handles click + keyboard (Enter/Space). No Mendix-standard checkbox DOM — fully custom.
- CSS classes: `widget-selection-helper` always + `className` from props; `selection-helper-custom` or `selection-helper-checkbox mx-checkbox form-group label-after`.

**4. Is it user-facing?**
Yes — this is the visible, interactive component users click.

**5. What new did you learn from this file?**
The `id = Date.now().toString()` approach is technically correct (the component mounts once and the `useMemo` captures the mount timestamp) but non-deterministic and not SSR-safe. The TODO confirms this is known tech debt, with `useId()` as the intended replacement.

---

## src/SelectionHelper.editorConfig.ts

**1. What is the purpose of this file?**
Studio Pro property visibility rules and structure preview rendering.

**2. What kind of logic is described in this file?**
- `getProperties`: when `renderStyle === "checkbox"`, hides all three custom widget slots; when `renderStyle !== "checkbox"` (i.e., "custom"), hides `checkboxCaption`.
- `getPreview`:
  - Checkbox mode: renders an SVG image of an indeterminate checkbox (24×24) + text label from `checkboxCaption`.
  - Custom mode: renders three bordered dropzone containers with placeholders — "No items selected", "Some items selected", "All items selected".

**3. What part of behavior can be documented from this file?**
- In Studio Pro, the custom slots are shown as separate dropzones for each selection state — the developer can drag different widget configurations into each.
- Only one SVG asset (`CheckBoxIndeterminate.light.svg`) — no dark mode variant for the checkbox preview.

**4. Is it user-facing?**
No — Studio Pro only.

**5. What new did you learn from this file?**
The structure preview for custom mode shows all three state dropzones simultaneously, giving the developer a clear view of all three selection states at once. This is a good UX pattern for state-based widget slots.

---

## src/SelectionHelper.editorPreview.tsx

**1. What is the purpose of this file?**
Live React-rendered preview of the Selection Helper widget in Studio Pro design mode.

**2. What kind of logic is described in this file?**
- Always renders with `status="some"` (indeterminate state) regardless of actual data.
- Checkbox mode: passes `checkboxCaption` as children.
- Custom mode: renders all three slot renderers simultaneously (not one at a time), each with a caption placeholder.

**3. What part of behavior can be documented from this file?**
- Preview always shows "some" selected (indeterminate state) as the representative preview.
- Custom mode preview shows all three slots at once — gives the developer visibility into all configured widget content.
- `getPreviewCss()` returns empty string — no CSS injection needed (uses Mendix platform CSS).

**4. Is it user-facing?**
No — Studio Pro design mode preview only.

**5. What new did you learn from this file?**
The decision to show `status="some"` in preview (rather than a configurable or random state) makes sense: indeterminate is the most visually informative state for a checkbox preview — it shows that the checkbox is functional and three-state.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history since initial release (v1.0.0, 2023-03-29).

**2. What kind of logic is described in this file?**
- **v1.0.0 (2023-03-29)**: Initial release of Selection Helper widget.
- **v1.0.1 (2023-05-02)**: Added support for placement in Data Grid 2 header.
- **v1.0.2 (2023-05-26)**: Updated icons/tiles.
- **v1.0.3 (2023-10-13)**: Removed redundant code (performance).
- **v1.0.4 (2025-05-26)**: Internal improvements.
- **v3.6.1 (2025-10-14)**: Fixed checkbox state sync with selection.

**3. What part of behavior can be documented from this file?**
- The widget was introduced in Mendix 10-era (March 2023) as part of Data Grid 2/Gallery selection support.
- v1.0.1 explicitly added Data Grid 2 header support — the widget may not have worked in headers initially.
- The jump from v1.0.4 to v3.6.1 is significant — suggests this widget's versioning was synchronized with a parent package version (e.g., the Gallery module or a widget bundle).
- v3.6.1 fixed a state sync bug where the checkbox state was out of sync with the selection context.

**4. Is it user-facing?**
No — developer changelog.

**5. What new did you learn from this file?**
The version jump from 1.0.x to 3.6.1 is unusual and likely reflects a transition to a new versioning scheme tied to the broader `@mendix/widget-plugin-grid` package or a bundled release cycle. The recent sync fix (v3.6.1) suggests the MobX observable integration had edge cases around state initialization order.

---

## Summary of Key Findings

- **Purpose**: A companion widget for Data Grid 2 and Gallery — provides a visual select-all/deselect-all control that reflects the current multi-selection state.
- **Selection context**: All state is provided by `useSelectionContextValue()` from `@mendix/widget-plugin-grid/selection`; the widget has zero independent state.
- **Two render modes**: "checkbox" (ThreeStateCheckBox with label) and "custom" (developer-supplied widget slots for each of the three states: none/some/all).
- **Three-state checkbox**: Maps "all"/"some"/"none" selection status to checked/indeterminate/unchecked.
- **Error handling**: Renders an Alert with instructions when placed outside a compatible parent widget.
- **MobX reactive**: Wrapped in `observer` — automatically re-renders when grid/gallery selection changes.
- **Web-only**: `supportedPlatform="Web"` — not for native mobile.
- **No event actions**: No configurable Mendix actions; interaction is solely `togglePageSelection()`.
- **Compact**: 7 source files (entry, XML, types, component, editorConfig, editorPreview, CHANGELOG). No unit or E2E test files present.
