# Draft: datagrid-date-filter-web

Widget package: `packages/pluggableWidgets/datagrid-date-filter-web`  
Extracted by: worker  
Date: 2026-05-09

---

## src/DatagridDateFilter.tsx

**1. What is the purpose of this file?**  
This is the root entry point for the DatagridDateFilter widget. It selects between two rendering paths based on the `attrChoice` prop: "auto" (parent DataGrid provides the filter store via React context) or "linked" (the widget owns and manages its own filter store using explicitly configured attributes).

**2. What kind of logic is described in this file?**  
Routing logic only. Three HOC pipelines are assembled at module level: `FilterAuto` wraps the container with a preloader and the parent-provided store HOC; `FilterLinked` wraps it with attribute guard, filter API context, linked store provider, and preloader. The default export selects between them.

**3. What part of behavior can be documented from this file?**  
The `attrChoice` prop is the switch point for the two integration modes. When `attrChoice === "auto"`, the widget expects its parent widget (e.g., Data Grid 2) to inject a `Date_InputFilterInterface` store via context. When `attrChoice === "linked"`, the widget creates its own store from configured attributes and a linked datasource. The preloader delays rendering until default values are resolved.

**4. Is it user-facing?**  
Not directly — it is the invisible routing layer. Users never interact with it but its routing determines which configuration path applies.

**5. What new did you learn from this file?**  
The widget has two distinct operational modes gated by `attrChoice`. The "linked" mode requires `withAttributeGuard` and `withFilterAPI`, indicating those HOCs enforce that a valid DateTime attribute is configured before rendering. The preloader (`withPreloader`) gates rendering on `isLoadingDefaultValues`, preventing flash of empty state.

---

## src/DatagridDateFilter.xml

**1. What is the purpose of this file?**  
Widget descriptor XML that defines all user-configurable properties exposed in Mendix Studio Pro. This is the contract between the platform and the widget.

**2. What kind of logic is described in this file?**  
Property declarations with types, captions, and enumerations. Covers four property groups: General (filter attributes, defaults, placeholder, adjustable), Configurations (saved attributes), Events (onChange action), and Accessibility (screen reader captions).

**3. What part of behavior can be documented from this file?**  
- `attrChoice` (enum: "auto" | "linked") controls attribute source mode.  
- `defaultFilter` (enum, default "equal") sets the initial filter comparison operator — 9 options: between, greater, greaterEqual, equal, notEqual, smaller, smallerEqual, empty, notEmpty.  
- `defaultValue` (DateTime expression) applies when filter is not "between".  
- `defaultStartDate` and `defaultEndDate` (DateTime expressions) apply only when filter is "between".  
- `adjustable` (boolean, default true) allows end-users to switch the filter type at runtime.  
- `valueAttribute`, `startDateAttribute`, `endDateAttribute` (EditableValue<DateTime>) store the last filter values for persistence.  
- `onChange` (action) fires when filter value or type changes.  
- Three screen reader captions (textTemplate) for the comparison button, calendar button, and input element provide ARIA label customization.

**4. Is it user-facing?**  
Yes — this file defines everything developers configure in Studio Pro, which directly determines end-user behavior.

**5. What new did you learn from this file?**  
The widget is categorized under "Data controls" in both Studio and Studio Pro. It supports offline capability (`offlineCapable="true"`). The `attributes` list property is linked to `linkedDs` and only accepts `DateTime` attribute type. The `attributes` object list is only relevant in "linked" (`attrChoice === "linked"`) mode; in "auto" mode, the parent DataGrid provides the filterable attribute.

---

## src/DatagridDateFilter.editorConfig.ts

**1. What is the purpose of this file?**  
Defines the Studio Pro editor configuration: which properties are shown or hidden based on current prop values, and what the widget looks like in the structure/design mode canvas preview.

**2. What kind of logic is described in this file?**  
`getProperties` hides irrelevant properties based on `adjustable` and `attrChoice` values. `getPreview` builds a structure preview using `@mendix/widget-plugin-platform`'s preview API — renders a bordered row with optional filter icon (when adjustable), placeholder text, and a calendar icon.

**3. What part of behavior can be documented from this file?**  
- When `adjustable = false`, the `screenReaderButtonCaption` property is hidden (since there is no filter type button).  
- When `attrChoice === "auto"`, the `attributes` list and `linkedDs` properties are hidden.  
- When `defaultFilter !== "between"`, `defaultStartDate`, `defaultEndDate`, `startDateAttribute`, and `endDateAttribute` are hidden; when it is "between", `defaultValue` and `valueAttribute` are hidden. These two groups are mutually exclusive.  
- The preview renders icons for all 9 filter types using dedicated SVG assets (light/dark variants).

**4. Is it user-facing?**  
Only in Studio Pro's design canvas — no runtime behavior.

**5. What new did you learn from this file?**  
The mutual exclusion between "between" and single-date modes is enforced at the property panel level: the saved attribute and default value fields completely change depending on the filter mode. This is a strong behavioral constraint — a widget configured for "between" stores `startDateAttribute` and `endDateAttribute` separately, while other filter types use `valueAttribute`.

---

## src/DatagridDateFilter.editorPreview.tsx

**1. What is the purpose of this file?**  
Provides the React preview component used by Studio Pro's page editor to render the widget in design/xray/structure mode.

**2. What kind of logic is described in this file?**  
Enables MobX static rendering (for SSR context), then exports `preview` = `FilterComponent` wrapped in `withPreviewAdapter`. Also exports `getPreviewCss` which injects the react-datepicker stylesheet into the preview.

**3. What part of behavior can be documented from this file?**  
The preview renders a functional (but non-interactive) instance of the `FilterComponent` with preview-adapted props. The calendar picker CSS is included so the preview accurately reflects the picker's visual appearance.

**4. Is it user-facing?**  
Only in Studio Pro design mode — not at runtime.

**5. What new did you learn from this file?**  
MobX static rendering is explicitly enabled for the preview context, confirming the component tree uses MobX observers that would otherwise subscribe to stores. The preview uses `withPreviewAdapter` to bridge the preview props schema to the runtime component's interface.

---

## src/components/DateFilterContainer.tsx

**1. What is the purpose of this file?**  
The primary runtime container that bridges the Mendix widget props and the filter store to the `FilterComponent` presentation layer. It is an MobX observer component.

**2. What kind of logic is described in this file?**  
Uses `useSetup` to create and initialize the `DatePickerController`; uses `useActionEvents` to subscribe to external reset/set-value events; uses `useDateSync` to synchronize `valueAttribute`/`startDateAttribute`/`endDateAttribute` with the filter store; then renders `FilterComponent` with controller-derived state.

**3. What part of behavior can be documented from this file?**  
- The controller's `pickerState` is reactive (MobX computed) — the component re-renders when filter state changes.  
- `tabIndex` defaults to 0 if not provided.  
- `placeholder`, `screenReaderButtonCaption`, `screenReaderCalendarCaption`, `screenReaderInputCaption` are all optional and passed through to the UI.  
- `parentChannelName` enables cross-widget communication when the filter is inside a container that manages a filter group.  
- `useDateSync` handles two-way sync between the filter store and the Mendix editable attributes (`valueAttribute`, `startDateAttribute`, `endDateAttribute`).

**4. Is it user-facing?**  
Yes — this component controls the direct user interaction layer.

**5. What new did you learn from this file?**  
The component is explicitly typed as `observer` from `mobx-react-lite`, meaning it tracks observable reads from the `DatePickerController`. The `useSetup` + `useActionEvents` pattern separates initialization (once) from reactive rendering (on every store change). `parentChannelName` is threaded through from the parent HOC, enabling filter group coordination.

---

## src/components/FilterComponent.tsx

**1. What is the purpose of this file?**  
The presentational root component for the date filter UI. It assembles the optional filter type selector and the date picker into a single container div.

**2. What kind of logic is described in this file?**  
Renders a div with class `filter-container` + widget CSS class. Conditionally renders the `FilterSelector` (from `@mendix/widget-plugin-filtering/controls`) when `adjustable=true`. Always renders `DatePicker`. Passes accessibility props through. Defines the static `OPTIONS` array for the 9 filter types.

**3. What part of behavior can be documented from this file?**  
- When `adjustable=true`, a `FilterSelector` button appears that lets the user change the filter type via a dropdown. The default aria label is "Select filter type" (overridable via `screenReaderButtonCaption`).  
- The 9 filter options are: between, greater, greaterEqual, equal, notEqual, smaller, smallerEqual, empty, notEmpty.  
- The container uses `data-focusindex` for focus management.

**4. Is it user-facing?**  
Yes — this is the top-level UI the end user sees and interacts with.

**5. What new did you learn from this file?**  
The `FilterSelector` is entirely hidden when `adjustable=false` — the filter type is then fixed and the end user cannot change it. The OPTIONS array is static (not dynamic), confirming the 9 filter types are hardcoded and cannot be extended by widget configuration.

---

## src/components/DatePicker.tsx

**1. What is the purpose of this file?**  
Wraps the `react-datepicker` library component with Mendix-specific behavior: a portal container for proper z-index layering, a screen-reader label, and a calendar open/close button.

**2. What kind of logic is described in this file?**  
Initializes stable props (`portalId`, `popperPlacement`, `popperProps`) once via `useSetup`/`useState` to avoid re-creation. Renders the `ReactDatePicker` with all picker configuration, plus a calendar button. The button uses `onMouseDown` (not `onClick`) to avoid race conditions with the calendar's outside-click handler.

**3. What part of behavior can be documented from this file?**  
- `popperProps.strategy = "fixed"` prevents clipping when the filter is inside scrollable containers.  
- `shouldCloseOnSelect={false}` keeps the calendar open when selecting a range start date.  
- `allowSameDay={false}` prevents start and end date being the same in range mode.  
- `strictParsing` is enabled — typed dates that don't match the format are rejected.  
- `showMonthDropdown` and `showYearDropdown` are always shown — users can navigate by month and year dropdowns.  
- `isClearable={props.selectsRange}` — the clear button only appears in "between" range mode.  
- `enableTabLoop` keeps keyboard tab focus inside the open calendar.  
- The calendar button toggles the picker; screen reader label defaults to "Show calendar".  
- The `date-filter-container` div is the portal target — created at component mount and stable per instance.

**4. Is it user-facing?**  
Yes — this is the core UI that users directly interact with.

**5. What new did you learn from this file?**  
The `portalId` is generated with `Math.random()` once at mount — this is what ensures unique DOM attachment for each date filter instance on a page. The calendar button uses `mousedown` not `click` to win the race with `react-datepicker`'s outside-click dismissal logic. `useWeekdaysShort={false}` is set — the picker uses full day names, but the test suite confirms these are truncated locale-appropriately.

---

## src/components/CalendarIcon.tsx

**1. What is the purpose of this file?**  
Static SVG calendar icon rendered inside the calendar toggle button.

**2. What kind of logic is described in this file?**  
No logic — returns a single `<svg>` element with `fill="currentColor"` so it inherits the button's text color.

**3. What part of behavior can be documented from this file?**  
The icon uses `currentColor` so it adapts to the button's color/theme automatically. The viewBox is `0 0 32 32`.

**4. Is it user-facing?**  
Yes — it is the visual indicator for the calendar open button.

**5. What new did you learn from this file?**  
The icon is an inline SVG (not an external file or font icon), which avoids HTTP requests and works offline.

---

## src/components/typings.ts

**1. What is the purpose of this file?**  
Defines the `DateFilterProps` interface shared between the container and HOCs — the interface that any HOC must provide to `DateFilterContainer`.

**2. What kind of logic is described in this file?**  
Type definition only: `filterStore: Date_InputFilterInterface` (required) and `parentChannelName?: string` (optional).

**3. What part of behavior can be documented from this file?**  
Every runtime HOC path must provide a `Date_InputFilterInterface` store to the container. The `parentChannelName` enables channel-scoped event delivery for filter group coordination.

**4. Is it user-facing?**  
No — internal interface definition.

**5. What new did you learn from this file?**  
The `filterStore` prop type (`Date_InputFilterInterface`) comes from `@mendix/widget-plugin-filtering` — a shared internal package. This confirms the date filter integrates with the standard Mendix filter store abstraction, not a custom one.

---

## src/helpers/DatePickerController.ts

**1. What is the purpose of this file?**  
MobX observable controller that manages all date filter state and user interactions. It is the single source of truth for picker state derived from the filter store.

**2. What kind of logic is described in this file?**  
MobX class with observables (`expanded`), computed (`pickerState`), and action handlers for user events (picker change, calendar open/close, button mouse/keyboard, key down, filter type change). Also handles external events via `handleReset` and `handleSetValue`. Initializes the filter store with default state on `setup()`.

**3. What part of behavior can be documented from this file?**  
- `pickerState` computes UI state from the filter store: `disabled=true` when filter is "empty" or "notEmpty" (no date needed); `selectsRange=true` only for "between"; `selected` is used for single-date modes; `startDate`/`endDate` are used for range mode.  
- Backspace on the input in range mode clears both dates (via `this._filter.clear()`).  
- Filter type change (`handleFilterChange`) immediately opens the calendar.  
- Calendar button uses `mousedown` with a `setTimeout`-deferred `setFocus()` to avoid calling focus on a momentarily-disabled element.  
- `handleSetValue` supports external programmatic control: sets `filterFunction`, `arg1` (dateTimeValue), and `arg2` (dateTimeValue2) atomically via `runInAction`.  
- `UNSAFE_handleChangeRaw` prevents direct text input in range mode — users must use the calendar UI or interact with the range start/end separately.  
- `getDefaults` produces `[filterFn, startDate, endDate]` for "between" or `[filterFn, date]` for other types.

**4. Is it user-facing?**  
Indirectly — all user-visible filter behaviors are orchestrated here.

**5. What new did you learn from this file?**  
The disabled state is filter-function-driven: the input element is disabled when the filter is "empty" or "notEmpty" because those comparisons don't require a date value. The `UNSAFE_` prefix on `handleChangeRaw` is documented in the file itself as a "UX tweak" that prevents typed input in range mode, with an explicit note that it can be removed if needed. External `Set_Filter` actions can set both `filterFunction` and values atomically.

---

## src/helpers/useSetup.ts

**1. What is the purpose of this file?**  
React hook that initializes the `DatePickerController` once per component mount, derives locale and date format from Mendix session config, and generates a unique ID for the filter.

**2. What kind of logic is described in this file?**  
Creates `DatePickerController` in a stable `useState` initializer. Uses `useEffect` to call `controller.setup()` on mount (which initializes the filter store defaults). Returns a memoized object with `calendarStartDay`, `dateFormat`, `controller`, `id`, and `locale`.

**3. What part of behavior can be documented from this file?**  
- The controller is created only once (stable across re-renders).  
- `calendarStartDay` comes from `locale.firstDayOfWeek` — the app's locale setting controls which day the calendar week starts on.  
- `dateFormat` is computed from `pickerDateFormat(locale)`, which normalizes single-digit format patterns to their double-digit equivalents.  
- `id` is `DateFilter` + UUID, unique per filter instance on the page.  
- `locale` is registered via `setupLocales` using the Mendix session's language tag.

**4. Is it user-facing?**  
Indirectly — locale, date format, and calendar week start are all user-visible.

**5. What new did you learn from this file?**  
The date format used by the picker input is derived from the Mendix app's session locale at mount time — the widget adapts to the app's regional format, not a fixed one. The `generateUUID` function (from `@mendix/widget-plugin-platform`) ensures the accessible label ID is unique across multiple filter instances on a single page, which was a historical bug fixed in v2.0.1.

---

## src/helpers/useActionEvents.ts

**1. What is the purpose of this file?**  
React hook that subscribes the date filter to external "Reset Filter" and "Set Filter Value" action events from the Mendix platform.

**2. What kind of logic is described in this file?**  
Creates stable params objects in `useState` to avoid re-subscribing on every render. Calls `useOnResetValueEvent` and `useOnSetValueEvent` from `@mendix/widget-plugin-external-events`. The controller's `handleReset` and `handleSetValue` are the event listeners.

**3. What part of behavior can be documented from this file?**  
- `Reset_Filter` action (scoped to this widget's name + optional parentChannelName) triggers `handleReset` — can reset to defaults or clear all values.  
- `Set_Filter` action triggers `handleSetValue` — allows external microflows or nanoflows to programmatically set the filter's operator and date values.  
- `parentChannelName` scopes event delivery: when the filter is inside a widget that manages a filter group (e.g., a custom filter bar), events are routed through the channel rather than by widget name alone.

**4. Is it user-facing?**  
Indirectly — enables microflow/nanoflow actions to control the filter programmatically.

**5. What new did you learn from this file?**  
The channel-based event routing (`parentChannelName`) enables filter group scenarios where a single "Reset All" button resets all filters within a container simultaneously. The stable params pattern (single `useState` call) prevents hook re-subscription on every render, which is important for correctness with event subscription APIs.

---

## src/helpers/base-types.ts

**1. What is the purpose of this file?**  
Re-exports `DefaultFilterEnum` from the generated typings as `FilterTypeEnum` — provides a stable internal name for the filter type union.

**2. What kind of logic is described in this file?**  
Single type re-export. No logic.

**3. What part of behavior can be documented from this file?**  
The internal type `FilterTypeEnum` is identical to `DefaultFilterEnum` from the generated props: `"between" | "greater" | "greaterEqual" | "equal" | "notEqual" | "smaller" | "smallerEqual" | "empty" | "notEmpty"`.

**4. Is it user-facing?**  
No — internal type alias.

**5. What new did you learn from this file?**  
The generated type is used directly as the internal filter function type — no separate internal enum, confirming the filter types are fully specified by the widget XML schema.

---

## src/hocs/withLinkedDateStore.tsx

**1. What is the purpose of this file?**  
Higher-order component that creates a `DateStoreProvider` (a MobX store) from the widget's explicitly configured attributes and filter API, then passes the resulting store to the wrapped component.

**2. What kind of logic is described in this file?**  
Uses `useSetup` from `@mendix/widget-plugin-mobx-kit` to instantiate `DateStoreProvider` once per mount. The `DateStoreProvider` is constructed with the `filterAPI` context, the list of DateTime attributes from `props.attributes`, and a `dataKey` (`props.name`) for store identity.

**3. What part of behavior can be documented from this file?**  
- Used only when `attrChoice === "linked"` (Custom mode).  
- Requires `attributes` array and `name` prop.  
- The `DateStoreProvider` creates and manages the filter store, registering it with the `filterAPI` using the widget's `name` as key.  
- `parentChannelName` is threaded from the `filterAPI` context.

**4. Is it user-facing?**  
Indirectly — enables the "Custom" configuration mode.

**5. What new did you learn from this file?**  
The `dataKey: props.name` means the filter store is keyed by the widget's Mendix name — important for uniqueness when multiple date filters are on the same page in linked mode. The `DateStoreProvider` is from a shared internal package (`@mendix/widget-plugin-filtering`), which manages the store lifecycle and registration with the parent filter API context.

---

## src/hocs/withParentProvidedDateStore.tsx

**1. What is the purpose of this file?**  
Higher-order component that reads the date filter store from the parent widget's React context (Data Grid 2 or Gallery) and passes it to the wrapped component. Handles all error states.

**2. What kind of logic is described in this file?**  
`useDateFilterAPI` reads the filter context using `useFilterAPI`. Validates that: the context is present, the provider has no error, the store type is "direct" (not delegated), the store is an input-type store, and the store is for DateTime (not another type). Returns a Result monad. On error, renders an `<Alert>` with the error message.

**3. What part of behavior can be documented from this file?**  
- Used when `attrChoice === "auto"`.  
- If the widget is placed outside a Data Grid 2 or Gallery header, it renders an error alert: "The filter widget must be placed inside the column or header of the Data grid 2.0 or inside header of the Gallery widget."  
- If the parent provides a store of the wrong type (e.g., a text filter store), a type mismatch error is shown.  
- On success, the `filterStore` and `parentChannelName` are passed down from the parent's filter API.

**4. Is it user-facing?**  
Yes — the error alert is shown to end users (or developers during development) if the widget is placed incorrectly.

**5. What new did you learn from this file?**  
The `dateAPI.current ??= {...}` pattern caches the resolved API object across re-renders — this prevents the wrapped component from receiving a new reference on every render (which would break memoization). The error message for misplaced widget is user-readable and explicitly mentions Data Grid 2 and Gallery, confirming these are the only supported parent widgets for "auto" mode.

---

## src/hocs/withPreviewAdapter.tsx

**1. What is the purpose of this file?**  
Adapts the Studio preview props (`DatagridDateFilterPreviewProps`) to the runtime `FilterComponent` interface for rendering in Studio Pro's canvas.

**2. What kind of logic is described in this file?**  
Creates a wrapper that remounts `FilterComponent` (via a random `key`) when `adjustable` or `defaultFilter` changes in the preview — this ensures the component re-initializes rather than updating in place. Passes non-interactive handlers (`() => {}`) for `onChange` and `onFilterChange`.

**3. What part of behavior can be documented from this file?**  
- Preview is non-interactive (no real callbacks).  
- `filterFn` is set to `props.defaultFilter`, so the preview icon matches the configured default filter.  
- `styleObject` (Studio preview's style object) maps to the runtime `style` prop.  
- The `key` reset on `adjustable`/`defaultFilter` change ensures the preview updates when those properties change in Studio.

**4. Is it user-facing?**  
Only in Studio Pro design mode.

**5. What new did you learn from this file?**  
The key remount technique is used specifically because the picker component's internal state (initialized via `useState`) would not respond to prop changes without a full remount. This is an important behavioral note: the filter component does not dynamically react to `adjustable` or `defaultFilter` prop changes at runtime after mount.

---

## src/utils/date-utils.ts

**1. What is the purpose of this file?**  
Provides locale-aware date format utilities and locale registration for the react-datepicker library.

**2. What kind of logic is described in this file?**  
Three main functions: `doubleMonthOrDayWhenSingle` normalizes short date format patterns; `dayOfWeekWhenUpperCase` converts uppercase 'E' to lowercase 'e' for compatibility; `pickerDateFormat` derives the picker's accepted format from the app locale; `setupLocales` registers the locale from the Mendix session with react-datepicker; `getLocale` reads the Mendix session locale from `window.mx` with a fallback to en-US.

**3. What part of behavior can be documented from this file?**  
- The picker accepts both the normalized format and the original format (as an array) when they differ — users can type with or without leading zeros.  
- Locale support covers any language tag available in `date-fns/locale` (e.g., en-US, pt-BR, fi-FI). Both exact matches (e.g., "ptBR") and language-only fallbacks (e.g., "pt") are attempted.  
- The `firstDayOfWeek` from the Mendix session locale determines which day the calendar week starts on.  
- When `window.mx` is not available (e.g., in tests or offline), the fallback locale is en-US with M/d/yyyy format.  
- Custom date format patterns using uppercase 'E' (day of week) are converted to lowercase 'e' to conform to the unicode standard used by `date-fns`.

**4. Is it user-facing?**  
Yes — date format and locale directly affect how users type and read dates in the filter.

**5. What new did you learn from this file?**  
The dual-format array (`[normalized, original]`) is a UX workaround: if the app uses "d/M/yyyy" (single-digit), users can still type "09/02/2002" with leading zeros. This was an explicitly documented bug fix. The uppercase 'E' fix was introduced to handle customers using custom formats like "YYww.E" — the format was causing errors before v2.7.0.

---

## src/utils/widget-utils.ts

**1. What is the purpose of this file?**  
Provides the `isLoadingDefaultValues` predicate used by the preloader HOC to delay rendering until default values are resolved.

**2. What kind of logic is described in this file?**  
Checks the `.status` of `defaultValue`, `defaultStartDate`, and `defaultEndDate` expressions. Returns `true` if any of them is "loading".

**3. What part of behavior can be documented from this file?**  
The widget defers rendering while any default date expression is loading. This prevents the filter from initializing with undefined defaults and then jumping to the correct value — ensuring the filter is initialized exactly once with the correct defaults.

**4. Is it user-facing?**  
Indirectly — prevents visual flash or incorrect initial filter state.

**5. What new did you learn from this file?**  
The check covers all three possible default value expressions (`defaultValue`, `defaultStartDate`, `defaultEndDate`). Even if only one is configured, the widget waits for all three (though only the relevant ones will be non-undefined). This is a defensive approach that guarantees initialization correctness.

---

## typings/DatagridDateFilterProps.d.ts

**1. What is the purpose of this file?**  
Auto-generated TypeScript type definitions from the widget XML. Provides the full typed interface for the widget's container props and preview props.

**2. What kind of logic is described in this file?**  
Type declarations only. Defines `AttrChoiceEnum`, `AttributesType`, `DefaultFilterEnum`, `DatagridDateFilterContainerProps`, and `DatagridDateFilterPreviewProps`.

**3. What part of behavior can be documented from this file?**  
Complete props interface:  
- `name`, `class`, `style?`, `tabIndex?` — standard Mendix widget props  
- `attrChoice: AttrChoiceEnum` — "auto" | "linked"  
- `attributes: AttributesType[]` — list of DateTime attributes for linked mode  
- `defaultValue?: DynamicValue<Date>`, `defaultStartDate?: DynamicValue<Date>`, `defaultEndDate?: DynamicValue<Date>`  
- `defaultFilter: DefaultFilterEnum`  
- `placeholder?: DynamicValue<string>`  
- `adjustable: boolean`  
- `valueAttribute?: EditableValue<Date>`, `startDateAttribute?: EditableValue<Date>`, `endDateAttribute?: EditableValue<Date>`  
- `onChange?: ActionValue`  
- `screenReaderButtonCaption?`, `screenReaderCalendarCaption?`, `screenReaderInputCaption?: DynamicValue<string>`

**4. Is it user-facing?**  
No — internal type definitions.

**5. What new did you learn from this file?**  
`className` in the preview props is marked `@deprecated since version 9.18.0` — use `class` instead. This is a Mendix framework migration note.

---

## typings/global.d.ts

**1. What is the purpose of this file?**  
Defines TypeScript interfaces for the `window.mx` Mendix global object, specifically the session locale configuration.

**2. What kind of logic is described in this file?**  
Type interfaces for `MXLocalePatterns`, `MXSessionLocale`, `MXSessionConfig`, `MXSession`, and `MXGlobalObject`. Extends the global `Window` interface with optional `mx` property.

**3. What part of behavior can be documented from this file?**  
The session locale exposes: `code` (e.g., "en_US"), `firstDayOfWeek` (0=Sunday, 1=Monday), `languageTag` (e.g., "en-US"), and `patterns` with `date`, `datetime`, and `time` format strings.

**4. Is it user-facing?**  
No — internal type definitions.

**5. What new did you learn from this file?**  
`window.mx` is typed as optional (`mx?`), confirming the fallback locale in `date-utils.ts` is necessary for non-Mendix test environments.

---

## e2e/DataGridDateFilter.spec.js

**1. What is the purpose of this file?**  
End-to-end Playwright tests for the DataGrid Date Filter widget.

**2. What kind of logic is described in this file?**  
Tests cover: visual screenshot baseline; typed date filtering; range ("between") date selection via calendar UI; default value initialization; default start/end date initialization; and WCAG 2.1 AA accessibility scan using axe-core.

**3. What part of behavior can be documented from this file?**  
- Typed date filtering: user can type "10/5/2020" into the filter input and the datagrid filters to matching rows (e2e-confirmed behavioral path).  
- "Between" filter: user selects the "Between" option, then picks a date range via calendar; the datagrid filters to rows in that range.  
- The filter selector button is labeled "Between" when that mode is active (accessible name confirmed).  
- Default value initializes the filter on page load — confirmed with 8 rows visible (matching rows for a specific birth date).  
- Default start/end dates initialize the "between" filter on load — confirmed with 11 rows visible.  
- WCAG 2.1 AA compliance is verified excluding the navigation tree.  
- One test is marked `test.fixme` — the calendar date picker screenshot test is skipped, indicating it is flaky or not yet stable.  
- Session logout is performed after each test to stay within the 5-session Mendix license limit.

**4. Is it user-facing?**  
Tests only, but confirms end-user-facing behaviors.

**5. What new did you learn from this file?**  
The `filter_init_condition` route hosts a dedicated page for testing default value initialization. Two separate datagrids on that page test single default value vs. date range defaults. The calendar interaction requires closing the calendar by clicking outside (the `layoutGrid` click after range selection), confirming the calendar doesn't auto-close on the first date of a range selection — consistent with `shouldCloseOnSelect={false}`.

---

## src/components/__tests__/DatagridDateFilter.spec.tsx

**1. What is the purpose of this file?**  
Unit tests for the root `DatagridDateFilter` component, covering rendering with context, event handling, default value behavior, locale-sensitive calendar rendering, and multi-instance uniqueness.

**2. What kind of logic is described in this file?**  
Tests: render snapshot with single/double DateTime attributes; error alert when no FilterAPI context; trigger `onChange` and `valueAttribute.setValue` on filter value change; default value initialization; no sync when defaultValue changes after mount; calendar week day rendering for en-US (Sunday start), en-US (Monday start), pt-BR, and fi-FI; unique span IDs across multiple instances.

**3. What part of behavior can be documented from this file?**  
- Without FilterAPI context, widget renders an alert: "The filter widget must be placed inside the column or header of the Data grid 2.0 or inside header of the Gallery widget."  
- On filter value change, both `onChange` action and `valueAttribute.setValue` are called (each once per change event).  
- `defaultValue` sets the initial input value (e.g., 01/01/2000 from epoch 946684800000).  
- **Behavioral constraint**: `defaultValue` is only used as an initial value. After mount, changes to `defaultValue` prop (from undefined to date, or from date to undefined) do NOT update the filter — confirmed by two unit tests.  
- Calendar week start: en-US defaults to Sunday ("SuMoTuWeThFrSa"), en-US with `firstDayOfWeek=1` starts Monday ("MoTuWeThFrSaSu").  
- pt-BR locale: abbreviated day names in Portuguese ("domsegterquaquisexsab").  
- fi-FI locale: abbreviated day names in Finnish ("matiketopelasu" for Monday start, "sumatiketopela" for Sunday start).  
- Multiple filter instances get unique accessible label IDs (span id values differ between instances).

**4. Is it user-facing?**  
Tests only, but directly confirms user-facing behaviors.

**5. What new did you learn from this file?**  
The "default value only as initial value" behavior is explicitly tested and confirmed — this was a breaking change in v2.5.0. The locale tests confirm that the calendar header uses abbreviated (not single-letter) day names. The FilterAPI is injected via a global window context key `"com.mendix.widgets.web.filterable.filterContext.v2"` — the widget reads this at render time.

---

## src/components/__tests__/FilterComponent.spec.tsx

**1. What is the purpose of this file?**  
Unit tests for the `FilterComponent` — the top-level UI wrapper.

**2. What kind of logic is described in this file?**  
Three snapshot tests: default render (adjustable=true); render when not adjustable (no filter selector); render with custom aria labels.

**3. What part of behavior can be documented from this file?**  
- When `adjustable=true`, the `FilterSelector` is rendered (confirmed by snapshot).  
- When `adjustable=false`, the `FilterSelector` is absent from the output.  
- `screenReaderButtonCaption` and `screenReaderInputCaption` are rendered in the accessible label.

**4. Is it user-facing?**  
Tests only — confirms visible UI structure.

**5. What new did you learn from this file?**  
`ReactDOM.createPortal` is mocked in tests to inline portal content rather than attaching to a real DOM node — necessary because the date picker uses a portal for z-index layering. The `Math.random` mock (0.123456789) stabilizes the portal ID for snapshot comparisons.

---

## src/components/__tests__/DatePicker.spec.tsx

**1. What is the purpose of this file?**  
Unit tests for the `DatePicker` component, covering rendering variants and the `doubleMonthOrDayWhenSingle` date format normalization utility.

**2. What kind of logic is described in this file?**  
Snapshot tests for: default render (adjustable=true), non-adjustable render (no filter-input class), locale+dateFormat render (fr_FR, yyyyMMdd format), accessibility label render. Plus a comprehensive parameterized test for `doubleMonthOrDayWhenSingle` with 29 input/output pairs.

**3. What part of behavior can be documented from this file?**  
- `adjustable=true` adds `"filter-input"` class to the date input; `adjustable=false` omits it.  
- Locale and dateFormat props are forwarded to the underlying `ReactDatePicker`.  
- `screenReaderInputCaption` overrides the default "date filter" label on the hidden span.  
- `screenReaderCalendarCaption` overrides the default "Show calendar" on the calendar button.  
- Date format normalization: single-digit `d` becomes `dd`, single-digit `M` becomes `MM`. Examples: "d/M/yyyy" → "dd/MM/yyyy"; "yyyy-MM-d" → "yyyy-MM-dd"; "yyyyMdd" → "yyyyMMdd". The lowercase `m` and `mm` patterns are NOT normalized (they represent minutes, not months).

**4. Is it user-facing?**  
Tests only — confirms user-visible behaviors.

**5. What new did you learn from this file?**  
The `doubleMonthOrDayWhenSingle` tests are the most comprehensive spec for the format normalization behavior, covering 29 cases. The distinction between uppercase M (month) and lowercase m (minute) is critical — the function only normalizes `M` and `d`, not `m`. The `filter-input` class on the date input is conditional on `adjustable`, which affects CSS styling.

---

## CHANGELOG.md

**1. What is the purpose of this file?**  
Documents the version history of the datagrid-date-filter-web widget from v2.0.0 (2021-09-28) to v3.10.0 (2026-05-06).

**2. What kind of logic is described in this file?**  
No logic — version entries with added, changed, fixed, and breaking change categories.

**3. What part of behavior can be documented from this file?**  

**Version history (selected behavioral findings):**

- **v3.10.0 (2026-05-06):** Fixed filter selector dropdown placement on small viewports.  
- **v3.8.1 (2026-02-19):** Filter selector dropdown now auto-selects best placement based on available space.  
- **v3.8.0 (2026-01-16):** Fixed background-color hover styles in date picker.  
- **v3.4.0 (2025-09-12):** Fixed Axe a11y label issues.  
- **v2.11.2 (2025-03-20):** Fixed: non-adjustable default filters were overridden by personalization settings — behavioral constraint: if `adjustable=false`, the filter type cannot be changed by personalization.  
- **v2.11.1 (2025-01-21):** Fixed incorrect range date filter behavior in some cases.  
- **v2.10.4 (2024-11-13):** Breaking: filter type select button now opens on Enter, Space, and arrow keys (keyboard behavior change). Added improved screen reader integration.  
- **v2.10.3 (2024-10-31):** Fixed grid-wide filter not resetting.  
- **v2.10.2 (2024-09-23):** Fixed "empty" and "not empty" filters showing incorrect results.  
- **v2.10.0 (2024-09-20):** Breaking: removed "Group key" property.  
- **v2.9.0 (2024-09-13):** Added Group key for filter groups. (Removed in v2.10.0.)  
- **v2.8.0 (2024-06-19):** Fixed: default filter values restored on initial page load. Added `Set_Filter` action hook. Updated `Reset_Filter` to support reset-to-defaults.  
- **v2.7.0 (2024-03-27):** Fixed custom date format with uppercase 'E'. Added external events hook.  
- **v2.6.0 (2023-08-10):** DOM structure changed to render inline — improved accessibility.  
- **v2.5.0 (2023-05-01):** Breaking: default value is now used as initial value only — changes after mount are ignored.  
- **v2.4.0 (2022-05-11):** Added "empty" and "not empty" filter options.  
- **v2.3.0 (2022-02-10):** Added "between" (range) filter mode.  
- **v2.2.1 (2022-01-06):** Added `date-filter-container` CSS class on the portal div.  
- **v2.0.2 (2021-10-13):** Added `valueAttribute` persistence and `onChange` event.  
- **v2.0.0 (2021-09-28):** Initial public release. Added Gallery compatibility.

**4. Is it user-facing?**  
The documented behaviors are user-facing; the file itself is for developers.

**5. What new did you learn from this file?**  
v2.5.0's "initial value only" default value behavior is explicitly a breaking change — pre-2.5.0 widgets that relied on live default value synchronization would break. v2.10.0 removed the Group key property that was just added in v2.9.0 — this was a short-lived feature. v2.11.2 confirmed that `adjustable=false` protects the filter type from personalization overrides, which is a meaningful constraint for fixed-filter use cases.
