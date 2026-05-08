# Draft: calendar-web

Widget path: `packages/pluggableWidgets/calendar-web/`

---

## src/Calendar.xml

1. **What is the purpose of this file?**
   Mendix widget descriptor. Declares the widget identity (`com.mendix.widget.web.calendar.Calendar`), all configurable properties and their groups, and system-level metadata. The widget is `offlineCapable="true"` and `supportedPlatform="Web"` only (not native). Category: Display.

2. **What kind of logic is described in this file?**
   No runtime logic. Declares five major property groups: Data source (events list, title attribute/expression, allDay, start, end, color), View (editable, view mode, default view, time format, top bar date format, min/max hour, showAllEvents, step, timeslots, startDateAttribute), Custom view (toolbar items list with 9 item types and 3 positions, visible days Mon-Sun), Events (onEditEvent, onCreateEvent with variables, onDragDropResize with variables, onViewRangeChange with variables), and Dimensions (width/height with unit options, min/max height, overflowY).

3. **What part of behavior can be documented from this file?**
   Two view modes: `standard` (day/week/month) and `custom` (day/week/month/work_week/agenda + configurable toolbar). `onCreateEvent` receives three action variables: `startDate`, `endDate`, `allDay`. `onDragDropResize` receives four: `oldStart`, `oldEnd`, `newStart`, `newEnd`. `onViewRangeChange` receives three: `rangeStart`, `rangeEnd`, `currentView`. Event color accepts Enum or String attribute (valid HTML color strings: named, hex, rgb, rgba). `startDateAttribute` is a plain (non-list) DateTime attribute used to control the calendar's initial date.

4. **Is it user-facing?**
   Indirectly — defines the Studio Pro property panel configuration. End-users see the rendered calendar.

5. **What new did you learn from this file?**
   The widget does NOT have `needsEntityContext` — it can be placed anywhere in the page, with its own list datasource (`databaseDataSource` of type `datasource`). `toolbarItems` is a child object list — each item carries its own position, caption, renderMode, buttonStyle, tooltip, and per-view format strings. This makes the toolbar highly composable (each view button is an independent configurable item).

---

## typings/CalendarProps.d.ts

1. **What is the purpose of this file?**
   Auto-generated TypeScript types from Calendar.xml. Provides `CalendarContainerProps` (runtime) and `CalendarPreviewProps` (Studio Pro preview) interfaces, plus all enum and object types.

2. **What kind of logic is described in this file?**
   Runtime types: `databaseDataSource` is `ListValue` (optional), list-scoped data props use `ListAttributeValue`/`ListExpressionValue`/`ListActionValue`. `editable` and `showEventDate` are `DynamicValue<boolean>`. `startDateAttribute` is `EditableValue<Date>` (writable, not just readable). `onCreateEvent` has typed variables via `Option<>` wrappers. `toolbarItems` is `ToolbarItemsType[]` with `DynamicValue<string>` captions/formats.

3. **What part of behavior can be documented from this file?**
   `startDateAttribute` uses `EditableValue<Date>` — the widget can READ from it (to set initial date) and potentially WRITE to it. `onDragDropResize` and `onViewRangeChange` both use `ListActionValue` variants, meaning they receive the event's `ObjectItem` context and action variables simultaneously. `onCreateEvent` is a plain `ActionValue` (not list-scoped) since new events don't exist in the datasource yet. `databaseDataSource` is optional — the calendar can render without events.

4. **Is it user-facing?**
   No — internal type contract.

5. **What new did you learn from this file?**
   `eventColor` is `ListAttributeValue<string>` (not typed as enum in the TypeScript interface even though XML allows Enum). The preview `databaseDataSource` type is `{} | { caption: string } | { type: string } | null` — the standard Mendix preview union for datasource properties.

---

## src/Calendar.tsx (main entry)

1. **What is the purpose of this file?**
   Root widget component. Creates wrapper style and `CalendarPropsBuilder` (once via `useMemo`). Gets locale-aware localizer via `useLocalizer`. Builds calendar props on every prop update. Gets event handlers via `useCalendarEvents`. Renders the DnD Calendar or a loading bar.

2. **What kind of logic is described in this file?**
   Loading state: when `startDateAttribute.status === "loading"`, renders a `<progress>` element (indeterminate loading bar) instead of the calendar. Otherwise renders `DnDCalendar` inside a `.widget-calendar` container div. Uses `useMemo` with empty dependency array for `wrapperStyle` and `calendarController` (one-time initialization — intentional `eslint-disable` comments explain this). Re-computes `calendarProps` on every props/localizer change.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: the calendar shows a progress bar while `startDateAttribute` is loading — this is the only loading state visible to users. When `startDateAttribute` is undefined or available, the calendar renders immediately. The `CalendarPropsBuilder` is constructed once and mutated via `updateProps()` on re-renders — this is deliberate to preserve state such as default date and visible days.

4. **Is it user-facing?**
   Yes — renders the calendar and loading bar visible to end-users.

5. **What new did you learn from this file?**
   The widget uses `react-big-calendar` with the drag-and-drop addon (`withDragAndDrop` wrapping). The loading bar is `<progress className="widget-calendar-loading-bar">` — indeterminate progress styled as a thin animated bar. The `calendarController` is intentionally stable (created once) to avoid resetting calendar state on every Mendix prop change.

---

## src/helpers/CalendarPropsBuilder.ts

1. **What is the purpose of this file?**
   The main props builder class. Translates Mendix `CalendarContainerProps` into `react-big-calendar` props (`DragAndDropCalendarProps`). Handles event mapping, visible views, formats, toolbar, visible days, messages, time range, step/timeslots clamping.

2. **What kind of logic is described in this file?**
   **Constructor**: sets up stable state (visibleDays Set, defaultView, events, minTime/maxTime, toolbarItems, step, timeslots, defaultDate). **`updateProps()`**: updates events, toolbarItems, defaultDate on re-render. **`build()`**: assembles the full RBC props object. **`buildFormats()`**: constructs date format functions based on `timeFormat`, `topBarDateFormat`, and per-toolbar-item custom formats. **`buildVisibleViews()`**: in standard mode returns `{day, week, month}`; in custom mode derives enabled views from toolbar items. **`buildVisibleDays()`**: builds a `Set<number>` (0=Sunday..6=Saturday) from the 7 boolean day props. **`buildTime()`**: creates Date objects for min/max hour. **`buildMessages()`**: sets RBC `messages.work_week` caption and agenda column labels.

3. **What part of behavior can be documented from this file?**
   **Step constraint**: clamped to [1, 60] with `console.warn` if out of range. **Timeslots constraint**: clamped to [1, 4]. **Safe default view**: if `defaultView` is not in the enabled views set, falls back to the first enabled view — prevents RBC crash when a selected default view is not in the toolbar. **Formats**: `showEventDate = false` forces `eventTimeRangeFormat: () => ""` (hides time range in events). **Work week caption**: taken from the `work_week` toolbar item caption, falls back to `"Custom"`. Time format has a fallback: on invalid pattern string, falls back to `"p"` (locale-aware time format token).

4. **Is it user-facing?**
   No — internal builder.

5. **What new did you learn from this file?**
   In custom view mode, visible views are derived purely from toolbar items — if no toolbar items are configured, ALL views are enabled by default. The work_week view is handled specially: it returns a `CustomWeekController.getComponent()` result (a custom RBC view component) rather than `true`. This injects the configurable visible days and title pattern.

---

## src/helpers/CustomWeekController.ts

1. **What is the purpose of this file?**
   Custom RBC view component factory for the configurable work week view. Creates a React component that renders a `TimeGrid` for only the selected weekdays (set via the 7 day checkboxes).

2. **What kind of logic is described in this file?**
   `getComponent()` is a static factory that returns a React function component (compatible with RBC's custom view API) with `navigate`, `title`, and `range` static methods attached. `render()` uses `TimeGrid` (internal RBC component accessed via non-public import) with a custom range from `getRange()`. `title()` formats the visible date range: if contiguous days, shows first-last range; if non-contiguous, shows comma-separated day abbreviations. Navigation moves ±1 week regardless of visible days.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: custom week always navigates in weekly increments (`addWeeks(date, ±1)`) even when only some days of the week are shown. The range is computed from the start of the week containing `date`, then filtered to visible days — so the "current week" anchor is the standard week boundary (Sunday-based), not relative to visible days. If `topBarDateFormat` is configured, it applies to the work_week title as well.

4. **Is it user-facing?**
   Yes (indirectly) — renders the custom week view that end-users see and interact with.

5. **What new did you learn from this file?**
   Uses `react-big-calendar/lib/TimeGrid` via `@ts-expect-error` (not a public export). This is a deliberate coupling to an internal RBC component. The visible days Set uses JS day numbering: 0=Sunday, 1=Monday, ..., 6=Saturday, matching the order in `buildVisibleDays()` (which maps Sun/Mon/Tue/Wed/Thu/Fri/Sat in that order).

---

## src/helpers/useCalendarEvents.ts

1. **What is the purpose of this file?**
   React hook that creates all RBC event handlers: select, double-click, keyboard (Enter), slot selection (create), drag/drop/resize, navigate, range change. Returns a `CalendarEventHandlers` object for spread into the DnDCalendar.

2. **What kind of logic is described in this file?**
   **Single-click vs. double-click**: a 250ms timer deduplicates — single click selects the event (state), second single-click on selected event invokes edit, double-click clears timer and invokes edit. **Create**: `onSelectSlot` triggers `invokeCreate` only if no event is currently selected and `editable === true`. **All-day detection**: `(duration_ms / 86400000) % 1 === 0` — exact multiple of 24h means all-day. **Range change**: `currentViewRef` tracks the active view via `onNavigate` because RBC's `onRangeChange` callback doesn't pass the view name — the ref is synchronized before `handleRangeChange` fires. Supports both array (day/week/custom) and object `{start, end}` (month/agenda) range formats.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: slot selection (create) is blocked when an event is selected — clicking an event then clicking a slot does NOT create a new event. User must deselect first. **All-day detection**: calculated from duration modulo 24h. **onEditEvent**: uses `ListActionValue.get(item)` — only executes if `canExecute` is true (respects Mendix conditional editability). **Memory leak prevention**: timeout ref is cleared on component unmount.

4. **Is it user-facing?**
   No — event handler logic, but drives visible interactive behavior.

5. **What new did you learn from this file?**
   The `currentViewRef` pattern is a known workaround for a react-big-calendar API limitation: `onRangeChange` is called WITHOUT the current view name for month/agenda views. The `onNavigate` fires synchronously before `onRangeChange`, so the ref is always populated. This is explicitly documented in a comment referencing the RBC issue.

---

## src/helpers/useLocalizer.ts

1. **What is the purpose of this file?**
   Creates a `date-fns` localizer for react-big-calendar from Mendix's runtime locale data (`window.mx.session.sessionData.locale`). Provides correct weekday names, month names, AM/PM, first day of week, and date/time format patterns.

2. **What kind of logic is described in this file?**
   `getMendixLocale()` reads from `window.mx.session.sessionData.locale`. `createLocaleFromMendixData()` builds a `date-fns Locale` object: `localize.month` maps to Mendix's `dates.months` and `dates.abbreviatedMonths`; `localize.day` maps to `dates.weekdays` and `dates.shortWeekdays`; `formatLong.date/time/dateTime` use Mendix's `patterns.date/time/datetime`; `options.weekStartsOn` uses Mendix's `firstDayOfWeek`. Falls back to en-US defaults if locale is unavailable.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: the `P`/`p` date-fns tokens (locale-aware date/time) resolve to Mendix's configured locale patterns — so "p" renders time in the user's configured language format. `firstDayOfWeek` from Mendix controls the week start (0=Sunday, 1=Monday, etc.) — affects week/work_week view column order. `localizer` is memoized on `[culture, firstDayOfWeek, mendixLocale]` — updates reactively when locale changes.

4. **Is it user-facing?**
   Indirectly — locale-aware formatting is visible to end-users in date labels, weekday names, and time formats.

5. **What new did you learn from this file?**
   This widget avoids importing external date-fns locale files by constructing a minimal Locale object from Mendix's session data. This means no locale bundles need to be included in the widget package — locale data is always current with the running Mendix application's language settings. The `dayPeriod` localization handles AM/PM from `dates.dayPeriods[0/1]`.

---

## src/components/Toolbar.tsx

1. **What is the purpose of this file?**
   Defines two toolbar components: `CustomToolbar` (default, fixed 3-section layout) and `createConfigurableToolbar()` (factory for custom view mode with per-item configuration). Also defines `ResolvedToolbarItem` type.

2. **What kind of logic is described in this file?**
   `CustomToolbar`: renders prev/today/next on left, date label in center, view buttons on right. Uses `@mendix/widget-plugin-component-kit/Button` and `IconInternal` (glyph icons) for prev/next. `createConfigurableToolbar()`: groups items by position (left/center/right), renders each in order. Supports 9 item types. Button styles: `default/primary/success/info/warning/danger`. Render modes: `button` (btn-default/btn-primary/etc) vs `link` (btn-link). Title item always shows the date label (ignores caption). Today button always shows localizer message as fallback. View buttons show configured caption or localizer message.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: the `title` item type always renders the calendar date label from RBC — its `caption` property is ignored (hardcoded to use `{label}`). **Render mode `link`**: forces `btn-link` CSS class regardless of `buttonStyle` setting. **Active state**: view buttons get `active` CSS class when `view === name`. All toolbar buttons use Mendix's Button component (not native `<button>`) and IconInternal for glyphs.

4. **Is it user-facing?**
   Yes — the toolbar is the primary navigation UI visible to end-users.

5. **What new did you learn from this file?**
   The toolbar CSS uses a 3-column CSS grid layout (`grid-template-columns: 1fr auto 1fr`) with left/center/right sections. This means center content is exactly centered regardless of left/right content width. The `createConfigurableToolbar` is a closure factory — it captures the `items` array and returns a stable React component function. The `work_week` view button triggers `onView("work_week")` which RBC routes to the custom `CustomWeekController` component.

---

## src/utils/calendar-utils.ts

1. **What is the purpose of this file?**
   Shared utilities: exports `DnDCalendar` (react-big-calendar + DnD addon), `eventPropGetter` (event color styling), `getRange` (custom week day range), `getTextValue` (empty string normalization), `getMendixLocale` (Mendix session locale access), date-fns re-exports, and a default localizer export.

2. **What kind of logic is described in this file?**
   `DnDCalendar`: `withDragAndDrop(Calendar<CalendarEvent>)`. `eventPropGetter`: returns `{ style: { backgroundColor } }` using `event.color`; if selected, lightens by 25% via `lightenColor()`. `lightenColor()`: only processes `#RGB` and `#RRGGBB` hex colors; falls back to original for non-hex. `getRange()`: uses `startOfWeek` (Sunday=0), iterates 7 days, filters to `visibleDays` Set. Imports `react-big-calendar` CSS and DnD CSS directly. `getMendixLocale()`: reads `window.mx.session.sessionData.locale` with try/catch.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: `lightenColor` only supports hex colors (`#` prefix, 3 or 6 chars). RGB, RGBA, named colors, and other formats are returned unchanged (no lightening on selection). This means selection highlighting only works visually for hex color values in `eventColor`. **`getRange`**: always starts from Sunday (weekStartsOn: 0) regardless of `firstDayOfWeek` — the custom week range is Sunday-anchored.

4. **Is it user-facing?**
   No — utility layer, but `eventPropGetter` drives visible event color rendering.

5. **What new did you learn from this file?**
   The default `localizer` exported from this file uses empty `locales: {}` — it's not locale-aware. The actual runtime uses `useLocalizer()` which creates a proper locale-aware instance. The export exists for legacy compatibility or the preview component. `react-big-calendar/lib/css/react-big-calendar.css` and DnD addon CSS are imported here — they apply globally via module system.

---

## src/utils/style-utils.ts

1. **What is the purpose of this file?**
   `constructWrapperStyle()` function that maps the widget's dimension props to a CSS `CSSProperties` object for the outer wrapper div.

2. **What kind of logic is described in this file?**
   Width: always set as `{width}{px|%}`. Height: when `heightUnit === "percentageOfWidth"` (the "Auto" option), height is `"auto"` and min/max height are applied instead. When other height units: height is set directly. `percentageOfParent` → `%`, `percentageOfView` → `vh`, `pixels` → `px`. User-provided `style` is spread first, then dimension-derived styles override.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: when `heightUnit === "percentageOfWidth"`, the `height` property is ignored — only `minHeight`/`maxHeight`/`overflowY` apply (when their units are not "none"). When `maxHeightUnit !== "none"`, `overflowY` is also set — so overflow only applies in "auto height" mode with a max height constraint. The user's inline `style` prop is applied but can be overridden by the dimension settings.

4. **Is it user-facing?**
   Yes (indirectly) — controls the calendar container size visible to end-users.

5. **What new did you learn from this file?**
   The `heightUnit: "percentageOfWidth"` is labeled "Auto" in the UI — it means height is determined by content, not a fixed value. This mode enables a responsive height pattern where min/max constrain the range. The function is also used in `editorPreview.tsx` for the Studio Pro preview rendering (exported `WrapperStyleProps` type is referenced there).

---

## src/utils/typings.ts

1. **What is the purpose of this file?**
   Defines `CalendarEvent`, `EventDropOrResize`, and `DragAndDropCalendarProps` interfaces used throughout the widget.

2. **What kind of logic is described in this file?**
   `CalendarEvent`: `{ title, start, end, allDay, color?, item: ObjectItem }`. `EventDropOrResize`: `{ event: CalendarEvent, start: Date, end: Date }`. `DragAndDropCalendarProps`: extends both `ReactCalendarProps` and `withDragAndDropProps`.

3. **What part of behavior can be documented from this file?**
   `CalendarEvent.color` is optional — events without a color use RBC's default styling. `CalendarEvent.item` holds the original Mendix `ObjectItem` — this is how `onEditEvent.get(event.item)` works (resolving the ListActionValue). `EventDropOrResize` captures only start/end (new values) + the event itself (which carries old start/end).

4. **Is it user-facing?**
   No — internal type definitions.

5. **What new did you learn from this file?**
   The `CalendarEvent.item` field bridges the gap between react-big-calendar's event model and Mendix's ObjectItem system. Without it, edit/drag actions couldn't be resolved back to the correct Mendix entity instance. This is the core integration pattern for list-scoped actions in Mendix pluggable widgets.

---

## src/Calendar.editorConfig.ts

1. **What is the purpose of this file?**
   Studio Pro editor configuration: `getProperties()` hides irrelevant props based on view mode and dimension settings, `getPreview()` returns a structure preview (calendar icon + "Calendar" label).

2. **What kind of logic is described in this file?**
   **Height mode visibility**: `percentageOfWidth` (Auto) hides `height`; other modes hide min/max/overflow controls. `minHeight` hidden when `minHeightUnit === "none"`, `maxHeight`/`overflowY` hidden when `maxHeightUnit === "none"`. **View mode**: standard mode hides Custom view props; custom mode hides `defaultViewStandard`, `topBarDateFormat`, `timeFormat`. **Per-toolbar-item visibility**: title items hide button-specific props. Day/week/work_week items hide cell/gutter/text headers. Month items hide gutter/agenda-text. Agenda items hide cellDateFormat. **Title type**: attribute mode hides expression, expression mode hides attribute. **Preview**: renders a mini calendar icon with "Calendar" text using structure-preview-api.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraints from visibility rules**: `topBarDateFormat` and `timeFormat` are NOT available in custom view mode (hidden) — custom view uses per-toolbar-item formats instead. `defaultViewStandard` is hidden in custom mode; `defaultViewCustom` is hidden in standard mode. The `toolbarItems` list is hidden entirely in standard view mode. Toolbar item formats are scoped: gutter formats appear only for time-grid views (day/week/work_week), not month.

4. **Is it user-facing?**
   Studio Pro designers only.

5. **What new did you learn from this file?**
   `getPreview()` uses `@mendix/widget-plugin-platform/preview/structure-preview-api` — the calendar shows a tiny calendar SVG icon next to the text "Calendar" in the structure preview. Two separate SVG assets exist for light (`StructureCalendarLight.svg`) and dark (`StructureCalendarDark.svg`) Studio Pro themes.

---

## src/Calendar.editorPreview.tsx

1. **What is the purpose of this file?**
   Studio Pro canvas preview. Renders a live react-big-calendar with hardcoded sample events (Leave, BD, Bank Holiday) and static localizer (no Mendix locale). Uses the configured toolbar and dimensions.

2. **What kind of logic is described in this file?**
   Creates a `dateFnsLocalizer` with empty locales (English defaults). Generates 6 sample events relative to today (past and future dates). Renders `<Calendar>` from react-big-calendar (not the DnD wrapper) with the sample events, configured toolbar (configurable in custom view mode), and wrapper style from `constructWrapperStyle`. Views list is fixed (standard: day/week/month; custom: day/week/month/work_week).

3. **What part of behavior can be documented from this file?**
   Studio Pro preview shows a real calendar UI with sample events — developers see the actual layout, not just an icon. Preview uses static locale (English) regardless of the app's configured language. The preview does NOT use the DnD addon — sample events cannot be dragged. Toolbar in custom view mode reflects the configured toolbar items and their positions/captions.

4. **Is it user-facing?**
   Studio Pro designers only.

5. **What new did you learn from this file?**
   The preview's sample events use large date offsets (e.g., `+9000 * 3600 * 24` hours ≈ 375 days from now) — this ensures events are visible across multiple months in the preview. The preview always shows `work_week` caption as `"Custom"` (hardcoded in `messages`). `editorPreview.tsx` imports both `Calendar.scss` and `react-big-calendar/lib/css/react-big-calendar.css` — the full styling is applied in preview as well.

---

## src/__tests__/Calendar.spec.tsx

1. **What is the purpose of this file?**
   Unit test suite for the Calendar widget using Jest and React Testing Library. Tests: basic rendering, class names, step/timeslots passing, loading bar behavior, startDateAttribute states, defaultDate passing. Plus `CalendarPropsBuilder` validation tests for step/timeslots clamping.

2. **What kind of logic is described in this file?**
   Mocks: react-big-calendar `Calendar` component (renders as `<div data-testid="mock-calendar">` with data attributes), `dateFnsLocalizer` (returns empty mock), DnD addon (identity function). Tests use `ListValueBuilder` for empty datasource. Fake timers set to `2025-04-28T12:00:00Z`. CalendarPropsBuilder clamping tests cover: step/timeslots of 0, negative, above max, boundary values.

3. **What part of behavior can be documented from this file?**
   Loading bar (`<progress>`) shown only when `startDateAttribute.status === "loading"`. When status is `"available"`, `"unavailable"`, or `undefined` — calendar renders without loading bar. `defaultDate` prop is passed from `startDateAttribute.value` when available. Step clamping confirmed: [1, 60]. Timeslots clamping confirmed: [1, 4]. CSS class test confirms both `.widget-calendar` and the developer's custom `class` are applied.

4. **Is it user-facing?**
   No — test file.

5. **What new did you learn from this file?**
   `CalendarPropsBuilder.build()` is tested directly (not via the widget component) for clamping logic. The mock localizer in tests has empty `messages` object — this confirms the test suite does not test toolbar label rendering. The snapshot test (`renders correctly with basic props`) captures the full rendered structure — this is the primary regression guard.

---

## typings/global.d.ts

1. **What is the purpose of this file?**
   TypeScript global type declarations for `window.mx` session data structure. Provides types for Mendix's runtime session locale object.

2. **What kind of logic is described in this file?**
   Declares interfaces: `MXLocalePatterns` (date/datetime/time format strings), `MXLocaleDates` (weekdays/shortWeekdays/months/shortMonths/abbreviatedMonths/abbreviatedShortMonths/dayPeriods/eras arrays), `MXLocaleNumbers` (separators), `MXSessionLocale`, `MXSessionData`, `MXSession`, `MXGlobalObject`. Augments global `Window` interface with `mx?: { session?: MXSession }`.

3. **What part of behavior can be documented from this file?**
   `MXLocaleDates` has 8 arrays including both `shortMonths` and `abbreviatedMonths` (separate concepts in Mendix). `dayPeriods` is an array (index 0 = AM equivalent, 1 = PM equivalent). `firstDayOfWeek` is a number (0–6). The `mx` property and `session` are both optional — `getMendixLocale()` must handle `undefined` gracefully.

4. **Is it user-facing?**
   No — TypeScript type definitions for the Mendix global object.

5. **What new did you learn from this file?**
   This file is the authoritative contract for the Mendix session locale API. It reveals that Mendix provides both `shortMonths` (e.g., "Jan") and `abbreviatedMonths` (shorter variants) — the widget uses `abbreviatedMonths` for the `abbreviated` width in `localize.month()`. The `eras` and `numbers` fields are declared but not used by the calendar widget.

---

## src/ui/Calendar.scss

1. **What is the purpose of this file?**
   Widget stylesheet. Wraps all calendar styles inside `.widget-calendar`. Overrides react-big-calendar default styles for events, today highlighting, off-range backgrounds, resize cursors, toolbar layout, and loading bar animation.

2. **What kind of logic is described in this file?**
   `min-height: 350px` enforced on `.widget-calendar`. Imports Sass `color.mix()` for lighter color variants. Toolbar: CSS grid with `1fr auto 1fr` columns. Loading bar: a styled `<progress>` element with cross-browser `-webkit-`/`-moz-`/`-ms-` prefixes and an `@keyframes progress-linear` animation (3-step indeterminate animation with background-position/size tricks). Event selection: `filter: brightness(0.85)` on `.rbc-selected`. Today: `--color-primary-lighter` background. Off-range: `--color-default-lighter` background.

3. **What part of behavior can be documented from this file?**
   **Behavioral constraint**: the calendar has a hard minimum height of 350px — it cannot be shorter regardless of `minHeight` configuration. Events have `min-height: 20px`. The loading bar animation uses a pure CSS technique (no JS animation) with 3 keyframes producing an indeterminate progress effect. Uses Mendix CSS custom properties (`--brand-primary`, `--brand-danger`, `--spacing-medium`, `--font-size-default`, etc.) for theme integration.

4. **Is it user-facing?**
   Yes — defines all visual styles for the calendar widget.

5. **What new did you learn from this file?**
   The `user-select: none` rule (`-ms-`, `-webkit-`, `user-select`) on the root prevents accidental text selection during drag operations. The `rbc-addons-dnd-resize-ns-icon` and `rbc-addons-dnd-resize-ew-icon` classes set `cursor: row-resize` and `cursor: col-resize` respectively — these are the resize handle cursors shown during event drag-resize.

---

## CHANGELOG.md

1. **What is the purpose of this file?**
   Version history following Keep a Changelog / Semantic Versioning.

2. **What kind of logic is described in this file?**
   Four releases: v2.0.0 (2025-08-12), v2.2.0 (2025-11-11), v2.3.0 (2026-02-17), v2.4.0 (2026-03-20).

3. **What part of behavior can be documented from this file?**
   **v2.0.0** (2025-08-12): Initial v2 release. **Breaking change**: upgrading from v1.x requires reconfiguring the widget in Studio Pro (property panel was reorganized). **v2.2.0** (2025-11-11): Added configurable toolbar items with captions, tooltips, link-style appearance. Added consistent cross-view title formatting. Time formatting applied to time gutter AND event/agenda ranges. `showEventDate: false` no longer shows start/end time text. Fixed localization (dates, weekday/month names). Fixed custom format patterns. Fixed empty format field error. Fixed abbreviated month names reverting to English. Fixed default view error when view not in toolbar. Fixed title expression not rendering correctly. **Breaking change**: custom view captions now set inside `toolbarItems` config. **v2.3.0** (2026-02-17): Added `step` and `timeslots` properties. Fixed `onViewRangeChange` not triggering when switching from Day/Week to Month. **v2.4.0** (2026-03-20): Fixed `startDateAttribute` handling for correct initialization.

4. **Is it user-facing?**
   No — developer/release notes.

5. **What new did you learn from this file?**
   The widget underwent a major API break at v2.0.0. The v2.2.0 breaking change (toolbar captions moved into toolbarItems) means any app using v2.1.x-era custom captions needs reconfiguration. The `onViewRangeChange` bug (Month view not getting correct range) was specifically fixed in v2.3.0 — this was the motivation for the `currentViewRef` workaround in `useCalendarEvents.ts`. The `startDateAttribute` initialization fix in v2.4.0 aligns with the loading-state progress bar shown in `Calendar.tsx`.
