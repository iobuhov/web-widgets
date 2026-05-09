# Draft: fieldset-web

Widget: `@mendix/fieldset-web` v3.2.2  
Source path: `packages/pluggableWidgets/fieldset-web/`

---

## src/Fieldset.tsx

**1. What is the purpose of this file?**  
This is the top-level entry point for the Mendix pluggable widget. It receives the Mendix platform-injected `FieldsetContainerProps` and delegates rendering to the inner `Fieldset` UI component.

**2. What kind of logic is described in this file?**  
Prop destructuring and prop forwarding. It extracts `legend`, `content`, `name`, `tabIndex`, `style`, and `class` from the container props. The `legend` prop is a `DynamicValue<string>`, so `.value` is read before passing down. The `content` slot (a `ReactNode`) is passed as children.

**3. What part of behavior can be documented from this file?**  
The `legend` prop is optional (`DynamicValue<string> | undefined`) — if absent, `legend?.value` evaluates to `undefined`, and the component renders without a `<legend>` element. This is the behavioral gate for conditional legend rendering. The content is rendered as children inside the fieldset.

**4. Is it user-facing?**  
Not directly. This file is the Mendix widget wrapper; end-users interact with the rendered HTML fieldset. Studio Pro users configure the widget via its properties.

**5. What new did you learn from this file?**  
The legend is driven by a `DynamicValue<string>`, meaning it can be bound to a Mendix attribute or expression — it is not limited to static text. The content slot accepts arbitrary Mendix widgets as children via the `ReactNode` type.

---

## src/components/Fieldset.tsx

**1. What is the purpose of this file?**  
This is the pure UI component that renders the native HTML `<fieldset>` element. It is decoupled from the Mendix platform layer and focuses entirely on rendering behavior.

**2. What kind of logic is described in this file?**  
Conditional rendering of the `<legend>` element: `{legend && <legend>{legend}</legend>}`. It accepts `name`, `className`, `style`, `tabIndex`, and `children` as props, forwarding them directly to the `<fieldset>` DOM element.

**3. What part of behavior can be documented from this file?**  
The `<legend>` element is only rendered when the `legend` prop is a non-empty truthy string. If `legend` is `undefined` or an empty string, no `<legend>` element is emitted in the DOM. Children are always rendered inside the `<fieldset>`, regardless of whether a legend is present.

**4. Is it user-facing?**  
Yes — the rendered `<fieldset>` and optional `<legend>` are visible HTML elements. This is what end-users see in the browser.

**5. What new did you learn from this file?**  
The component does not apply any default class names — the `className` prop must come from the Mendix platform (via the `class` system property). The `tabIndex` prop allows the fieldset itself to be keyboard-focusable, which has accessibility implications.

---

## src/Fieldset.xml

**1. What is the purpose of this file?**  
The widget definition XML. It declares the widget's identity, Mendix Marketplace metadata, Studio Pro category placement, and the configurable properties exposed to developers in Studio Pro.

**2. What kind of logic is described in this file?**  
Property schema declaration: `legend` is a `textTemplate` (optional), `content` is a `widgets` slot, and the standard system properties `Visibility`, `Name`, and `TabIndex` are included. The widget is marked `offlineCapable="true"` and `pluginWidget="true"`.

**3. What part of behavior can be documented from this file?**  
The widget is placed in the **"Input elements"** category in both Studio Pro and Studio. The `legend` property is of type `textTemplate`, supporting dynamic expressions and attribute bindings. The `content` property is a widget container (drop zone). The widget supports offline use.

**4. Is it user-facing?**  
This file is developer-facing (Studio Pro UI). It defines the configuration surface visible to Mendix developers building apps.

**5. What new did you learn from this file?**  
The widget is categorized under "Input elements" (not "Container" or "Layout"), suggesting its intended purpose is to group form input controls, consistent with the HTML `<fieldset>` semantics. Minimum Mendix version is 9.6.0 (from package.json marketplace config).

---

## src/Fieldset.editorConfig.ts

**1. What is the purpose of this file?**  
Defines the Structure mode preview for Studio Pro. It provides `getPreview` (visual structure representation) and `getCustomCaption` (the label shown in the page explorer tree) for the widget.

**2. What kind of logic is described in this file?**  
`getPreview` constructs a nested container layout: an outer bordered container, a header bar (filled with the topbar standard background color) showing the legend text at 14px bold, and a drop zone row for the content slot. `getCustomCaption` returns the legend value or falls back to `"Fieldset"` when no legend is configured.

**3. What part of behavior can be documented from this file?**  
In Structure mode, the widget renders as a bordered box with a colored header displaying the legend. The drop zone is labeled `"Place fieldset content here"`. The page explorer caption is driven by the legend value — if no legend is set, the caption defaults to `"Fieldset"`. Dark mode is supported via the palette API.

**4. Is it user-facing?**  
Developer-facing (Studio Pro design-time experience). End-users do not see Structure mode.

**5. What new did you learn from this file?**  
The legend fallback in `getCustomCaption` means that even without a legend configured, the widget has a meaningful label in the Studio Pro page tree. The structural preview visually mirrors the HTML `<fieldset>` + `<legend>` layout, giving developers an accurate design-time representation.

---

## src/Fieldset.editorPreview.tsx

**1. What is the purpose of this file?**  
Provides the live preview rendering of the widget inside Studio Pro's design canvas. This is separate from Structure mode and renders using the actual React component.

**2. What kind of logic is described in this file?**  
The `preview` function uses `parseStyle` from `@mendix/widget-plugin-platform/preview/parse-style` to convert the string-typed style prop (design-time format) into a `CSSProperties` object. It then renders the `Fieldset` component with a `ContentRenderer` wrapping a placeholder `<div />`.

**3. What part of behavior can be documented from this file?**  
In Studio Pro's design canvas, the widget renders the actual `<fieldset>` + `<legend>` HTML structure using the same UI component as production. The content area displays whatever child widgets are placed in the drop zone, rendered through the `ContentRenderer` wrapper. Style and class are applied in the preview.

**4. Is it user-facing?**  
Developer-facing (Studio Pro design canvas). Reflects what end-users will see, but is rendered in an editor context.

**5. What new did you learn from this file?**  
`tabIndex` and `name` are not passed in the preview — the preview only uses `className`, `style`, and `legend`. This means keyboard focus behavior and DOM naming are not simulated in the design-time preview.

---

## typings/FieldsetProps.d.ts

**1. What is the purpose of this file?**  
Auto-generated TypeScript type definitions derived from `Fieldset.xml`. Provides the `FieldsetContainerProps` (runtime) and `FieldsetPreviewProps` (design-time) interfaces used throughout the widget code.

**2. What kind of logic is described in this file?**  
Type declarations only. `FieldsetContainerProps` includes `name: string`, `class: string`, `style?: CSSProperties`, `tabIndex?: number`, `legend?: DynamicValue<string>`, and `content: ReactNode`. `FieldsetPreviewProps` mirrors these for Studio Pro, with `legend: string` (plain string, not dynamic) and `content` as a renderer object with `widgetCount`.

**3. What part of behavior can be documented from this file?**  
At runtime, `legend` is a `DynamicValue<string>` (can be `undefined` if not configured). At design time, `legend` is a plain string. The `class` prop (not `className`) is the standard Mendix CSS class prop at runtime. The `FieldsetPreviewProps.className` field is marked deprecated since Mendix 9.18.0 in favor of `class`.

**4. Is it user-facing?**  
No — this is an internal TypeScript contract between framework-generated code and widget implementation.

**5. What new did you learn from this file?**  
The `renderMode` field in `FieldsetPreviewProps` can be `"design"`, `"xray"`, or `"structure"`, indicating the widget preview can adapt to multiple Studio Pro rendering modes. The `translate` function for i18n is present in preview props but not used in the current implementation.

---

## src/__tests__/Fieldset.spec.tsx

**1. What is the purpose of this file?**  
Unit tests for the `Fieldset` UI component using React Testing Library and Jest snapshots.

**2. What kind of logic is described in this file?**  
Two test cases: (1) renders `<fieldset>` with `<legend>` and children when all props are provided, (2) renders `<fieldset>` with children but no `<legend>` when `legend` is `undefined`. Both assert via snapshot matching.

**3. What part of behavior can be documented from this file?**  
Confirmed behavioral: when `legend` is `undefined`, no `<legend>` element appears in the DOM output — the fieldset renders children directly inside `<fieldset>`. The `name`, `className`, `style`, and `tabIndex` props are all forwarded to the `<fieldset>` DOM element (confirmed by snapshot output).

**4. Is it user-facing?**  
No — developer testing artifact.

**5. What new did you learn from this file?**  
The `tabIndex` renders as the `tabindex` HTML attribute on the `<fieldset>` (lowercase, as per HTML standard). The unit test uses a standard `<label>` + `<input>` pair as children, reflecting the intended use case of grouping form controls.

---

## src/__tests__/__snapshots__/Fieldset.spec.tsx.snap

**1. What is the purpose of this file?**  
Jest snapshot baseline capturing the exact HTML output of the Fieldset component for regression testing.

**2. What kind of logic is described in this file?**  
Two snapshots: one showing `<fieldset class="className" name="fieldset" tabindex="0"><legend>legend</legend>...children...</fieldset>`, and one without the `<legend>` element.

**3. What part of behavior can be documented from this file?**  
The rendered HTML structure is: `<fieldset>` element with `class`, `name`, `tabindex` attributes → optional `<legend>` as first child → then arbitrary children. The attribute order and casing is confirmed: `class` (not `className`), `tabindex` (lowercase), `name`.

**4. Is it user-facing?**  
No — internal regression baseline.

**5. What new did you learn from this file?**  
The `<legend>` appears as the very first child of `<fieldset>` in the rendered output, before any content children. This is standard HTML behavior for fieldset/legend and is confirmed as the actual render order.

---

## e2e/Fieldset.spec.js

**1. What is the purpose of this file?**  
Playwright end-to-end tests for the Fieldset widget running against a live Mendix test application.

**2. What kind of logic is described in this file?**  
Four E2E tests:
1. **Renders content and legend**: Verifies that `fieldsetLegendYes` renders a legend reading `"Smith's personal info"` and contains two text box widgets as children.
2. **Renders content without legend**: Verifies that `fieldsetLegendNo` renders two inputs without any legend.
3. **Renders when content is hidden by conditional visibility**: Verifies that children count changes from 2 to 0 when a checkbox toggle hides them — but the fieldset and legend remain visible even when content is conditionally hidden.
4. **Updates legend when attribute value changes**: Verifies that the legend text re-renders reactively when the underlying Mendix attribute changes (user edits the last name input, legend updates from `"Smith's personal info"` to `"Smiths's personal info"`).

**3. What part of behavior can be documented from this file?**  
- The legend is reactive: it updates live when the bound Mendix attribute changes (confirmed E2E behavioral constraint).
- Content is conditionally rendered inside the fieldset: child widgets can be shown/hidden via Mendix conditional visibility, and the fieldset container persists even when all content is hidden.
- The fieldset legend remains visible even when all content children are conditionally hidden.
- Standard Mendix widget class names are applied to child widgets inside the fieldset (e.g., `mx-name-LegendYesFirstNameTextBox mx-textbox form-group`).
- Session logout cleanup is forced after each test to avoid exceeding Mendix license session limits (5 sessions).

**4. Is it user-facing?**  
Yes — E2E tests validate the runtime behavior visible to end-users in a real browser.

**5. What new did you learn from this file?**  
The legend attribute binding is live/reactive — it is not a one-time render. When the user edits a field whose value drives the legend expression, the legend text updates without a page reload. This is a key behavioral constraint for dynamic form labeling use cases.

---

## CHANGELOG.md

**1. What is the purpose of this file?**  
Records all notable changes to the widget across versions following Keep a Changelog / Semantic Versioning conventions.

**2. What kind of logic is described in this file?**  
Version history from v2.0.0 (2021-07-28) through v3.2.2 (2026-02-09).

**3. What part of behavior can be documented from this file?**  
Key version findings:
- **v2.0.0 (2021-07-28)**: Added Structure mode preview for Studio Pro. First version with design-time structural representation.
- **v3.0.0 (2021-09-28)**: Added toolbox category and tile image for Studio and Studio Pro.
- **v3.0.1 (2021-12-03)**: Fixed design properties and styles not being applied in Design mode and Studio (behavioral fix — styles and custom CSS classes now correctly applied).
- **v3.1.0 (2021-12-23)**: Added dark mode to Structure mode preview; added dark icons for Tile and List view.
- **v3.1.1 (2022-10-15)**: Test release automation only — no user-facing change.
- **v3.2.0 (2023-06-05)**: Page explorer caption now displays legend value (previously may have shown a generic label). Updated icons and dark/light mode Structure preview colors.
- **v3.2.1 (2023-09-27)**: Removed redundant code to improve widget load time.
- **v3.2.2 (2026-02-09)**: Added license file and readme for open source dependencies.

**4. Is it user-facing?**  
Partially — version history is visible to developers/operators; the behavioral changes (style fixes, legend caption) are end-user-visible.

**5. What new did you learn from this file?**  
The page explorer caption displaying the legend value was added in v3.2.0 — prior to this, the Studio Pro page tree may have shown a generic caption. The design property/style application bug fix in v3.0.1 is a significant behavioral finding: custom styles and CSS classes may not have worked correctly in earlier versions.

---

## package.json

**1. What is the purpose of this file?**  
Defines the npm package metadata, scripts, dependencies, and Mendix Marketplace configuration for the fieldset-web widget.

**2. What kind of logic is described in this file?**  
Package identity (`@mendix/fieldset-web` v3.2.2), build/test/lint/release scripts, dependencies (`@mendix/widget-plugin-component-kit`), and Mendix-specific config including marketplace app number 113922, minimum Mendix version 9.6.0, and MPK name `com.mendix.widget.web.Fieldset.mpk`.

**3. What part of behavior can be documented from this file?**  
- Minimum Mendix version: **9.6.0**
- The widget uses `@mendix/widget-plugin-component-kit` as its only runtime dependency.
- The widget is distributed as an MPK file: `com.mendix.widget.web.Fieldset.mpk`.
- License: Apache-2.0.
- Marketplace app number: 113922 (for reference linking).

**4. Is it user-facing?**  
No — package metadata consumed by build tooling and the Mendix Marketplace.

**5. What new did you learn from this file?**  
The widget has only one runtime dependency (`@mendix/widget-plugin-component-kit`), making it very lightweight. The minimum Mendix version of 9.6.0 sets the platform compatibility floor for all users.
