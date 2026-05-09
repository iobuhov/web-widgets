# Draft: timeline-web

Widget package: `packages/pluggableWidgets/timeline-web`

---

## src/Timeline.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, props schema, and Studio Pro categorization. Generates `TimelineProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares: `data` (ListValue datasource); `title` (ListExpressionValue<string>, required in basic mode); `description`, `timeIndication` (ListExpressionValue<string>, optional); `customVisualization` (boolean, default false); `icon` (ListAttributeValue<WebIcon>, optional); grouping properties (`groupEvents`, `groupAttribute: ListAttributeValue<Date>`, `groupByKey: day/month/year`, `groupByDayOptions`, `groupByMonthOptions`, `ungroupedEventsPosition: beginning/end`); custom visualization slots (`customIcon`, `customGroupHeader`, `customTitle`, `customEventDateTime`, `customDescription` — all ListWidgetValue); `onClick: ListActionValue`. Widget is `needsEntityContext="true"`, `offlineCapable="true"`.

**3. What part of behavior can be documented from this file?**
- Basic mode and custom mode are mutually exclusive — `customVisualization=false` uses text fields; `customVisualization=true` uses widget content slots.
- Three grouping levels with format options: day (dayName, dayMonth, fullDate), month (month, monthYear), year.
- Events without a `groupAttribute` value are "ungrouped" and can be placed at the beginning or end of the list.
- Each data item can trigger an `onClick` action — the action runs in the context of that item.
- Custom visualization supports a separate `customGroupHeader` slot for fully custom group header rendering.

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
The `customGroupHeader` slot is exclusive to custom visualization mode — basic mode does not support custom group headers. This means custom mode provides complete control over every visual element of the timeline, including group separators.

---

## src/Timeline.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. Routes between `BasicView` and `CustomView` based on the `customVisualization` prop.

**2. What kind of logic is described in this file?**
A simple conditional: when `customVisualization=false`, renders `withBasicItems(TimelineComponent)`; when `customVisualization=true`, renders `withCustomItems(TimelineComponent)`. This is the only logic — all data transformation is in the HOCs and hooks.

**3. What part of behavior can be documented from this file?**
- The two visualization modes are selected at render time, not at configuration compile time — they share the same `TimelineComponent` but inject different data via different HOCs.
- CSS styling is imported once at this root level.

**4. Is it user-facing?**
No — internal routing component.

**5. What new did you learn from this file?**
The HOC pattern (withBasicItems / withCustomItems) wrapping the same `TimelineComponent` is a clean architectural choice — `TimelineComponent` is mode-agnostic; the HOCs adapt different data shapes into the same `TimelineData` structure.

---

## src/components/TimelineComponent.tsx

**1. What is the purpose of this file?**
The core presentation component rendering the visual timeline — grouped event lists, group headers, icons, and interactive event items.

**2. What kind of logic is described in this file?**
`getItems()`: Iterates the `TimelineData` map (grouped event map, keyed by date string). For each group: renders a `<li>` header element if `groupEvents=true` and the group key is non-empty. Then renders event items by calling `getBasicEventsFromDay()` or `getCustomEventsFromDay()`. Ungrouped events (empty key) are rendered at the beginning or end based on `ungroupedEventsPosition`. Events get `"clickable"` CSS class when `onClick` is defined. `hasChildren(element)`: returns `true` only if the React element has non-empty children — prevents rendering empty custom elements.

**3. What part of behavior can be documented from this file?**
- Group headers are rendered only when `groupEvents=true` AND the group key is non-empty (ungrouped events get no header).
- Events get CSS class `"clickable"` when they have an `onClick` action — enabling cursor and hover styles.
- In custom visualization, `hasChildren()` prevents rendering empty `<li>` elements for custom slots that have no content.
- Basic mode renders: Mendix `Icon` component, title span, description span, event time span.
- Custom mode renders: whatever ReactNode content was provided per-item.

**4. Is it user-facing?**
Yes — this is the visible timeline UI.

**5. What new did you learn from this file?**
The `hasChildren()` check on custom elements (checking `React.Children.count(element.props.children) > 0`) prevents rendering empty `<li>` items when custom widget slots have no content configured. This graceful handling prevents visual artifacts in the timeline for partially-configured custom items.

---

## src/hocs/withBasicItems.tsx

**1. What is the purpose of this file?**
Higher-order component that injects `useBasicItems`-transformed data into `TimelineComponent` for basic visualization mode.

**2. What kind of logic is described in this file?**
Calls `useBasicItems(props)` to get the `TimelineData` map, passes it to `TimelineComponent` with `customVisualization=false`. Threads through `className`, `groupEvents`, `ungroupedEventsPosition`.

**3. What part of behavior can be documented from this file?**
- `customVisualization` is hardcoded to `false` in this HOC — the wrapped component always renders in basic mode.
- Data transformation (from Mendix `ListValue` to `TimelineData`) happens in `useBasicItems`, not here.

**4. Is it user-facing?**
No — internal HOC.

**5. What new did you learn from this file?**
The HOC is intentionally thin — it delegates to the hook for data transformation and to the component for rendering. This maintains single responsibility at each layer.

---

## src/hocs/withCustomItems.tsx

**1. What is the purpose of this file?**
Higher-order component that injects `useCustomItems`-transformed data into `TimelineComponent` for custom visualization mode.

**2. What kind of logic is described in this file?**
Symmetric to `withBasicItems`: calls `useCustomItems(props)` and passes resulting `TimelineData` to `TimelineComponent` with `customVisualization=true`.

**3. What part of behavior can be documented from this file?**
- `customVisualization=true` is hardcoded in this HOC.
- Custom widget content (ReactNodes) per item is wrapped in the `customItem()` shape.

**4. Is it user-facing?**
No — internal HOC.

**5. What new did you learn from this file?**
The parallel structure between `withBasicItems` and `withCustomItems` is intentional — both HOCs are interchangeable adapters for the same `TimelineComponent`. Adding a third visualization mode would require adding a third HOC, following the same pattern.

---

## src/hooks/useBasicItems.ts

**1. What is the purpose of this file?**
React hook transforming Mendix `ListValue` into a `TimelineData` (Map) structure for basic visualization mode.

**2. What kind of logic is described in this file?**
`useBasicItems(props)`: `useMemo` wrapping `reduceData(props)`. `reduceData`: iterates `props.data.items`, calls `getGroup(item, props)` for the group key, pushes `basicItem(item, props)` to the map entry for that key. `basicItem(item)`: extracts `icon.get(item).value`, `title.get(item).value`, `timeIndication.get(item).value`, `description.get(item).value`, `onClick.get(item)`.

**3. What part of behavior can be documented from this file?**
- The static icon is retrieved via `icon.value` (not per-item) — all events in basic mode share the same icon.
- Expression values (`title`, `description`, `timeIndication`) are per-item via `.get(item)`.
- The `onClick` action is per-item via `onClick.get(item)`.
- `useMemo` dependencies include all data properties — memo cache is invalidated when any data changes.

**4. Is it user-facing?**
No — internal data transformation hook.

**5. What new did you learn from this file?**
The icon is `icon.value` (not `icon.get(item).value`) — confirmed that icons are configuration-level (same for all items) in basic mode. Each item could have a different title/description/time but not a different icon. Custom mode does support per-item icons via `customIcon.get(item)`.

---

## src/hooks/useCustomItems.ts

**1. What is the purpose of this file?**
React hook transforming Mendix `ListValue` into `TimelineData` for custom visualization mode.

**2. What kind of logic is described in this file?**
Parallel to `useBasicItems` but extracts widget content per item: `customIcon.get(item)`, `customGroupHeader.get(item)`, `customTitle.get(item)`, `customEventDateTime.get(item)`, `customDescription.get(item)`. All are `ReactNode` instances.

**3. What part of behavior can be documented from this file?**
- ALL content in custom mode is per-item (via `.get(item)`) — icon, group header, title, date/time, description.
- Custom mode supports `customGroupHeader` — a per-item widget slot that renders as the group separator header.
- `useMemo` dependencies include all custom widget list values.

**4. Is it user-facing?**
No — internal data transformation hook.

**5. What new did you learn from this file?**
`customGroupHeader` is retrieved per-item but represents the group header — this means the developer must configure a consistent group header widget for all items in the same group. The widget renders the header from the first item in each group (or the most recently processed one, based on how the map is populated).

---

## src/helpers/grouping.ts

**1. What is the purpose of this file?**
Utility functions for date-based grouping — computing group keys from dates using locale-aware formatting.

**2. What kind of logic is described in this file?**
`getGroupingMethodForBasicMode(groupByKey, options)`: selects from 6 format methods (dayName, dayMonth, fullDate, month, monthYear, year) based on the groupByKey and format options. `getGroupingMethodForCustomMode(groupByKey, options)`: same but upgrades `month` → `monthYear` for custom mode. `getFormatConfigByGroupingMethod(method)`: returns a `DateTimeFormatterConfig` with custom patterns (e.g., `"EEEE"` for weekday name, `"d MMMM"` for day+month). `groupKey(item, props)`: formats the item's `groupAttribute` date using Mendix's `DateTimeFormatter`. `getGroup(item, props)`: returns `groupKey()` when grouping is enabled, empty string otherwise.

**3. What part of behavior can be documented from this file?**
- Day grouping format options: `dayName` (`"EEEE"` — full weekday name), `dayMonth` (`"d MMMM"`), `fullDate` (`"d MMMM yyyy"`).
- Month grouping format options: `month` (`"MMMM"`), `monthYear` (`"MMMM yyyy"`).
- Year grouping: `"yyyy"`.
- Custom mode uses `monthYear` instead of `month` — to avoid ambiguous month labels across years.
- Ungrouped items (no `groupAttribute`) get empty string as key — rendered separately.
- Date formatting uses Mendix's `DateTimeFormatter` — locale-aware output.

**4. Is it user-facing?**
No — internal grouping logic. Affects the group header labels users see.

**5. What new did you learn from this file?**
The custom mode month-to-monthYear upgrade prevents a bug: if events span multiple years and are grouped by month only, "January" could appear twice (once per year). Custom mode forces `monthYear` to disambiguate. This difference between basic and custom mode grouping is subtle but important for multi-year datasets.

---

## src/ui/Timeline.scss

**1. What is the purpose of this file?**
Core stylesheet for the timeline widget — layout, colors, timeline line/dot, event spacing, hover states.

**2. What kind of logic is described in this file?**
`.widget-timeline-date-header`: 110px width, pill shape (30px border-radius), bordered. `.widget-timeline-events-wrapper`: flex container, left margin for line alignment. `.widget-timeline-event`: left border (the timeline line), padding, hover/focus states. `.widget-timeline-icon-wrapper`: absolutely positioned at -50% transform to center icon on the timeline line. `.widget-timeline-icon-circle`: 18px circle, blue (`#264ae5`) background — default dot when no icon is configured. `.clickable`: hover changes title color to a lighter gray. `.no-divider`: removes the left border and margins.

**3. What part of behavior can be documented from this file?**
- The timeline "line" is the `border-left` of `.widget-timeline-event` — a continuous left border connecting events visually.
- The dot/icon is absolutely positioned with `transform: translateY(-50%)` to center it on the timeline line.
- Default icon (when no custom icon): 18px blue circle (`#264ae5`).
- Clickable events: hover changes title to lighter gray — subtle visual feedback.
- `.no-divider` class removes the timeline line and margins — useful for ungrouped or last-item scenarios.
- Primary colors: link blue `#264ae5`, text dark `#0a1325`, detail gray `#6c717c`, border `#dadcde`.

**4. Is it user-facing?**
Yes — all visual appearance is defined here.

**5. What new did you learn from this file?**
The entire timeline "line" connecting events is implemented as a continuous CSS `border-left` on each event item — not as a separate `<div>` element. This means the line appears and disappears based on the presence of `.widget-timeline-event` elements, and can be removed selectively via `.no-divider`.

---

## typings/TimelineProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `Timeline.xml`. Defines `TimelineContainerProps` (runtime) and `TimelinePreviewProps` (Studio design-mode).

**2. What kind of logic is described in this file?**
Enumerations: `GroupByKeyEnum` (day/month/year), `GroupByDayOptionsEnum` (dayName/dayMonth/fullDate), `GroupByMonthOptionsEnum` (month/monthYear), `UngroupedEventsPositionEnum` (beginning/end). Container props: `data: ListValue`, `title: ListExpressionValue<string>`, `icon: ListAttributeValue<WebIcon>`, `groupAttribute: ListAttributeValue<Date>`, custom slots as `ListWidgetValue`, `onClick: ListActionValue`. Preview props: string representations, renderers with caption.

**3. What part of behavior can be documented from this file?**
- `icon` is `ListAttributeValue<WebIcon>` — an attribute on the datasource entity, not a static configuration value. However, from the implementation, it's read as `icon.value` (not per-item), suggesting it's configured as a constant expression.
- `onClick` is `ListActionValue` — per-item action.
- All five custom slots (`customIcon`, `customGroupHeader`, `customTitle`, `customEventDateTime`, `customDescription`) are `ListWidgetValue` — per-item widget content.

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
`icon` is typed as `ListAttributeValue<WebIcon>` but used as a static value (`icon.value`). This means the icon is typically configured as a constant icon in Mendix (same icon for all items), even though the type technically allows per-item icon expressions.

---

## src/components/__tests__/TimelineComponent.spec.tsx

**1. What is the purpose of this file?**
Unit tests for `TimelineComponent` using Jest and React Testing Library.

**2. What kind of logic is described in this file?**
Tests: snapshot for basic configuration; snapshot for custom configuration; group header hidden when `groupEvents=false`; click handler executes action; hover state applies styling; icon via WebIcon works; custom ReactNode icon works; empty custom elements are not rendered (hasChildren guard).

**3. What part of behavior can be documented from this file?**
- When `groupEvents=false`, no `<li>` header elements are rendered.
- Click events invoke the `ActionValue.execute()` method.
- `hasChildren` guard: empty custom elements produce no rendered `<li>`.
- Snapshot confirms DOM structure including group headers, event items, and icons.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The tests confirm that group headers and event items are both `<li>` elements in an outer `<ul>` — the timeline is semantically a list, not a custom layout. This is accessible by default (screen readers announce it as a list).

---

## e2e/timeline.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the Timeline widget in a real Mendix application.

**2. What kind of logic is described in this file?**
Tests: basic mode screenshot comparison (0.2 threshold); basic mode click triggers modal ("Event called"); custom mode screenshot comparison; custom mode click triggers modal. Session logout after each test.

**3. What part of behavior can be documented from this file?**
- Both basic and custom modes are e2e-tested independently.
- Click events are confirmed to trigger Mendix actions (modal with "Event called").
- Screenshot threshold of 0.2 (20%) allows minor rendering differences.
- Tests navigate to separate pages: `basicTimelinePage` and `customTimelineLayoutGrid`.

**4. Is it user-facing?**
The tested behaviors (timeline rendering, click actions) are user-facing.

**5. What new did you learn from this file?**
The e2e tests for both modes are structurally identical — same test pattern (screenshot + click verification), different page names. This confirms that the widget's two modes produce the same interaction contract (clickable events with action support) with only visual differences.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the timeline-web widget.

**2. What kind of logic is described in this file?**
Key versions: 3.2.3 (2026-02-10, license docs), 3.2.2 (2024-06-26, fix: data retrieval via association), 3.2.1 (2023-09-27, redundant code removal), 3.2.0 (2023-06-06, caption/icon/tile updates), 3.1.1 (2022-12-12, fix: month grouping in custom mode), 3.1.0 (2021-12-23, dark mode preview), 3.0.1 (2021-10-06, design property fix), 3.0.0 (2021-09-28, initial major release).

**3. What part of behavior can be documented from this file?**
- v3.2.2 fixed data retrieval via Mendix association — previously, data accessed through an association (not direct entity) may not have loaded correctly.
- v3.1.1 fixed month grouping in custom mode — the monthYear format for custom mode (documented in grouping.ts) was added/fixed in this release.
- No functional changes since v3.0.0 — only bug fixes and maintenance.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The v3.1.1 month grouping fix for custom mode confirms that the basic/custom grouping method difference (month vs monthYear) was introduced as a bug fix, not original design. The bug was that custom mode was showing ambiguous month labels for multi-year data.
