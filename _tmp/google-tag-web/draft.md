# Draft: google-tag-web

Widget package name: `google-tag-web`  
Widget XML ID: `com.mendix.widget.web.txhhdgfn.TXhHdGFn`  
Widget display name: "Google Tag Command"

---

## src/TXhHdGFn.tsx

**1. What is the purpose of this file?**
This is the main React entry point for the widget. It exports the `GoogleTag` component, which dispatches between two sub-components based on the `widgetMode` prop: `GoogleTagBasicPageView` (for `"basic"`) and `GoogleTagAdvancedMode` (for `"advanced"`).

**2. What kind of logic is described in this file?**
Two modes of operation are implemented. In Basic mode, the component waits for `targetId` and all parameters to be available (Mendix expression evaluation), then fires a `config` command followed by a `page_view` event with a fixed set of predefined page metadata fields. It also optionally sends the authenticated user ID if `sendUserID` is true. In Advanced mode, the component fires a user-configured command (either `"event"` or `"config"`) with user-defined parameters; if `trackPageChanges` is true, it re-fires on every Dojo navigation event.

**3. What part of behavior can be documented from this file?**
- The widget renders `null` — it produces no DOM output; it is a tracking-only widget.
- A `needsExecution` ref guard prevents re-firing within the same page load cycle; the guard is reset on navigation events.
- Basic mode fires two gtag commands in sequence: `config` (with `send_page_view: false`) and then `event` `page_view`. It does not send a raw `page_view` through the `config` call.
- Basic mode sends `page_location`, `client_id`, `language`, `page_title`, `mx_page_name`, `mx_module_name` automatically, plus any user-defined extra `parameters`.
- Advanced mode's `trackPageChanges` option uses `useDojoOnNavigation` to reset the execution guard and re-run on navigation; this option only applies to Advanced mode, not Basic.
- Execution is deferred until `targetId.status === "available"` and all parameters with `valueType === "custom"` have their `customValue.status === "available"`.

**4. Is it user-facing?**
Not directly visible to end users. The widget produces no UI. Developers/configurators interact with it via Mendix Studio Pro properties.

**5. What new did you learn from this file?**
Basic mode always sends `send_page_view: false` in the config call — meaning it suppresses the default Google page_view and instead fires its own explicit `page_view` event. This distinction is important for analytics setup: the widget takes ownership of the page_view event rather than relying on the gtag library's built-in behavior.

---

## src/TXhHdGFn.xml

**1. What is the purpose of this file?**
The Mendix widget definition XML. Declares the widget's identity (`pluginWidget="true"`, `needsEntityContext="true"`, `offlineCapable="true"`, `supportedPlatform="Web"`), display name, and all configurable properties used in Studio Pro.

**2. What kind of logic is described in this file?**
Defines the widget's property schema: `widgetMode` (Basic/Advanced), `targetId` (expression returning String, optional), `parameters` (list of objects each with `name`, `valueType`, `predefinedValue`, and `customValue`), `sendUserID` (boolean), `command` (event/config), `eventName` (string), and `trackPageChanges` (boolean).

**3. What part of behavior can be documented from this file?**
- `targetId` is optional (`required="false"`), accepting a String expression — supports tag IDs of formats GT-XXXXXXXXX, G-XXXXXXXXX, AW-XXXXXXXXX.
- `parameters` is a list prop (`isList="true"`, `required="false"`) supporting multiple key/value pairs.
- Each parameter's value can be `"predefined"` (choosing from 7 built-in options: pageTitle, pageUrl, pageName, moduleName, pageAndModuleName, sessionId, userLocale) or `"custom"` (any String expression).
- `sendUserID` defaults to `false`; when `true`, exposes the authenticated user ID to Google Analytics.
- `command` defaults to `"event"` with options `"event"` and `"config"`.
- `trackPageChanges` defaults to `false`.
- The widget supports offline-capable flag, indicating it's designed for web but declares `offlineCapable="true"`.

**4. Is it user-facing?**
This file defines the configuration surface in Studio Pro — not visible to end users but directly shapes the developer/configurator experience.

**5. What new did you learn from this file?**
The widget requires entity context (`needsEntityContext="true"`), meaning it must be placed inside a data view or similar context in Mendix. This is a placement constraint that affects where the widget can be used in layouts.

---

## src/commonGtag.ts

**1. What is the purpose of this file?**
Implements a singleton pattern for Google Tag (gtag) initialization. It ensures the `window.dataLayer` array and the `gtag` function are created only once per document load, and that the Google Tag Manager script is injected into the `<head>` at most once.

**2. What kind of logic is described in this file?**
Uses `document.mxGtag` as the singleton guard (`if (document.mxGtag === undefined)`). Defines a `gtagMethod` that pushes arguments to `window.dataLayer`. Exposes two methods on `document.mxGtag`: `getGtag()` returns the `gtagMethod` function; `ensureGtagIncluded(tagId)` injects the GTM script tag asynchronously the first time it is called and sets the initialization timestamp. Subsequent calls to `ensureGtagIncluded` are no-ops and return `false`.

**3. What part of behavior can be documented from this file?**
- The GTM script (`https://www.googletagmanager.com/gtag/js?id={tagId}`) is loaded asynchronously from Google's CDN.
- The script is injected exactly once; multiple widget instances on the same page (or re-renders) do not re-inject it.
- `document.mxGtag` is the cross-instance singleton — it persists across React renders and widget re-mounts.
- The `gtagMethod` function uses `arguments` object (classic function, not arrow) to push to `dataLayer`, which matches the standard gtag implementation pattern.

**4. Is it user-facing?**
Not user-facing. This is an internal infrastructure file. No configuration is exposed through it.

**5. What new did you learn from this file?**
The singleton is stored on `document.mxGtag` rather than in a module-level variable. This allows the singleton to survive across separate widget instances (since different widget instances may run in different module scopes), ensuring only one `<script>` tag and one `dataLayer` exist per page regardless of how many Google Tag widget instances are placed.

---

## src/utils.ts

**1. What is the purpose of this file?**
Provides utility functions used by the main widget component: parameter readiness checking, parameter preparation, predefined value resolution, gtag command execution, and Dojo navigation event subscription.

**2. What kind of logic is described in this file?**
- `areParametersReady`: returns `true` if all parameters either have `valueType === "predefined"` or have `customValue.status === "available"`.
- `prepareParameters`: maps the parameters list to a key/value record, delegating to `prepareValue`.
- `prepareValue`: resolves predefined parameters via `getPredefinedValue`, and for custom values handles boolean coercion (`"true"`/`"false"` strings → boolean) and the `{{__ModuleAndPageName__}}` token replacement.
- `getPredefinedValue`: reads live Mendix runtime values from `window.mx.ui.getContentForm()` and `window.mx.session.*`.
- `executeCommand`: dispatches to `commonGtag` — `"config"` calls `ensureGtagIncluded` then `gtag(command, tagId, params)`; `"event"` calls `gtag(command, eventName, params)`.
- `useDojoOnNavigation`: subscribes to the `onNavigation` event of the current form via `window.dojo.connect`; disconnects on cleanup. If `window.dojo` is not found, logs an error and disables tracking silently.

**3. What part of behavior can be documented from this file?**
- Custom parameter values support boolean coercion: the string `"true"` becomes `true` and `"false"` becomes `false` — all other strings pass through as strings.
- The `{{__ModuleAndPageName__}}` token in custom values is replaced with the current page's module/page path.
- Predefined values are resolved at execution time from the live Mendix form, not at configuration time.
- The `"config"` command requires `ensureGtagIncluded` (i.e., it triggers or verifies the script injection), while `"event"` does not — meaning at least one `"config"` command must have run before events can be received.
- `useDojoOnNavigation` requires `window.dojo` to be present; if absent (e.g., modern React client), navigation tracking is silently disabled with a console error.
- `pageName` is extracted as the second path segment; `moduleName` is the first path segment; `pageAndModuleName` is the full path minus the `.page.xml` suffix.

**4. Is it user-facing?**
Not directly. Internal logic that shapes what data gets sent to Google Analytics based on user configuration.

**5. What new did you learn from this file?**
The `getPathAndModule` helper strips `.page.xml` by removing the last 9 characters of `formPath` — a fixed-length suffix assumption. This means `pageName` and `moduleName` are derived from the Mendix form's file path structure, not a dynamic API, which is a tight coupling to Mendix's path conventions.

---

## src/TXhHdGFn.editorConfig.ts

**1. What is the purpose of this file?**
Defines Studio Pro design-time behavior: property visibility rules (`getProperties`) and validation errors (`check`). Imported from `@mendix/pluggable-widgets-tools` (external npm package — not a local workspace package).

**2. What kind of logic is described in this file?**
`getProperties`: in Basic mode, hides `command`, `eventName`, and `trackPageChanges`. In Advanced mode with `command === "config"`, hides `valueType` and `predefinedValue` nested properties (forcing custom-only parameter values) and hides `eventName`/`trackPageChanges`. In Advanced mode with `command === "event"`, hides `targetId` and `sendUserID`, and applies `handleValueTypes`. `handleValueTypes` toggles `customValue` or `predefinedValue` visibility based on each parameter's `valueType`. `check`: validates that `targetId` is non-empty for Basic mode and for Advanced config command; validates that `eventName` is non-empty for Advanced event command.

**3. What part of behavior can be documented from this file?**
- In Advanced config command, parameters can only use custom values (predefined type is hidden from the editor).
- In Advanced event command, `targetId` and `sendUserID` are hidden — the event command does not need a tag ID.
- In Basic mode, `command`, `eventName`, and `trackPageChanges` are hidden — they are irrelevant to basic operation.
- Tag ID (`targetId`) is required (editor validation enforced) in Basic mode and in Advanced config command; it is not required for Advanced event command.
- Event name is required (editor validation enforced) only for Advanced event command.

**4. Is it user-facing?**
Affects the Mendix Studio Pro developer experience. End users never see this.

**5. What new did you learn from this file?**
The Advanced config command hides the `valueType` and `predefinedValue` nested parameter properties entirely, forcing developers to use only custom expression values when configuring a `config` command. This is a strict behavioral constraint: predefined values are only available in Basic mode and Advanced event command, not in Advanced config command.

---

## typings/TXhHdGFnProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript type definitions for the widget's props, derived from `TXhHdGFn.xml`. Provides compile-time types for both the container (runtime) props and preview (Studio Pro design-time) props.

**2. What kind of logic is described in this file?**
Defines enums: `WidgetModeEnum` (`"basic" | "advanced"`), `ValueTypeEnum` (`"predefined" | "custom"`), `PredefinedValueEnum` (7 values), `CommandEnum` (`"event" | "config"`). Defines `ParametersType` with `name: string`, `valueType`, `predefinedValue`, and optional `customValue?: DynamicValue<string>`. `TXhHdGFnContainerProps` includes all runtime props; `TXhHdGFnPreviewProps` is for Studio Pro previews.

**3. What part of behavior can be documented from this file?**
- `targetId` is typed as `DynamicValue<string>` (optional), confirming it's a Mendix expression evaluated at runtime.
- `customValue` in `ParametersType` is optional (`DynamicValue<string> | undefined`), meaning a parameter in custom mode without a value configured results in `undefined`.
- The preview type uses plain `string` for `targetId` and `customValue` rather than `DynamicValue`, which is the standard pattern for design-time vs. runtime in Mendix pluggable widgets.

**4. Is it user-facing?**
No. Internal TypeScript infrastructure. Developer-facing only.

**5. What new did you learn from this file?**
The generated file is marked `WARNING: All changes made to this file will be overwritten`, confirming it is derived from the XML schema and should not be manually edited.

---

## typings/global.d.ts

**1. What is the purpose of this file?**
Declares global TypeScript types for browser globals the widget depends on: `window.dataLayer`, `window.mx` (Mendix client API), `window.dojo` (Dojo framework), and `document.mxGtag` (the widget's own singleton).

**2. What kind of logic is described in this file?**
Augments the `Window` interface with `dataLayer: any[]`, `mx.ui.getContentForm()` returning `{ path: string; url: string; title: string }`, `mx.session` with `getSessionObjectId()`, `getConfig()`, and `getUserId()`, and `dojo` with `connect`/`disconnect` methods. Augments `Document` with optional `mxGtag`.

**3. What part of behavior can be documented from this file?**
- `getContentForm().url` is typed as `string` (not `string | undefined`), though `utils.ts` uses a nullish coalescing fallback (`?? window.location.origin`), suggesting the url may be falsy in practice.
- `getUserId()` returns `string` (not optional), but the widget guards against it with `getUserId() !== undefined` in `TXhHdGFn.tsx`, implying the runtime may return `undefined` despite the typing.
- `DojoConnectHandle` is a branded string type to provide type safety for connect/disconnect pairs.

**4. Is it user-facing?**
No. TypeScript infrastructure.

**5. What new did you learn from this file?**
The Mendix `mx.ui.getContentForm()` API shape is defined here as returning `{ path, url, title }` — note there is no `onNavigation` property typed here, yet `utils.ts` calls `window.mx.ui.getContentForm().onNavigation` via Dojo's connect. This means the `onNavigation` callback is a Dojo event hook on the form object not represented in the TypeScript type, confirming a Dojo-specific integration that bypasses TypeScript type safety.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Records all published versions of the google-tag-web widget with dates and change summaries.

**2. What kind of logic is described in this file?**
Version history from v1.0.0 (2023-02-17) through v1.4.0 (2025-04-16), plus an unreleased breaking change section.

**3. What part of behavior can be documented from this file?**
- **Unreleased** — Breaking change: the widget was re-published as a new Marketplace item. Any existing installations referencing the old item will need to migrate.
- **v1.4.0** (2025-04-16) — Updated Studio Pro minimum version to 10.21 for Mendix 11 support. This is a compatibility constraint: the widget requires Studio Pro 10.21+.
- **v1.3.0** (2024-10-18) — Added optional user ID sharing (`sendUserID` prop) to expose the authenticated user ID to Google Analytics. Also removed redundant code to improve browser load time.
- **v1.2.0** (2023-06-28) — Added ability to configure additional tags via the `"config"` command in Advanced mode. This was a new capability added after initial release.
- **v1.1.0** (2023-06-05) — Updated light and dark icons and tiles; cosmetic/branding change only.
- **v1.0.0** (2023-02-17) — Initial release of the Google Tag widget.

**4. Is it user-facing?**
Not visible to end users. Relevant to developers and operators managing widget versions.

**5. What new did you learn from this file?**
The Advanced mode `"config"` command support (v1.2.0) was added 4 months after the initial release, meaning the original widget only supported basic page_view tracking. The `sendUserID` feature (v1.3.0) was a community contribution (`@rborer`). The upcoming re-publication as a new Marketplace item (Unreleased) is a breaking distribution change that does not affect functionality but affects installation/upgrade paths.
