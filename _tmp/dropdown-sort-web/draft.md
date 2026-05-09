# Draft: dropdown-sort-web

Widget package: `@mendix/dropdown-sort-web`  
Version: 3.10.0  
Explored: 2026-05-09

---

## src/DropdownSort.xml

**1. What is the purpose of this file?**
This is the Mendix widget definition file. It registers the widget as `com.mendix.widget.web.dropdownsort.DropdownSort`, declares it as a pluggable widget with offline capability, and defines all configurable properties exposed in Studio Pro.

**2. What kind of logic is described in this file?**
Property schema logic: which properties exist, their types, captions, and groupings. The widget is listed under the "Data controls" category in Studio Pro. It declares a `linkedDs` datasource (the gallery datasource to sort), a list of sortable `attributes`, an `emptyOptionCaption`, and two accessibility text templates (`screenReaderButtonCaption`, `screenReaderInputCaption`).

**3. What part of behavior can be documented from this file?**
The `linkedDs` property is typed `isLinked="true"` meaning this widget must be placed inside a context that exposes a datasource (i.e., Gallery header). The `attributes` list allows multiple sort attributes, each with an `attribute` (metadata-only, from `linkedDs`) and a `caption`. Supported attribute types for sorting are: AutoNumber, Decimal, Integer, Long, String, DateTime, Boolean, Enum. The `emptyOptionCaption` is optional and represents the placeholder/unselected state.

**4. Is it user-facing?**
Yes — this file drives the Studio Pro configuration UI, including property captions and descriptions visible to developers building Mendix apps.

**5. What new did you learn from this file?**
The widget is `offlineCapable="true"`, meaning it can function in offline-first Mendix apps. The help URL points to the Gallery module documentation, confirming the widget is bundled as part of the Gallery feature set. The `attribute` within each list item is typed as `isMetaData="true"`, which means only the attribute definition (not runtime values) is passed — the widget uses this to set up sorting instructions, not to read data.

---

## typings/DropdownSortProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript interface file derived from `DropdownSort.xml`. It provides type-safe prop interfaces for the container and preview components.

**2. What kind of logic is described in this file?**
Type declarations only. Defines `AttributesType` (attribute metadata + caption), `DropdownSortContainerProps` (runtime container props), and `DropdownSortPreviewProps` (Studio Pro design preview props).

**3. What part of behavior can be documented from this file?**
`attribute` in `AttributesType` is typed as `AttributeMetaData<Big | string | Date | boolean>`, confirming the widget accepts attributes of numeric (Decimal/Big), string, Date, and boolean types. `emptyOptionCaption`, `screenReaderButtonCaption`, and `screenReaderInputCaption` are all `DynamicValue<string>` and optional. `tabIndex` is optional numeric. The preview type includes `renderMode: "design" | "xray" | "structure"` for Studio Pro canvas rendering modes.

**4. Is it user-facing?**
No — internal TypeScript types for compile-time safety.

**5. What new did you learn from this file?**
The `DropdownSortPreviewProps.className` carries a deprecation note: `@deprecated Deprecated since version 9.18.0. Please use class property instead.` This confirms the widget has existed since before Mendix 9.18.0 and was updated to follow the newer class naming convention.

---

## src/DropdownSort.tsx

**1. What is the purpose of this file?**
The main container component entry point. It wires the Mendix props to the sort state management layer and renders `SortComponent`.

**2. What kind of logic is described in this file?**
Component composition logic. The exported `DropdownSort` is built by applying three HOCs in sequence:
1. `withSortAPI` — fetches the sort API from React context (provided by the Gallery widget)
2. `withLinkedSortStore` — creates a `BasicSortStore` linked to the gallery's sort host
3. `observer` from MobX — makes the component reactive to observable state

Inside the `Container` function, `useSortSelect` drives the UI props (options list, current value, direction, event handlers). A unique `id` is generated via `generateUUID` to ensure each widget instance has unique ARIA attributes.

**3. What part of behavior can be documented from this file?**
The widget requires a sort context injected by a parent Gallery widget — if no context is present, `withSortAPI` shows an error: "widget is out of context. Please place the widget inside the Gallery header." Each widget instance gets a unique UUID-based ID (`DropdownSort{uuid}`) to avoid ARIA collision when multiple instances are on the same page. The `emptyOptionCaption` prop flows to both `useSortSelect` and the `SortComponent` as `placeholder`.

**4. Is it user-facing?**
Yes — the rendered output is the visible dropdown sort control.

**5. What new did you learn from this file?**
Multiple instances of DropdownSort can coexist on a page (e.g., two galleries each with their own sort), each with an independent sort store linked to their respective gallery's host. The MobX `observer` wrapper ensures the component re-renders only when observable sort state changes, not on every parent render.

---

## src/components/SortComponent.tsx

**1. What is the purpose of this file?**
The pure UI component for the dropdown sort widget. It renders a read-only text input (as a trigger), a sort-direction toggle button, and a portal-based dropdown list of sort options.

**2. What kind of logic is described in this file?**
Rendering and interaction logic. Key behaviors:
- The dropdown list is rendered via `createPortal` into `document.body`, positioned using `usePositionObserver` to track the trigger's location. This means the dropdown floats above all other content.
- `useOnClickOutside` closes the dropdown when the user clicks elsewhere.
- The dropdown width matches the input element's width (measured via `clientWidth` ref callback).
- The currently selected option is displayed in the input field by looking up the matching caption from `props.options`.
- The sort direction button toggles between `asc` and `desc` states, reflected via CSS classes `icon-asc` and `icon-desc`.

**3. What part of behavior can be documented from this file?**
The dropdown list has `role="menu"` and each item has `role="menuitem"`. Keyboard behavior is fully implemented: Enter/Space selects an option; Tab on the last item wraps focus to the direction button; Shift+Tab on the first item or Escape returns focus to the input. The input has `aria-haspopup`, `aria-expanded`, `aria-controls`, and `aria-label` attributes. The button has an `aria-label` attribute. The currently selected option receives the `filter-selected` CSS class. On open, focus is automatically moved to the selected item (or the first item if nothing is selected) after a 10ms delay.

**4. Is it user-facing?**
Yes — this is the complete visible and interactive control.

**5. What new did you learn from this file?**
The dropdown is portal-rendered to `document.body` to avoid z-index and overflow clipping issues within the Gallery layout. The `dropdownWidth` state is set dynamically by measuring the input element, ensuring the dropdown matches the trigger width. The `data-focusindex` attribute is set both on the outer container and the `ul` list — this is used by Mendix's focus management system.

---

## src/DropdownSort.editorConfig.ts

**1. What is the purpose of this file?**
Defines the visual structure-mode preview of the widget in Studio Pro's canvas. Returns a `StructurePreviewProps` tree that renders a mock of the dropdown input and sort button.

**2. What kind of logic is described in this file?**
Preview rendering logic for Studio Pro's design/structure modes. It constructs a `RowLayout` with:
- A container showing the `emptyOptionCaption` (or a blank space if not set) styled as secondary italic text
- A chevron-down icon rendered inline as an SVG
- A separator border
- A sort direction icon (asc) with separate light/dark SVG variants

**3. What part of behavior can be documented from this file?**
The widget supports dark mode in Studio Pro's structure preview — different SVG icons are used for `isDarkMode: true`. The preview uses `structurePreviewPalette` for consistent color theming. The empty option caption is shown as placeholder text in the preview input area.

**4. Is it user-facing?**
No — only visible to developers in Studio Pro canvas; not rendered at runtime.

**5. What new did you learn from this file?**
The chevron icon and asc icon are distinct SVG assets. The sort button always shows the ascending icon in the structure preview, regardless of actual sort state. The preview has `borders: true` with `borderRadius: 5` giving it a rounded widget appearance in Studio Pro canvas.

---

## src/DropdownSort.editorPreview.tsx

**1. What is the purpose of this file?**
Provides a live design-mode preview of the DropdownSort widget in Studio Pro — renders the actual `SortComponent` with mock props.

**2. What kind of logic is described in this file?**
Preview composition: wraps `SortComponent` with `withSortAPI` HOC and passes static preview data: `value={null}`, `direction="asc"`, a single mock option `[{ caption: "optionCaption", value: "option" }]`, and the configured accessibility captions and styles.

**3. What part of behavior can be documented from this file?**
In design mode, the widget shows a single mock option to represent the dropdown's appearance. The preview always starts with `value={null}` and `direction="asc"`. No real data binding is active in design mode.

**4. Is it user-facing?**
No — only visible to developers in Studio Pro's design canvas; not rendered at runtime.

**5. What new did you learn from this file?**
The preview uses `parseStyle` from `@mendix/widget-plugin-platform/preview/parse-style` to safely convert the style string from Studio Pro. The `isPreview` flag passed to `withSortAPI` prevents the HOC from trying to locate a real sort context.

---

## e2e/DropDownSort.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the dropdown-sort-web widget in a live Mendix test app.

**2. What kind of logic is described in this file?**
Two test scenarios: (1) sorting in descending order — clicks the dropdown, selects the second attribute, clicks the sort direction button to toggle to descending, and asserts that the first gallery item shows "test3"; (2) sorting in ascending order — toggles descending then back to ascending, and asserts the first item shows "test". Also includes an accessibility test using `@axe-core/playwright` that scans for WCAG 2.1 AA violations, excluding a navigation tree element.

**3. What part of behavior can be documented from this file?**
Confirmed behavioral flow: click input to open dropdown → click a list item to select → click the sort button to toggle direction. The sort direction button is the first `.btn` inside the widget. Toggling direction twice returns to the original sort order. The widget integrates with a Gallery widget (`.mx-name-gallery1`) whose items reflect the sort state. Each test session logs out to avoid session limit issues.

**4. Is it user-facing?**
Yes — tests confirm end-user interactions with the widget at the application level.

**5. What new did you learn from this file?**
The widget passes WCAG 2.1 AA accessibility audit (with `axe-core`). The test confirms the widget works in a real Mendix app connected to a Gallery widget. The sort direction toggle is the sole button element in the widget (the input is not a button). The widget selector uses the `.mx-name-drop_downSort1` CSS class, confirming Mendix's standard widget naming convention.

---

## src/components/__test__/DropdownSort.spec.tsx

**1. What is the purpose of this file?**
Integration unit tests for the `DropdownSort` container component — tests the full component including HOC wiring and sort context.

**2. What kind of logic is described in this file?**
Tests in three groups:
1. **Correct context** — renders with `SortAPI` context, confirms option captions are loaded from `attributes`, snapshot test.
2. **View state** — confirms the initial sort instruction (`attrId("1"), "asc"`) causes `Option 1` to appear as the selected value in the input.
3. **No context** — when `sortContext` is undefined, the widget renders an error alert: "Error: widget is out of context. Please place the widget inside the Gallery header."
4. **Multiple instances** — two instances have different `aria-controls` values on their inputs, confirming UUID-based uniqueness.

**3. What part of behavior can be documented from this file?**
The `initSort` passed to `SortStoreHost` sets the initial selected attribute and direction. The initial sort instruction pre-selects the matching option in the dropdown input. Supported attribute types in tests: String (attrId 1) and Decimal (attrId 2). When placed outside Gallery context, the widget renders a danger alert instead of crashing. Multiple instances on the same page have distinct ARIA `aria-controls` IDs.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The `SortStoreHost` accepts `initSort: [[attrId, direction]]` to pre-initialize sort state, which flows through to the widget's displayed selected value. The error message is exact: "Error: widget is out of context. Please place the widget inside the Gallery header." — this is the user-visible error when the widget is misplaced.

---

## src/components/__test__/SortComponent.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `SortComponent` UI component in isolation — tests rendering, selection behavior, and keyboard focus management.

**2. What kind of logic is described in this file?**
Rendering tests: with options, with no options, with a11y properties, with placeholder. Interaction tests: `onSelect` is called when a menu item is clicked; input displays the selected option caption. Focus management tests: clicking the input opens the dropdown and auto-focuses the first item; Shift+Tab on first item returns focus to input; Tab on last item moves focus to sort button; Escape from any item returns focus to input.

**3. What part of behavior can be documented from this file?**
Focus loop behavior is confirmed: Tab on the last menu item → sort direction button gets focus (not input). Shift+Tab on the first item → input gets focus. Escape from any item → input gets focus. Auto-focus after open uses a `setTimeout(10ms)` delay internally. The placeholder text is set via the HTML `placeholder` attribute on the input. With no options, the component renders without error (snapshot confirmed). The `screenReaderButtonCaption` and `screenReaderInputCaption` are rendered as `aria-label` attributes on their respective elements.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The focus order after Tab on last item is: last menu item → sort button (not input), then presumably Tab again would move out of widget. The sort direction button acts as the last focusable element in the widget's tab order. `jest.useFakeTimers()` is used globally in this test file to control the 10ms focus-advance timeout.

---

## packages/shared/widget-plugin-sorting/src/react/useSortSelect.ts

**1. What is the purpose of this file?**
A React hook that bridges the `SingleSortController` business logic to the UI props expected by `SortComponent`.

**2. What kind of logic is described in this file?**
Maps `SingleSortController` state to `SortComponent` props: `ctrl.selected` → `value`, `ctrl.options` → `options`, `ctrl.direction` → `direction`, `ctrl.select` → `onSelect`, `ctrl.toggleDirection` → `onDirectionClick`. Uses `useSetup` to instantiate and lifecycle-manage the controller.

**3. What part of behavior can be documented from this file?**
The `emptyOptionCaption` passed to the hook becomes the default "empty" option's caption in the dropdown. The controller is set up once per component instance and cleaned up on unmount. The hook is the sole source of all interactive props for `SortComponent`.

**4. Is it user-facing?**
No — internal hook.

**5. What new did you learn from this file?**
The `emptyOptionCaption` has a fallback of "Select an attribute" (defined in `SingleSortController`) if not provided by the widget config — but since `DropdownSort.tsx` passes `props.emptyOptionCaption?.value` directly, an unconfigured value defaults to `undefined`, letting the controller use its fallback.

---

## packages/shared/widget-plugin-sorting/src/controllers/SingleSortController.ts

**1. What is the purpose of this file?**
MobX-observable controller managing the sort selection and direction state for a single-attribute dropdown sort widget.

**2. What kind of logic is described in this file?**
Observable state: `direction` (asc/desc). Computed: `options` (prepends the empty option to the store's attribute list), `selected` (first element of current sortOrder). Actions: `toggleDirection` (flips direction, updates store if an attribute is selected), `select` (sets sortOrder to the chosen attribute+direction, or clears it if "none"). A MobX `reaction` syncs the `direction` observable with the store's sort order when changed externally (e.g., by another widget sharing the same sort store).

**3. What part of behavior can be documented from this file?**
When "none" (the empty option) is selected, `setSortOrder()` is called with no arguments, clearing the sort. Toggling direction without a selected attribute only updates the local `direction` observable — it does not write to the store. The direction is initialized from the store's existing sort order on construction. If the store's sort order is changed externally, the controller's `direction` is synced via reaction.

**4. Is it user-facing?**
No — internal controller, not rendered.

**5. What new did you learn from this file?**
The controller supports external sort order changes — if the Gallery widget or another sort widget modifies the sort order, `SingleSortController` reacts and updates its local `direction` to match. This enables coordinated multi-widget sort scenarios where direction is shared. Selecting "none" fully resets the sort (not just direction).

---

## packages/shared/widget-plugin-sorting/src/react/hocs/withLinkedSortStore.tsx

**1. What is the purpose of this file?**
HOC that creates and links a `SortStoreProvider` to the Gallery's sort host, then injects the resulting `sortStore` into the wrapped component.

**2. What kind of logic is described in this file?**
On mount, instantiates `SortStoreProvider` with the sort host from the `SortAPI` context and the host's initial sort order. On `attributes` prop change (via `useEffect`), calls `store.setProps({ attributes })` to update the available sort options. Renders the wrapped component with `sortStore` injected.

**3. What part of behavior can be documented from this file?**
The sort store is linked to the Gallery's sort host — any selection made in the dropdown propagates up to the Gallery via the shared host. When the `attributes` prop changes (e.g., due to a Mendix prop expression re-evaluation), the available options in the store are updated reactively. The initial sort order from the host is preserved when the store is created.

**4. Is it user-facing?**
No — HOC wiring only.

**5. What new did you learn from this file?**
The widget supports dynamic attribute lists — if the `attributes` prop changes at runtime (which can happen with conditional Mendix expressions), the dropdown options update accordingly without remounting the component.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Version history for the dropdown-sort-web widget.

**2. What kind of logic is described in this file?**
Records of changes per version, following Keep a Changelog format.

**3. What part of behavior can be documented from this file?**
- **v3.4.0** (2025-10-01): Fixed Gallery errors when the app is used inside an iframe. Also fixed attribute captions not showing correctly in the dropdown.
- **v1.2.2** (2025-03-31): Removed incorrectly applied read-only style.
- **v1.2.1** (2024-09-23): Widget maintenance (no behavioral change).
- **v1.2.0** (2024-09-13): Reworked integration with Gallery (significant refactor).
- **v1.1.2** (2023-10-13): Removed redundant code to improve widget load time.
- **v1.1.1** (2023-05-26): Updated light/dark icons and tiles; updated structure mode preview colors.
- **v1.1.0** (2021-12-23): Added dark mode to structure mode preview; added dark icons for Tile and List view.
- **v1.0.0** (2021-09-28): Initial release — capability to sort Gallery widget.

**4. Is it user-facing?**
Indirectly — version history visible to operators and app developers.

**5. What new did you learn from this file?**
A read-only style was incorrectly applied (v1.2.2) — this means the widget previously appeared as read-only (e.g., greyed out input) even when it was not. The v1.2.0 Gallery integration rework aligns with the introduction of the shared `widget-plugin-sorting` package. The version jump from 1.x to 3.x (3.4.0) without intermediate changelog entries suggests a major version bump was part of a larger platform/module release cycle rather than a widget-specific change.
