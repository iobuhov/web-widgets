# Google Tag Command

## Purpose

The Google Tag Command widget integrates Google Analytics (gtag) into a Mendix application without producing any visible DOM output. It dispatches tracking commands to Google's `gtag` function — either a structured page view in Basic mode or a fully user-configurable event or config command in Advanced mode. A singleton pattern ensures the GTM script is injected into the document exactly once regardless of how many widget instances are placed on a page or how many times the component re-renders.

## User Scenarios

### [P1] Basic page view tracking
**Given** the widget is placed on a Mendix page with `widgetMode` set to `"basic"` and a valid `targetId` configured  
**When** the widget mounts and `targetId.status` is `"available"` and all custom parameters are resolved  
**Then** the widget fires a `config` command (with `send_page_view: false`) followed by an explicit `page_view` event, including `page_location`, `client_id`, `language`, `page_title`, `mx_page_name`, `mx_module_name`, and any additional user-defined parameters

#### Edge Cases
- Execution is guarded by a `needsExecution` ref — if the widget re-renders without a navigation event, commands are not re-fired on the same page load.
- Execution is deferred until `targetId.status === "available"` and all parameters with `valueType === "custom"` have `customValue.status === "available"`. If any value is still loading, no commands are sent.
- When `sendUserID` is `true`, the authenticated Mendix user ID is appended to the `page_view` event parameters. If `getUserId()` returns `undefined` at runtime, the user ID is not sent (guarded by an `!== undefined` check).

### [P2] Advanced event tracking
**Given** the widget is in `"advanced"` mode with `command` set to `"event"`  
**When** the resolved parameters are all available  
**Then** the widget fires `gtag("event", eventName, parameters)` with user-defined key/value pairs

#### Edge Cases
- In Advanced event mode, `targetId` and `sendUserID` are hidden in Studio Pro and are not required.
- Event name (`eventName`) is required in Studio Pro validation; saving without it produces an editor error.
- When `trackPageChanges` is `true`, the widget subscribes to Dojo navigation events and re-fires on each navigation. If `window.dojo` is not present (e.g., Mendix React client without Dojo), navigation tracking is silently disabled and a console error is logged.

### [P3] Advanced config command
**Given** the widget is in `"advanced"` mode with `command` set to `"config"`  
**When** the resolved parameters are available  
**Then** the widget fires `gtag("config", targetId, parameters)`, which also ensures the GTM script has been injected

#### Edge Cases
- In Advanced config mode, parameters can only use custom expression values. The `predefined` value type is hidden in Studio Pro, restricting configurators to custom expressions only.
- `targetId` is required in Advanced config mode; Studio Pro validation enforces this.
- `trackPageChanges` and `eventName` are hidden in Advanced config mode.

### [P4] GTM script singleton injection
**Given** one or more Google Tag widget instances are placed on the same page  
**When** any instance fires a `config` command  
**Then** the GTM script (`https://www.googletagmanager.com/gtag/js?id={targetId}`) is injected into `<head>` exactly once; subsequent injections by other instances or re-renders are no-ops

#### Edge Cases
- The singleton is stored on `document.mxGtag` (not a module-level variable) to survive across separate widget module scopes and React re-mounts.
- `ensureGtagIncluded` returns `false` on all calls after the first, confirming no-op behavior.

## Functional Requirements

- FR-001: The widget MUST render `null` — it produces no DOM output and is a tracking-only widget.
- FR-002: In Basic mode, the widget MUST fire two gtag commands in sequence: `config` with `send_page_view: false`, then `event` `page_view` with the standard Mendix page metadata fields.
- FR-003: The widget MUST NOT re-fire commands within the same page load cycle. A `needsExecution` ref guard MUST be checked before dispatching commands and reset on each Dojo navigation event.
- FR-004: Execution MUST be deferred until `targetId.status === "available"` (when required) and all `custom`-type parameters have `customValue.status === "available"`.
- FR-005: The GTM script MUST be injected asynchronously exactly once per document, using `document.mxGtag` as the cross-instance singleton guard.
- FR-006: Custom parameter values MUST support boolean coercion: the strings `"true"` and `"false"` MUST be converted to boolean `true` and `false` respectively.
- FR-007: Custom parameter values containing the token `{{__ModuleAndPageName__}}` MUST have the token replaced with the current Mendix page's module/page path.
- FR-008: Predefined parameter values MUST be resolved at execution time from the live Mendix form context (`window.mx.ui.getContentForm()` and `window.mx.session.*`), not at configuration time.
- FR-009: The `config` command MUST call `ensureGtagIncluded(tagId)` before dispatching; the `event` command MUST NOT call it.
- FR-010: The widget MUST be `offlineCapable` and require entity context (`needsEntityContext="true"`), restricting placement to within a Mendix data view or equivalent context.
- FR-011: In Advanced event mode with `trackPageChanges=true`, the widget MUST subscribe to Dojo navigation events via `window.dojo.connect` and unsubscribe on component cleanup.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `widgetMode` | Enum (`basic` \| `advanced`) | — | Widget mode | Selects between Basic (structured page view) and Advanced (custom command) operation. |
| `targetId` | Expression (String) | — | Target ID | Google Tag ID (formats: GT-XXXXXXXXX, G-XXXXXXXXX, AW-XXXXXXXXX). Required in Basic mode and Advanced config command. Optional otherwise. |
| `parameters` | List | — | Parameters | Key/value pairs sent with the gtag command. Each entry has `name`, `valueType`, `predefinedValue`, and `customValue`. |
| `parameters.name` | String | — | Name | Parameter key name sent to gtag. |
| `parameters.valueType` | Enum (`predefined` \| `custom`) | — | Value type | Selects a built-in Mendix value or a custom expression. |
| `parameters.predefinedValue` | Enum | — | Predefined value | One of: `pageTitle`, `pageUrl`, `pageName`, `moduleName`, `pageAndModuleName`, `sessionId`, `userLocale`. Available in Basic mode and Advanced event command only. |
| `parameters.customValue` | Expression (String) | — | Custom value | Arbitrary string expression. Supports boolean coercion and `{{__ModuleAndPageName__}}` token. |
| `sendUserID` | Boolean | `false` | Send user ID | When `true`, appends the authenticated Mendix user ID to the gtag parameters. Only available in Basic mode. |
| `command` | Enum (`event` \| `config`) | `"event"` | Command | gtag command type for Advanced mode. `event` sends a named event; `config` sends a configuration command and ensures GTM script injection. |
| `eventName` | String | — | Event name | Name of the gtag event (Advanced event command only). Required when `command` is `"event"`. |
| `trackPageChanges` | Boolean | `false` | Track page changes | When `true`, re-fires the command on each Dojo navigation event (Advanced event command only). Requires `window.dojo`. |

## Changelog

- **Unreleased** — Breaking change: the widget is being re-published as a new Mendix Marketplace item. Existing installations referencing the old item must migrate.
- **v1.4.0 (2025-04-16):** Updated minimum Studio Pro version to 10.21 for Mendix 11 compatibility.
- **v1.3.0 (2024-10-18):** Added optional user ID sharing (`sendUserID` prop). Removed redundant code to reduce browser load time.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] What is the exact Mendix entity context requirement? The widget declares `needsEntityContext="true"` but the gtag commands do not appear to read entity attributes directly — clarify whether the entity context is used for data binding or is a placement constraint only.
- [ ] When `trackPageChanges=true` and `window.dojo` is absent (Mendix React client), is this a supported configuration or a known unsupported combination? The console error is logged silently; clarify whether a Studio Pro validation warning should be added.
- [ ] The `getPathAndModule` helper removes the last 9 characters of the form path to strip `.page.xml`. Does this assumption hold for all Mendix page path formats, including nested module paths and pages with very short names?
