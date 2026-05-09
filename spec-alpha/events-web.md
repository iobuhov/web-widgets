# Events

## Purpose

The Events widget is an invisible action-dispatch mechanism for Mendix web applications. It solves the need to trigger microflows or nanoflows in response to two programmatic events — component mount and attribute value changes — without requiring any user interaction or visible UI element. Use it when a page must execute business logic on load, on a timer, or reactively whenever a data attribute changes.

## User Scenarios

### [P1] Execute an action on page load with a delay

**Given** a Mendix page with the Events widget placed in a layout  
**When** the page mounts and the configured component-load delay elapses  
**Then** the system executes the configured `onComponentLoad` action exactly once

#### Edge Cases

- If the delay is driven by an expression that is not yet available at mount time, the timer does not start until the expression resolves to a value.
- If the `onComponentLoad` action is executing when the timer fires, the execution is queued until `canExecute` returns to `true`; it does not fire a second time.
- A delay of `0` causes the action to execute in the same JavaScript tick as mount.

---

### [P2] Repeat an action on a fixed interval

**Given** the Events widget with `componentLoadRepeat` enabled and a repeat interval configured  
**When** the initial delay elapses and the first execution completes  
**Then** the system schedules the next execution after the configured interval, waiting for the previous execution to finish before starting the next

#### Edge Cases

- The next interval timer does not begin until the previous action's `canExecute` returns to `true` (action finished); burst execution is prevented even in inactive browser tabs.
- If the repeat interval expression is unavailable, the executor is not ready and no timer fires.
- Changing the delay or interval at runtime resets the timer cycle from scratch; the current interval is cancelled and a new one begins.

---

### [P3] Execute an action when an attribute value changes

**Given** a Mendix attribute bound to `onEventChangeAttribute` and a change action configured  
**When** the attribute's value changes after the initial render  
**Then** the system debounces the change for the configured delay and executes the `onEventChange` action

#### Edge Cases

- The action does NOT fire on the initial render — only on subsequent value changes.
- Attributes with `status: "unavailable"` or `status: "loading"` are ignored; no trigger occurs.
- If `canExecute` is `false` when the debounced callback fires, the execution is silently dropped.
- Value comparison uses strict equality (`!==`); type coercion is not applied.

---

## Functional Requirements

- FR-001: The system MUST fire `onComponentLoad` no earlier than the configured load delay after component mount.
- FR-002: The system MUST NOT start the next repeat interval timer until the previous `onComponentLoad` action execution has fully completed.
- FR-003: The system MUST NOT fire `onComponentLoad` while a previous execution is in progress (`canExecute === false`).
- FR-004: The system MUST skip the initial attribute value and fire `onEventChange` only on subsequent changes.
- FR-005: The system MUST debounce `onEventChange` by the configured change delay; rapid successive changes MUST result in a single deferred execution.
- FR-006: When a timing parameter is expression-driven, the system MUST withhold timer start until the expression status is `"available"`.
- FR-007: The system MUST reset and restart the component-load timer when `delay`, `interval`, or `repeat` props change at runtime.
- FR-008: The system MUST cancel any pending debounced attribute-change execution on component unmount.
- FR-009: The widget MUST render no visible UI elements; only an empty `<div class="widget-events">` is emitted to the DOM.
- FR-010: The widget `<div>` MUST apply `flex-grow: 1` CSS, and developers MUST account for this layout footprint when placing the widget in flex containers.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `onComponentLoad` | `ActionValue` | — | On load action | Action executed when the component mounts (after the configured delay). |
| `componentLoadDelayType` | `"number" \| "expression"` | `"number"` | Load delay type | Selects whether the load delay is a static integer or a dynamic expression. |
| `componentLoadDelay` | `integer` | `0` | Load delay (ms) | Static millisecond delay before the load action fires. |
| `componentLoadDelayExpression` | `DynamicValue<Big>` | — | Load delay expression | Expression returning the load delay in milliseconds; used when type is `"expression"`. Always present; timer withheld until status is `"available"`. |
| `componentLoadRepeat` | `boolean` | `false` | Repeat | When `true`, the load action repeats on the configured interval after each completion. |
| `componentLoadRepeatIntervalType` | `"number" \| "expression"` | `"number"` | Interval type | Selects whether the repeat interval is a static integer or a dynamic expression. |
| `componentLoadRepeatInterval` | `integer` | `30000` | Repeat interval (ms) | Static millisecond interval between repeated load action executions. |
| `componentLoadRepeatIntervalExpression` | `DynamicValue<Big>` (optional) | — | Repeat interval expression | Expression returning the repeat interval in milliseconds; optional. |
| `onEventChange` | `ActionValue` | — | On change action | Action executed when `onEventChangeAttribute` changes value. |
| `onEventChangeAttribute` | `EditableValue<Big \| any \| boolean \| Date \| string>` | — | Watch attribute | The Mendix attribute to monitor for value changes. Accepts all primitive types (AutoNumber, Binary, Boolean, DateTime, Enum, HashString, Integer, Long, String, Decimal). |
| `onEventChangeDelayType` | `"number" \| "expression"` | `"number"` | Change delay type | Selects whether the change debounce delay is a static integer or a dynamic expression. |
| `onEventChangeDelay` | `integer` | `0` | Change delay (ms) | Static debounce delay in milliseconds before the change action fires. |
| `onEventChangeDelayExpression` | `DynamicValue<Big>` | — | Change delay expression | Expression returning the change debounce delay in milliseconds; used when type is `"expression"`. |

## Changelog

**v1.3.0** (2026-02-25): Fixed a residual edge case where burst execution could still occur in some timing scenarios.

**v1.2.0**: Fixed burst execution in inactive browser tabs; repeating actions now wait for the previous execution to complete before scheduling the next interval timer.

**v1.1.0**: Added expression support for all three timing parameters (load delay, repeat interval, change delay), enabling runtime-dynamic timing.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] What is the intended behavior when `onComponentLoad` is configured but `onEventChange` is not, and vice versa — are there validation warnings in Studio Pro for partially configured states?
- [ ] Does `flex-grow: 1` on the widget div require a documented workaround (e.g., wrapping in a fixed-size container) to prevent unintended layout expansion in common page templates?
- [ ] Are there Mendix platform-level limits on how short the repeat interval can be before it causes performance issues?
