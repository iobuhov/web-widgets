# Draft: events-web

Widget package: `@mendix/app-events-web` v1.3.0  
Source path: `packages/pluggableWidgets/events-web/`

---

## Events.xml

**1. What is the purpose of this file?**  
This is the Mendix widget definition file. It declares the widget's identity (`com.mendix.widget.web.events.Events`), its offline and web platform support, and all configurable properties exposed to the Studio Pro editor. It is the authoritative source of truth for property names, types, defaults, and grouping.

**2. What kind of logic is described in this file?**  
The file declares two event groups: "Component load" and "On change". Component load properties control when an action fires after mount (delay, repeat toggle, repeat interval). On-change properties control what attribute to watch and how long to debounce the response action. Each timing property has a dual-mode input: a static integer or an expression. The `onEventChangeAttribute` accepts all Mendix data types: AutoNumber, Binary, Boolean, DateTime, Enum, HashString, Integer, Long, String, Decimal.

**3. What part of behavior can be documented from this file?**  
The widget fires an action on component mount with a configurable delay (default 0 ms, meaning immediately). Repeating is opt-in (`componentLoadRepeat` default false); when enabled, the interval defaults to 30 000 ms. A second behavior path watches a bound attribute for value changes and fires a separate action with its own delay. Both delay and interval parameters support static values or dynamic expressions returning an Integer.

**4. Is it user-facing?**  
Indirectly. Studio Pro users configure these properties in the widget panel. End users never interact directly with this widget — it renders no visible UI. It is an invisible event-dispatch mechanism.

**5. What new did you learn from this file?**  
The attribute accepted by `onEventChangeAttribute` spans every Mendix primitive type including Binary and HashString, which is broader than typical value-display widgets. The parameter-type enum pattern (value vs. expression) is used consistently for all three timing parameters (load delay, repeat interval, change delay), enabling runtime-dynamic timing.

---

## typings/EventsProps.d.ts

**1. What is the purpose of this file?**  
This is the auto-generated TypeScript type file derived from Events.xml. It provides strongly-typed interfaces (`EventsContainerProps`, `EventsPreviewProps`) consumed by the runtime component and editor preview. It must not be edited manually — it is regenerated from Events.xml by the Mendix Widgets Framework toolchain.

**2. What kind of logic is described in this file?**  
The file formalises all property types for both runtime and preview contexts. Runtime props use Mendix SDK types: `ActionValue` (actions), `DynamicValue<Big>` (expressions returning numeric values via big.js), and `EditableValue<Big | any | boolean | Date | string>` (attribute binding). Preview props replace SDK types with simpler representations (string for attribute, `{} | null` for actions) to allow editor-only rendering.

**3. What part of behavior can be documented from this file?**  
`componentLoadDelayExpression` is typed as non-optional `DynamicValue<Big>` — it is always present even when parameterType is "number", so the hook must handle its potentially non-available status gracefully. `componentLoadRepeatIntervalExpression` is typed optional (`?`), meaning it may be absent entirely. Delay expression values are `Big` (arbitrary-precision decimal), converted to `number` by calling `.toNumber()`.

**4. Is it user-facing?**  
No. This is an internal contract file between the widget XML and TypeScript source.

**5. What new did you learn from this file?**  
Numeric expression values use `big.js` (`Big`) rather than plain `number`, requiring an explicit `.toNumber()` conversion. The `onEventChangeAttribute` union type `EditableValue<Big | any | boolean | Date | string>` confirms broad attribute type support at the TypeScript level — the `any` in the union is a fallback for types without a more specific TypeScript mapping.

---

## src/Events.tsx

**1. What is the purpose of this file?**  
This is the main runtime React component for the Events widget. It orchestrates all event behaviors by composing three hooks, then renders a single empty div as its output. The component is the entry point called by the Mendix runtime on every render.

**2. What kind of logic is described in this file?**  
For each timing parameter pair (type + value/expression), the component calls `useParameterValue` to resolve the actual `number | undefined`. It then passes the resolved delay to `useOnLoadTimer` (component-load event) and the resolved delay to `useAttributeMonitor` (attribute-change event). The `canExecute` flag is derived from `action.canExecute && !action.isExecuting`, preventing re-entrant execution.

**3. What part of behavior can be documented from this file?**  
The widget renders only `<div className="widget-events" />` with no children — it is intentionally invisible. The `canExecute` guard (`!action.isExecuting`) ensures an action is never triggered while a previous execution is still running. When `onComponentLoad` is undefined, `canExecute` is hardcoded to `false`, so no timer fires. When `onEventChange` is undefined, the attribute monitor is inactive.

**4. Is it user-facing?**  
The rendered output is a zero-content div. Users do not see it directly, though it occupies space in the DOM (flex-grows per its CSS).

**5. What new did you learn from this file?**  
The `canExecute` guard is applied in the component, not in the hooks — keeping the hooks generic and reusable. The `useAttributeMonitor` hook receives `attribute` as part of its props, meaning the hook is responsible for detecting value changes rather than the component computing a diff itself.

---

## src/Events.editorConfig.ts

**1. What is the purpose of this file?**  
This file configures the Studio Pro design-time experience: conditional property visibility, the structure-preview panel rendering, and the custom caption shown on the widget in the canvas.

**2. What kind of logic is described in this file?**  
`getProperties` hides irrelevant properties based on the current configuration: repeat interval properties are hidden when `componentLoadRepeat` is false; for each timing parameter, only the active input type (value or expression) is shown. `getPreview` renders a bordered row in the canvas with an SVG icon and a textual caption, adjusting for dark mode. The icon switches to an "active" variant when at least one event is configured.

**3. What part of behavior can be documented from this file?**  
The canvas caption displays "[Configure events]" when neither `onComponentLoad` nor `onEventChange` is configured, and "[N] Event(s)" (e.g., "[2] Events") when 1 or 2 events are configured. This count-based feedback helps developers confirm both event slots at a glance. The structure-preview uses `@mendix/widget-plugin-platform/preview/structure-preview-api` for composable layout primitives.

**4. Is it user-facing?**  
Yes, this is the developer-facing Studio Pro UI (not end-user facing). It affects only the design-time canvas and property panel.

**5. What new did you learn from this file?**  
The caption function `getCaption` is exported and reused in `Events.editorPreview.tsx`, establishing a single source of truth for the displayed event count text. The `getEventsCount` function treats the two action properties as boolean flags (0 or 1 each), so the maximum event count is 2.

---

## src/Events.editorPreview.tsx

**1. What is the purpose of this file?**  
This is the React-based editor preview component rendered inside Studio Pro's design canvas. It provides a visual representation of the widget in the page layout editor.

**2. What kind of logic is described in this file?**  
The component renders a container div with class `widget-events-preview`, conditionally adding the `active` CSS class when at least one event is configured (`eventsCount > 0`). It renders an `EventsIcon` (SVG icon component) and the caption from `getCaption`. Both `getEventsCount` and `getCaption` are imported from `Events.editorConfig.ts`.

**3. What part of behavior can be documented from this file?**  
The preview shows two visual states: inactive (no events configured) and active (1–2 events). The `isActive` prop on the icon triggers a different SVG variant. This provides immediate visual feedback in Studio Pro when a developer has or has not yet configured any event handlers.

**4. Is it user-facing?**  
Yes, this is visible to Mendix developers in Studio Pro's canvas, not to end users.

**5. What new did you learn from this file?**  
The active/inactive icon state is toggled purely through a prop, not through CSS visibility — indicating the SVG icons themselves represent the two states as separate assets. The preview shares its data logic entirely with `editorConfig.ts`, avoiding duplication.

---

## src/helpers/TimerExecutor.ts

**1. What is the purpose of this file?**  
`TimerExecutor` is a state-machine class that manages timed action execution with precise lifecycle tracking. It handles the initial delay, repeat intervals, and action completion detection — correctly preventing burst execution when actions are slow or when the component is paused (e.g., in an inactive browser tab).

**2. What kind of logic is described in this file?**  
The state machine progresses through states: `initial → scheduled → pending → invoking → executing → (completed | idle)`. `setParams` initiates the first timer; `setCallback` is called on each React render with the latest `canExecute` value. When the timer fires, the executor moves to `pending` and waits for `canExecute` to be true before calling the callback. Transitioning from `invoking → executing` is detected when `canExecute` flips from true to false (action started). Transition from `executing → completed/idle` is detected when `canExecute` flips back to true (action finished). For repeating timers, `idle → scheduled` queues the next interval after execution completes. `stop()` resets to `initial` and cancels any pending `setTimeout`.

**3. What part of behavior can be documented from this file?**  
Key behavioral constraints: (1) The first execution uses `delay`; subsequent executions use `interval`. (2) The next interval timer is never started until the previous action has fully completed (`canExecute` returning to true). (3) A `pending` execution will wait indefinitely for `canExecute`; it does not time out. (4) `stop()` does not fire the callback — it hard-resets the executor.

**4. Is it user-facing?**  
No. This is an internal helper class, not directly visible to Mendix developers or end users.

**5. What new did you learn from this file?**  
The action completion detection is entirely passive — it relies on observing `canExecute` changing from false back to true through React re-renders, rather than using promises or callbacks. This design makes the class React-render-cycle-compatible without any async primitives.

---

## src/helpers/__tests__/TimerExecutor.spec.ts

**1. What is the purpose of this file?**  
This is the comprehensive unit test suite for `TimerExecutor`. It uses Jest fake timers to simulate time passage and verifies all documented state transitions, edge cases, and React lifecycle patterns.

**2. What kind of logic is described in this file?**  
Tests are organized into: basic functionality (`isReady` conditions), single execution (state progression, `canExecute` gating), repeated execution (delay vs. interval, no-burst guarantee), state-transition edge cases (rapid `canExecute` changes), stop behavior (cancellation, restart), React lifecycle simulation (mount/unmount/remount, prop updates, cleanup during execution), and error scenarios (missing callback, undefined params, zero delay).

**3. What part of behavior can be documented from this file?**  
Confirmed behavioral facts: (1) `isReady` is false when `delay` is undefined, or when `repeat` is true but `interval` is undefined. (2) Zero delay executes in the same tick (`setTimeout(fn, 0)`). (3) If an action never completes (canExecute stays false), the timer never fires again — it blocks indefinitely. (4) When `stop()` is called during execution, the executor correctly ignores the subsequent action completion. (5) Callback updates during a pending-but-not-yet-fired timer use the latest callback, not the one registered at timer creation.

**4. Is it user-facing?**  
No. Test file only.

**5. What new did you learn from this file?**  
The tests explicitly cover the "inactive tab" scenario implicitly: by simulating extremely long time advances while the action is "executing" (canExecute = false), confirming no spurious re-execution. This matches the v1.2.0 changelog fix for burst execution in inactive tabs.

---

## src/hooks/parameterValue.ts

**1. What is the purpose of this file?**  
This custom React hook resolves a timing parameter to a `number | undefined` based on whether the configuration uses a static value or a dynamic expression. It provides a single resolution point for the dual-mode input pattern used across all three timing parameters.

**2. What kind of logic is described in this file?**  
`useParameterValue` takes `parameterType` ("number" or "expression"), a static `parameterValue`, and a `parameterExpression` (Mendix `DynamicValue`-like object). When type is "number", it returns the static value directly. When type is "expression", it returns `expression.value.toNumber()` only if `status === "available"` and `value !== undefined`; otherwise it returns `undefined`. The result is memoized via `useMemo`.

**3. What part of behavior can be documented from this file?**  
If the expression is in a loading or error state (status is not "available"), the hook returns `undefined`. This propagates to `TimerExecutor.isReady`, which requires a defined delay — so the timer will not start until the expression resolves. This makes expression-driven timing safe to use with late-loading data. The `toNumber()` call converts `Big` precision decimals to JavaScript floating-point numbers.

**4. Is it user-facing?**  
No. Internal hook, not directly configurable.

**5. What new did you learn from this file?**  
Returning `undefined` when an expression is unavailable creates an implicit "hold" that prevents premature timer start. This is a clean design — the hook does not need to know anything about the timer; it simply withholds a value when data isn't ready.

---

## src/hooks/useAttributeMonitor.ts

**1. What is the purpose of this file?**  
This custom React hook monitors a Mendix `EditableValue` attribute for changes and triggers a debounced action when a change is detected. It encapsulates the first-load skip behavior, debounce management, and cleanup.

**2. What kind of logic is described in this file?**  
The hook creates a stable `AttributeMonitor` instance (via `useState` initializer). The monitor tracks `currentValue`: on first call with a non-undefined, available attribute it stores the value without triggering; on subsequent calls it compares `.value` and triggers the debounced callback if changed. Attributes with status "unavailable" or "loading" are ignored. `canExecute` gates whether the debounced callback actually fires. A cleanup effect calls `attributeMonitor.stop()` on unmount, cancelling any pending debounced execution.

**3. What part of behavior can be documented from this file?**  
Key behavioral constraints: (1) The action does NOT fire on the initial render — it only fires on subsequent value changes. (2) If `canExecute` is false when the debounced callback fires, the action is silently dropped. (3) The debounce delay is re-created (previous one cancelled) whenever `execute` or `delay` changes. (4) Attributes with "unavailable" status are completely ignored — no state update, no trigger.

**4. Is it user-facing?**  
No. Internal hook.

**5. What new did you learn from this file?**  
The monitor uses `debounce` from `@mendix/widget-plugin-platform/utils/debounce`, which returns a `[debouncedFn, cancelFn]` tuple. This tuple pattern allows clean cancellation without needing `useRef` for the cancel handle. The comparison uses `currentValue.value !== newValue.value` (strict inequality), meaning type coercion is not applied.

---

## src/hooks/useOnLoadTimer.ts

**1. What is the purpose of this file?**  
This custom React hook wires the `TimerExecutor` helper to React's lifecycle. It creates a stable `TimerExecutor` instance and synchronizes its callback and timing parameters with current props on each render.

**2. What kind of logic is described in this file?**  
Two `useEffect` hooks manage the executor: one updates the callback (with `execute`, `attribute`, and `canExecute` as dependencies), the other sets params (with `delay`, `interval`, `repeat`) and returns a cleanup that calls `timerExecutor.stop()`. The `useState` initializer pattern ensures the same executor instance is reused for the component's lifetime.

**3. What part of behavior can be documented from this file?**  
The cleanup effect calls `timerExecutor.stop()` when delay/interval/repeat change — not just on unmount. This means changing these props at runtime effectively resets and restarts the timer from scratch (the stop resets the executor to `initial`, then `setParams` starts a new cycle). The callback is bound as `() => execute?.call(attribute)`, passing the attribute as the `this` context — though in practice this is used for the action invocation pattern.

**4. Is it user-facing?**  
No. Internal hook.

**5. What new did you learn from this file?**  
Calling `timerExecutor.stop()` on every params change (not just unmount) means a developer who dynamically changes the delay or interval in Studio Pro will get a fresh timer cycle, not a continuation of the previous one. This is an intentional reset-on-reconfigure behavior.

---

## src/__tests__/AppEvents.spec.tsx

**1. What is the purpose of this file?**  
This is a shallow integration test for the `Events` React component. It verifies that the component renders without errors and produces the expected empty DOM structure.

**2. What kind of logic is described in this file?**  
A single test renders the `Events` component with a full default props object (zero delays, no attribute, no change action). It queries for the `.widget-events` div and asserts it is an empty DOM element.

**3. What part of behavior can be documented from this file?**  
Confirmed: the widget renders a single `.widget-events` div with no children when all event actions are in their default (configured but no-op) state. The test uses `actionValue()` and `dynamicValue()` from `@mendix/widget-plugin-test-utils`, confirming these test utilities are the standard approach for this widget suite.

**4. Is it user-facing?**  
No. Test file only.

**5. What new did you learn from this file?**  
Even when `onComponentLoad` is an `actionValue()` (a default action value with canExecute: false), the component renders cleanly without triggering any actions — confirming the `canExecute && !isExecuting` guard works at the integration level.

---

## src/ui/Events.scss

**1. What is the purpose of this file?**  
This is the runtime stylesheet for the Events widget. It controls the rendered div's layout behavior.

**2. What kind of logic is described in this file?**  
The `.widget-events` class sets `flex-grow: 1` and `position: relative`, with a smooth `color` transition. Two SCSS variables are declared (`$brand-primary`, `$gray-dark`) but are not used in the current ruleset — likely present for future use or consistency with other widgets.

**3. What part of behavior can be documented from this file?**  
The widget's div will grow to fill available flex space in the parent container. Since the widget is invisible, this matters for layout: placing the Events widget in a row layout may cause it to claim space. Developers should be aware that the widget occupies layout space even though it renders no visible content.

**4. Is it user-facing?**  
Indirectly — the CSS affects page layout behavior, even though no visible element is rendered.

**5. What new did you learn from this file?**  
The `flex-grow: 1` is a potential layout footprint concern: in a flex container, the invisible widget will expand to fill space. Developers placing this widget in a layout should account for its default flex behavior.

---

## CHANGELOG.md

**1. What is the purpose of this file?**  
This file documents the version history of the events-web widget, following Keep a Changelog format with semantic versioning.

**2. What kind of logic is described in this file?**  
The changelog records five releases from v1.0.0 (2024-03-19) to v1.3.0 (2026-02-25). Entries cover bug fixes (burst execution, MF/NF parameter handling) and feature additions (expression support for delay/interval).

**3. What part of behavior can be documented from this file?**  
- **v1.0.0**: Initial release with component-load event and attribute-change listener.
- **v1.0.1**: Fixed a regression where actions with Microflow/Nanoflow parameters were not fired.
- **v1.1.0**: Added expression support for all three timing parameters (load delay, repeat interval, change delay).
- **v1.2.0**: Fixed burst execution in inactive browser tabs; repeating actions now wait for the previous execution to complete before starting the next interval.
- **v1.3.0**: Fixed a residual edge case where burst execution could still occur in some scenarios.

**4. Is it user-facing?**  
Yes — the changelog is public and visible to developers using the widget from the Mendix Marketplace.

**5. What new did you learn from this file?**  
The burst-execution bug was fixed in two incremental releases (v1.2.0 and v1.3.0), suggesting the initial fix in v1.2.0 was incomplete for certain timing scenarios. The v1.0.1 fix for MF/NF parameters implies the initial release had a fundamental issue with parameterized action invocation, resolved before expression support was added.
