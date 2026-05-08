# Draft: chart-playground-web

Widget path: `packages/pluggableWidgets/chart-playground-web/`

---

## ChartPlayground.xml

**1. Purpose:**
Widget descriptor with zero configurable properties. Declares the widget as `needsEntityContext="true"` in the "Charts" category for Web platform.

**2. Core logic:**
No properties block — `<properties />` is empty. The widget has no design-time configurability; all runtime data is injected via React context from the parent chart widget.

**3. Documentable behavior:**
This widget cannot be dragged-and-dropped as a standalone widget with configuration. It is always used as a child/companion of a chart widget that provides context via `usePlaygroundContext()`.

**4. User-facing:**
None at the widget property level. Users cannot configure this widget in Studio Pro.

**5. New learnings:**
A Mendix widget descriptor can have an empty `<properties />` element, making the widget fully context-driven with no Studio Pro property panel at all.

---

## ChartPlaygroundProps.d.ts

**1. Purpose:**
Auto-generated TypeScript type definitions from the XML descriptor. Declares the `ChartPlaygroundContainerProps` and `ChartPlaygroundPreviewProps` interfaces.

**2. Core logic:**
Since the XML has no properties, both interfaces contain only the standard Mendix framework fields (className, style, tabIndex, id) with no widget-specific fields.

**3. Documentable behavior:**
The props type is effectively a minimal Mendix widget container — no business logic properties are typed here.

**4. User-facing:**
Not directly user-facing; consumed by the widget implementation.

**5. New learnings:**
Auto-generated props files for zero-property widgets still carry the boilerplate Mendix fields, confirming the framework always provides a base set of container props.

---

## ChartPlayground.tsx

**1. Purpose:**
Top-level widget entry point. Bridges the Mendix widget framework to the internal React component tree.

**2. Core logic:**
```tsx
export function ChartPlayground(): ReactElement | null {
    return <Playground />;
}
```
Completely delegates to the `Playground` component. Does not pass any props because everything is retrieved via context internally.

**3. Documentable behavior:**
The widget renders nothing of its own — it is a thin shell around `Playground`. This pattern keeps Mendix framework concerns (props, className, tabIndex) separated from the internal component tree.

**4. User-facing:**
No visible output beyond what `Playground` renders.

**5. New learnings:**
The extreme minimalism here (3 lines of real code) confirms the widget is designed as a pure context consumer — the parent chart widget is fully responsible for providing all data.

---

## components/Playground.tsx

**1. Purpose:**
Root React component that reads playground context and dispatches to the correct editor generation.

**2. Core logic:**
Calls `usePlaygroundContext()` from `@mendix/shared-charts/main`. On error, renders an `Alert`. On success, checks `context.type === "editor.data.v2"` to render `EditorGen2`; otherwise falls back to `EditorGen1`. Both editors receive the full context as props.

**3. Documentable behavior:**
- If the parent chart provides V1 data (`PlaygroundDataV1`): renders `EditorGen1` (simple state-based controller).
- If the parent chart provides V2 data (`PlaygroundDataV2`, type `"editor.data.v2"`): renders `EditorGen2` (MobX-based controller).
- If context is unavailable or errors: renders an error alert message.

**4. User-facing:**
The error alert is visible if the playground is used outside a chart context. Normal usage shows the appropriate editor.

**5. New learnings:**
The `type` discriminant on the context object is the single branch point between the two editor generations. V1 and V2 coexist in the same widget via this runtime dispatch.

---

## components/ComposedEditor.tsx

**1. Purpose:**
Main playground UI. Provides the sidebar toggle, two code panels (editable + read-only), view selector, and docs link.

**2. Core logic:**
- Renders a "Toggle Editor" button (absolutely positioned, top-right) that shows/hides the sidebar.
- When open, sidebar contains:
  - A `SidebarContentTooltip` with warning: "Changes are only for preview purposes. To persist changes copy value and paste it in Studio Pro."
  - **Panel 1** (editable `CodeEditor`): bound to `controller.editorInput` / `controller.onEditorChange`
  - **Panel 2** (read-only `CodeEditor`): shows current Studio Pro modeler settings
  - A `<Select>` for switching between views (Layout, individual traces, Configuration)
  - A `DocsLink` pointing to plotly.com reference docs
- `TabGuard` traps Tab focus within the editor panels.

**3. Documentable behavior:**
- Toggle button is always visible on the chart (top-right, z-index 50).
- Sidebar panels are only rendered when playground is open.
- The editable panel updates live; changes affect only the current preview session.
- The read-only panel reflects what the developer set in Studio Pro modeler.
- `irrelevantSeriesKeys = ["x", "y", "z", "customSeriesOptions", "dataSourceItems"]` are filtered from the modeler display to reduce noise.

**4. User-facing:**
All visible UI of the playground lives here: toggle button, sidebar panels, view selector, docs link, tooltip warning.

**5. New learnings:**
Filtering `irrelevantSeriesKeys` from the read-only panel is a deliberate UX decision — data-bound series keys that cannot be meaningfully edited are hidden from the developer view to keep the JSON manageable.

---

## components/CodeEditor.tsx

**1. Purpose:**
Thin wrapper around a `<textarea>` for JSON code editing within the playground sidebar.

**2. Core logic:**
Renders a `<textarea>` with:
- Monospace font family
- Configurable height prop (default 200px)
- Passes through `value`, `onChange`, `readOnly`, and `aria-label` props

**3. Documentable behavior:**
No syntax highlighting, no validation, no line numbers — intentionally minimal. The height is the only layout customization exposed.

**4. User-facing:**
Developers type raw JSON directly. There is no autocomplete or error highlighting; any JSON parse errors surface only when the chart attempts to apply the value.

**5. New learnings:**
The simplicity is intentional for a developer tool used in preview mode. A full editor (Monaco, CodeMirror) would add bundle size with little payoff for a temporary preview feature.

---

## helpers/useComposedEditorController.ts

**1. Purpose:**
V1 editor controller hook. Manages editor state for charts using `PlaygroundDataV1`.

**2. Core logic:**
- Reads `store.state.data[key]`, `store.state.layout`, `store.state.config` from the V1 store.
- Calls `fallback(value)` to get the default JSON string, then parses it as the editable value.
- On change: parses user JSON input, calls the appropriate store setter (e.g. `store.setLayout(obj)`, `store.setDataAt(key, value)`).
- Builds view options as `[Layout, trace-0..n named by trace name, Configuration]`.

**3. Documentable behavior:**
- View switching resets the editor content to the value for the selected key.
- Invalid JSON in the editor is silently ignored (parse fails → store not updated).
- The `fallback()` function is provided by the parent chart widget via context — it formats raw store values into display-ready JSON strings.

**4. User-facing:**
Controls what JSON appears in Panel 1 when a view is selected.

**5. New learnings:**
`fallback()` is an escape hatch for the parent chart to inject default or normalized values — the playground never hard-codes what "default" JSON looks like for any particular chart type.

---

## helpers/useV2EditorController.ts

**1. Purpose:**
V2 editor controller using MobX reactivity for charts using `PlaygroundDataV2`.

**2. Core logic:**
- Uses `observable.box<ConfigKey>()` to track the currently selected view key reactively.
- Uses `reaction()` to watch for external store changes (e.g. data refresh from Mendix) and sync them back into the textarea's input string.
- Store methods: `store.setLayout(obj)`, `store.setConfig(obj)`, `store.setDataAt(key, value)`.
- Uses `runInAction()` when updating observable state inside non-MobX callbacks.

**3. Documentable behavior:**
- When the chart's underlying data updates externally (e.g. user changes a Mendix attribute), the playground textarea automatically reflects the new value via the `reaction()` subscription.
- This reactivity is absent in V1 (which uses plain React state).

**4. User-facing:**
The V2 editor stays in sync with live data changes — the developer does not need to manually refresh or reopen the editor to see updated values.

**5. New learnings:**
Using MobX `reaction()` vs React `useEffect()` for store-to-editor sync is a deliberate architectural choice: V2 stores are MobX observables, so tapping into MobX's reaction system is more reliable than trying to integrate with React's render cycle for external store mutations.

---

## ChartPlayground.editorConfig.ts

**1. Purpose:**
Studio Pro design-time configuration (validation, captions, data source helpers). Exported as `check()` function.

**2. Core logic:**
The `check()` function body is empty — it returns no errors and performs no validation.

**3. Documentable behavior:**
No design-time validation exists for this widget. Studio Pro will not show any property errors or warnings.

**4. User-facing:**
No editor config feedback in Studio Pro.

**5. New learnings:**
An empty `check()` is valid and expected when the widget has no properties to validate. This is consistent with the zero-property XML descriptor.

---

## ChartPlayground.editorPreview.tsx

**1. Purpose:**
Provides the Studio Pro Design Mode (canvas) preview for the widget.

**2. Core logic:**
Renders a minimal div containing only the toggle button element. Does not attempt to show a live playground or chart preview. `getPreviewCss()` returns the `Preview.scss` stylesheet.

**3. Documentable behavior:**
In Studio Pro canvas, the widget appears as just the toggle button — a small, visible marker indicating the playground is placed. No chart or editor functionality is shown in the modeler.

**4. User-facing:**
Studio Pro developers see only the toggle button div on the design canvas, which serves as a visual placement indicator.

**5. New learnings:**
The preview is intentionally minimal because the playground's full behavior (context injection, sidebar, JSON editors) is only meaningful at runtime — there is nothing useful to render at design time.

---

## ui/Playground.scss

**1. Purpose:**
Styles for the playground toggle button, info tooltip, and chart border indicator.

**2. Core logic:**
- Toggle button: `position: absolute; right: 10px; top: 10px; z-index: 50` — floats over the chart.
- Info tooltip button: `border-radius: 50%; width: 20px; height: 20px; background: #172c89` (dark blue circle).
- When `.playground-open` class is active: chart SVG receives a blue border (`border: 2px solid #172c89`) to indicate the playground is connected.

**3. Documentable behavior:**
- The toggle button is always visible regardless of chart content.
- The blue border on the chart SVG is a visual feedback mechanism showing the playground is active/open.
- The dark blue (#172c89) is used consistently for playground UI chrome.

**4. User-facing:**
Entirely user-facing — these styles define all visual appearance of the playground UI overlay.

**5. New learnings:**
Using the `.playground-open` class on a parent element to add a border to the chart SVG is a clean CSS-only approach to visualizing state without JavaScript DOM manipulation for styling.

---

## CHANGELOG.md

**1. Purpose:**
Version history for the chart-playground-web package.

**2. Core logic:**
N/A — documentation file.

**3. Documentable behavior:**
Key versions:
- **v1.0.1**: Fixed resize issue.
- **v1.1.0**: Bundling change (likely to shared-charts package).
- **v2.0.0**: Updated plotly.js dependency to v3 (breaking change in plotly major version).
- **v2.1.1**: Updated `@mendix/shared-charts` dependency (released 2025-07-15, latest).

**4. User-facing:**
The plotly.js v3 upgrade in v2.0.0 is the most significant external dependency change. Any charts using plotly APIs that changed between v2 and v3 would be affected.

**5. New learnings:**
The playground widget tracks the shared-charts dependency closely — v2.1.1 was released the same day as shared-charts updates, indicating tight coupling and coordinated releases.
