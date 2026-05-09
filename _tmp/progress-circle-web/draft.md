# Draft: progress-circle-web

Widget package: `packages/pluggableWidgets/progress-circle-web`

---

## src/ProgressCircle.tsx

**1. What is the purpose of this file?**
The root Mendix pluggable widget entry component. It bridges Mendix framework props to the internal `ProgressCircleComponent`, resolving input type (static/dynamic/expression) into numeric `currentValue`, `minValue`, and `maxValue` before passing them down.

**2. What kind of logic is described in this file?**
A switch on `props.type` routes to three separate value extraction paths. All values are coerced to `Number()`. Dynamic and expression modes apply defaults (currentValue=0, minValue=0, maxValue=100) when attribute/expression values are unavailable. A `useCallback`-wrapped click handler fires `props.onClick?.execute()`. Label computation branches on `showLabel`, `labelType`, and the computed percentage.

**3. What part of behavior can be documented from this file?**
- Three value input modes: `static` (IDE-configured integers), `dynamic` (entity attributes — Decimal/Integer/Long), `expression` (Mendix expressions returning Decimal).
- Dynamic and expression modes fall back to defaults (current=0, min=0, max=100) when Mendix values are not yet available.
- `onClick` fires only when `props.onClick` is defined; otherwise `undefined` is passed (no handler).
- Label types: `"percentage"` → computed percent string; `"text"` → `props.labelText?.value`; `"custom"` → `props.customLabel` (ReactNode widget content); `showLabel=false` → `null`.

**4. Is it user-facing?**
No — internal Mendix-to-component adapter.

**5. What new did you learn from this file?**
The component is architecturally identical to progress-bar-web: the same three-mode value resolution pattern and the same label type branching are used in both widgets, suggesting a shared design pattern across the Mendix widget library.

---

## src/components/ProgressCircle.tsx

**1. What is the purpose of this file?**
The presentation-layer React component that renders the visible SVG progress circle with optional label and click support.

**2. What kind of logic is described in this file?**
On mount, a `Circle` instance is created in a `useEffect` and `circle.animate(percentage)` is called (0–1 scale). On value change, a second `useEffect` re-animates to the new percentage. The component validates values via `getValuesErrorMessage()` — if invalid, an error `Alert` is rendered in place of the circle. CSS classes are applied conditionally: `"widget-progress-circle-clickable"` when `onClick` is provided. The label is rendered inside the circle's center text container.

**3. What part of behavior can be documented from this file?**
- Animation duration: 800 ms (from `defaultOptions`).
- Values outside the valid range (currentValue < minValue, currentValue > maxValue, maxValue < minValue) produce a visible runtime error alert — the circle is not rendered.
- The circle is clickable when an `onClick` handler is provided, applying a `"widget-progress-circle-clickable"` CSS class (enabling cursor: pointer).
- The label is rendered as the center text of the SVG shape.
- `Circle.destroy()` is called on component unmount to clean up DOM elements.

**4. Is it user-facing?**
Yes — this is the visible component users interact with.

**5. What new did you learn from this file?**
Unlike progress-bar-web which renders a linear bar, this component delegates all drawing to the `Circle` class (SVG-based), keeping the React component thin. The separation between the React lifecycle (mount/update/unmount) and SVG animation state is cleanly handled through the two `useEffect` hooks.

---

## src/components/Circle/Circle.ts

**1. What is the purpose of this file?**
SVG circle shape class extending `Shape`. Defines the SVG path template for a circular arc.

**2. What kind of logic is described in this file?**
`_pathString()` computes the SVG arc path for a circle centered at (50, 50) within a 100×100 viewBox. The radius is `50 - max(strokeWidth, trailWidth) / 2` to keep the stroke inside the viewBox. The path uses two semicircular arcs: "M 50,50 m 0,-{radius} a {radius},{radius} 0 1 1 0,{2*radius} a {radius},{radius} 0 1 1 0,-{2*radius}". `_trailString()` returns the same path for the background trail.

**3. What part of behavior can be documented from this file?**
- The circle is always centered at (50, 50) in a 100×100 SVG viewBox.
- Stroke width adjusts the radius so the stroke never overflows the viewBox boundaries.
- Both the progress path and the background trail use the same SVG arc path, so they are perfectly aligned.

**4. Is it user-facing?**
No — internal SVG geometry.

**5. What new did you learn from this file?**
The SVG two-semicircle arc technique is necessary because SVG does not support `arc` paths that are complete circles (a 360° arc degenerates to nothing). Splitting into two 180° arcs is the standard workaround.

---

## src/components/Circle/Shape.ts

**1. What is the purpose of this file?**
Base class for SVG progress shapes, adapted from the progressbar.js library. Manages the SVG element lifecycle, optional trail path, main progress path, and optional center text.

**2. What kind of logic is described in this file?**
Constructor: creates an SVG element (`viewBox="0 0 100 100"`), appends optional trail `<path>`, appends main `<path>`, initializes a `Path` animator, and optionally creates a center text `<p>` element absolutely positioned via CSS transform. Methods: `animate(value, callback)`, `set(value)`, `value()`, `stop()`, `pause()`, `resume()`, `setText(text)`, `destroy()`. After `destroy()`, any method call throws `"Object is destroyed"`.

**3. What part of behavior can be documented from this file?**
- The SVG is set to `width: 100%` so it scales with the container.
- The trail path (background circle) is optional — omitting it shows only the progress arc.
- Center text is auto-positioned using `position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%)`.
- Post-destroy usage is detected and throws an error — the object enforces proper lifecycle.
- `setText()` supports HTML content via `innerHTML`.

**4. Is it user-facing?**
No — internal SVG element management.

**5. What new did you learn from this file?**
The center text uses `innerHTML` injection (not React rendering), consistent with how maps-web and markdown-web inject HTML. This pattern appears throughout the Mendix widget library for content that must be injected into non-React DOM elements.

---

## src/components/Circle/Path.ts

**1. What is the purpose of this file?**
SVG path animation controller, adapted from progressbar.js. Animates stroke-dashoffset to produce a fill animation on an SVG path.

**2. What kind of logic is described in this file?**
Constructor: measures path total length, sets `stroke-dasharray` to `[length, length]` (making the full stroke visible but offset-able), and calls `set(0)` to start fully invisible. `set(progress)`: sets `stroke-dashoffset` to `length * (1 - progress)`. `animate(progress, options, callback)`: uses `shifty` library to tween dashoffset from current to target. Easing aliases: `"easeIn"` → `"easeInCubic"`, `"easeOut"` → `"easeOutCubic"`, `"easeInOut"` → `"easeInOutCubic"`. A `getBoundingClientRect()` call forces layout before animation begins (ensures smoothness).

**3. What part of behavior can be documented from this file?**
- The stroke-dasharray animation technique: progress is encoded as `dashoffset = length × (1 - progress)`.
- Animation uses `shifty` for tweening — a lightweight animation library.
- Easing names are aliased from generic names to cubic variants.
- A layout flush (`getBoundingClientRect`) before animation is a browser workaround for ensuring the first animation frame runs correctly.
- `stop()` leaves the path at its current animated position.

**4. Is it user-facing?**
No — internal animation engine.

**5. What new did you learn from this file?**
The `getBoundingClientRect()` layout flush before `shifty` animation is a subtle browser compatibility fix. Without it, browsers may batch the initial style update and the animation starting point can be wrong. This is a well-known workaround in SVG animation code.

---

## src/components/Circle/Utils.ts

**1. What is the purpose of this file?**
Utility functions for the Circle/Shape/Path classes, adapted from progressbar.js. Handles DOM manipulation, object extension, type guards, and float comparison.

**2. What kind of logic is described in this file?**
- `extend(to, from)`: Deep merge of `ShapeOptions` with special handling for `text` property.
- `render(template, data)`: String template substitution (`{key}` → value).
- `setStyle(element, prop, value)`: Sets a CSS property with vendor prefixes (Webkit, Moz, O, ms).
- `setStyles(element, styles)`: Bulk CSS application.
- `capitalize(str)`: First-letter uppercase for vendor prefix construction.
- Type guards: `isString()`, `isFunction()`, `isArray()`, `isObject()`.
- `forEachObject(obj, fn)`: Object key iteration.
- `floatEquals(a, b)`: Float comparison with epsilon 0.001.
- `removeChildren(el)`: DOM child removal.

**3. What part of behavior can be documented from this file?**
- Vendor prefixes (Webkit, Moz, O, ms) are applied to CSS properties — targeting older browser compatibility.
- Float comparison uses epsilon 0.001 — this tolerates small floating-point rounding differences in progress values.
- Template rendering is used in `Circle._pathString()` to substitute radius values.

**4. Is it user-facing?**
No — internal utilities.

**5. What new did you learn from this file?**
The vendor prefix logic was written for broad browser support (IE, old Firefox, old Safari). This is legacy code from progressbar.js that predates widespread CSS standard adoption, now carried forward. It's harmless but no longer necessary for modern browsers.

---

## src/components/Circle/Types.ts

**1. What is the purpose of this file?**
TypeScript type definitions for all Circle/Shape configuration objects.

**2. What kind of logic is described in this file?**
- `ShapeOptions`: color, strokeWidth, trailColor, trailWidth, fill, duration, easing, from/to (color interpolation), step (render callback), svgStyle, text (TextOptions), warnings.
- `TextOptions`: value, style, autoStyle, alignToBottom, className, paddingTop/Bottom, anchorY.
- `SVGStyle`: Partial CSS Properties.
- `ShapeRenderFunction`: `(from, to, factor, shape) => void`.

**3. What part of behavior can be documented from this file?**
- The animation supports color interpolation (`from`/`to` colors with a `step` render callback) — enabling color changes as progress increases.
- `alignToBottom` in TextOptions controls whether the text is anchored to the bottom of the shape.
- Warnings can be disabled via `warnings: false`.

**4. Is it user-facing?**
No — internal type definitions.

**5. What new did you learn from this file?**
The `step` render callback allows custom rendering logic on each animation frame — this enables advanced effects like changing the circle color based on progress percentage. The widget does not use this feature currently, but the infrastructure supports it.

---

## src/progressCircleValues.ts

**1. What is the purpose of this file?**
Defines the `ProgressCircleValues` interface and default values used for all three input modes.

**2. What kind of logic is described in this file?**
Interface: `{ currentValue: number; minValue: number; maxValue: number }`. Defaults: `currentValue=50, minValue=0, maxValue=100`.

**3. What part of behavior can be documented from this file?**
- The default currentValue is 50 (50% progress), not 0 — this makes the widget visible with a meaningful preview when no data is connected.
- The default range is 0–100, matching percentage semantics naturally.

**4. Is it user-facing?**
No — internal type and default definition.

**5. What new did you learn from this file?**
The default of 50% (not 0%) is a deliberate UX decision — it ensures the widget is visibly non-empty in the IDE structure preview and when data loads slowly.

---

## src/util.ts

**1. What is the purpose of this file?**
Provides `calculatePercentage(currentValue, minValue, maxValue)` — the same utility function as in progress-bar-web.

**2. What kind of logic is described in this file?**
Clamps `currentValue` to `[minValue, maxValue]`, then computes `Math.round(Math.abs((clamped - minValue) / (maxValue - minValue)) * 100)`. Returns an integer 0–100.

**3. What part of behavior can be documented from this file?**
- Out-of-range values are clamped (not errored) before percentage computation — the error display for out-of-range is handled at the component level separately.
- Result is always an integer — no decimal percentages in labels.
- Works with any numeric range, not just 0–100.

**4. Is it user-facing?**
No — internal utility.

**5. What new did you learn from this file?**
This file is identical (or near-identical) to `util.ts` in progress-bar-web, confirming code duplication between the two widgets. Both share the same percentage calculation logic.

---

## typings/ProgressCircleProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings from `ProgressCircle.xml`. Defines `ProgressCircleContainerProps` (runtime) and `ProgressCirclePreviewProps` (Studio preview).

**2. What kind of logic is described in this file?**
`ProgressCircleContainerProps`: `type: TypeEnum` ("static"|"dynamic"|"expression"), `staticCurrentValue/MinValue/MaxValue: number`, `dynamicCurrentValue/MinValue/MaxValue: EditableValue<Big>`, `expressionCurrentValue/MinValue/MaxValue: DynamicValue<Big>`, `showLabel: boolean`, `labelType: LabelTypeEnum` ("text"|"percentage"|"custom"), `labelText: DynamicValue<string>`, `customLabel: ReactNode`, `onClick?: ActionValue`.

**3. What part of behavior can be documented from this file?**
- Dynamic values use `EditableValue<Big>` — the `Big` type provides arbitrary precision decimal arithmetic, needed for decimal-typed Mendix attributes.
- Expression values use `DynamicValue<Big>` — expressions are read-only computed values.
- `onClick` is optional (not required by the widget).
- `customLabel` is a `ReactNode` — any valid React content can be embedded in the center.

**4. Is it user-facing?**
No — internal type declarations.

**5. What new did you learn from this file?**
The use of `Big` (from big.js) for all numeric values (even "dynamic" attributes) means the widget can handle very large or very precise decimal values without floating-point precision loss — important for financial or scientific use cases.

---

## src/ProgressCircle.xml

**1. What is the purpose of this file?**
Widget descriptor XML defining the widget's identity, props schema, and Studio Pro categorization. Generates `ProgressCircleProps.d.ts`.

**2. What kind of logic is described in this file?**
Declares: `type` property (enumeration: static/dynamic/expression); three value groups (static integers, dynamic attributes, expression formulas) each with current/min/max sub-properties; `showLabel` toggle; `labelType` enumeration (text/percentage/custom); `labelText` template; `customLabel` widget container; `onClick` action. System properties: Label, Visibility. Widget is: `needsEntityContext="true"`, `pluginWidget="true"`, `offlineCapable="true"`, categorized under "Display".

**3. What part of behavior can be documented from this file?**
- The widget is **offline capable** — consistent with client-side SVG rendering requiring no server calls.
- Static mode accepts only Integer type; dynamic mode accepts Integer, Decimal, or Long.
- Expression mode accepts Decimal expression.
- Custom label accepts a widget content container — allowing any Mendix widget(s) to be embedded as the circle's center label.
- Conditional visibility is supported via system property.

**4. Is it user-facing?**
Defines the developer-facing configuration interface in Studio/Studio Pro.

**5. What new did you learn from this file?**
Static mode restricts values to Integer only, while dynamic/expression modes accept Decimal and Long. This means for non-integer progress tracking (e.g., 75.5%) the developer must use dynamic or expression mode, not static.

---

## src/ProgressCircle.editorConfig.ts

**1. What is the purpose of this file?**
Provides `getProperties()` (conditional prop visibility), `check()` (IDE validation), `getPreview()` (structure preview image), and `getCustomCaption()` (page explorer label) for Studio Pro.

**2. What kind of logic is described in this file?**
`getProperties()`: Hides/shows static, dynamic, or expression property groups based on `type` selection. Hides label sub-properties when `showLabel=false`. Hides `customLabel` when `labelType≠"custom"`. `check()`: Validates static mode constraints (currentValue between min and max). Validates required attribute/expression fields for dynamic and expression modes. `getPreview()`: Returns an SVG asset path (dark-mode-aware via `isDarkMode` parameter) or `null` when `labelType="custom"`. `getCustomCaption()`: Returns `"[{type}, {currentValue}]"` for page explorer.

**3. What part of behavior can be documented from this file?**
- The IDE hides irrelevant value properties based on the selected input type — only one mode's properties are visible at a time.
- Label sub-properties (`labelText`, `customLabel`) are hidden when `showLabel=false`.
- The structure preview image switches between light and dark mode variants.
- When `labelType="custom"`, the structure preview returns `null` (cannot preview embedded widget content).
- IDE validates static mode ranges: currentValue must be within [minValue, maxValue].
- Page explorer shows the active input type and current value for quick identification.

**4. Is it user-facing?**
Yes — visible to developers in Studio Pro.

**5. What new did you learn from this file?**
The `getPreview()` returning `null` for `customLabel` mode is a deliberate design choice — the widget cannot render an arbitrary embedded widget in the structure preview, so it opts out entirely. The dark-mode-aware SVG asset switching uses the `isDarkMode` parameter from the preview context.

---

## src/ProgressCircle.editorPreview.tsx

**1. What is the purpose of this file?**
Provides the live React preview component for Studio Pro's design canvas.

**2. What kind of logic is described in this file?**
`asNumber(value, defaultValue)` helper safely converts string prop values to numbers. The component mirrors the three input modes from the container but uses string-typed preview props. When `labelType="custom"`, returns `null` (no preview). Otherwise renders `ProgressCircleComponent` with computed preview values, disabling `onClick`.

**3. What part of behavior can be documented from this file?**
- Design canvas preview renders a real `ProgressCircleComponent` with computed values — the circle actually renders in Studio Pro.
- Preview is non-interactive (`onClick=undefined`).
- Custom label mode returns `null` — the widget is invisible in design canvas when using custom labels.
- String values from preview props are safely coerced to numbers with fallback defaults.

**4. Is it user-facing?**
Yes — visible to developers in Studio Pro design canvas.

**5. What new did you learn from this file?**
Unlike markdown-web's editor preview which shows only a placeholder, progress-circle-web renders the actual SVG circle in design mode. This gives developers an accurate visual preview of the widget's appearance at configuration time.

---

## src/components/__tests__/ProgressCircle.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `ProgressCircleComponent` using Jest and React Testing Library.

**2. What kind of logic is described in this file?**
Tests: circle renders correctly; `Circle.animate()` is called with computed 0–1 value on mount and on change; `onClick` handler fires on click; custom min/max ranges calculate correctly (e.g., current=70, min=20, max=100 → 62.5%); clamping (below min → 0%, above max → 100%); `"widget-progress-circle-clickable"` class added when `onClick` provided; error alerts for constraint violations (currentValue<minValue, currentValue>maxValue, maxValue<minValue); label rendering (text/percentage/ReactNode/null).

**3. What part of behavior can be documented from this file?**
- `Circle` is mocked to isolate component from SVG rendering.
- `Circle.animate()` is verified to receive the 0–1 normalized percentage value.
- Error alerts are rendered using a Bootstrap danger `Alert` component.
- `"widget-progress-circle-clickable"` class is present only when `onClick` is provided.
- Null label (showLabel=false) renders no label element.
- Custom ReactNode labels are rendered via `setText()` equivalent.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The tests confirm that error states replace (not supplement) the circle — when an error occurs, only the alert is rendered, not the circle. This is different from progress-bar-web where both the error and the bar might coexist.

---

## e2e/ProgressCircle.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for the Progress Circle widget in a real Mendix test application.

**2. What kind of logic is described in this file?**
Tests: widget renders with correct percentage label; label updates when a textbox input changes the underlying attribute value; multiple label type variants (percentage, text, custom widget, no label) are tested on a dedicated Playground page. Screenshot baseline comparisons are used for visual regression. Session logout is performed after each test.

**3. What part of behavior can be documented from this file?**
- The widget updates its display reactively when the underlying Mendix attribute changes.
- Four label variant configurations are e2e-confirmed: percentage label, text label, custom widget label, and no label.
- The Playground test page hosts multiple widget variants by name (e.g., `progressCirclePercentage`, `progressCircleNegative`).
- Session logout is required after each test (Mendix 5-session license limit).

**4. Is it user-facing?**
The tested behaviors are user-facing.

**5. What new did you learn from this file?**
The "Negative" test variant (`progressCircleNegative`) suggests a test for negative or below-minimum value handling — confirming that clamping or error display is tested in a real Mendix environment, not just in unit tests.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents version history for the progress-circle-web widget.

**2. What kind of logic is described in this file?**
No logic — version history only. Versions: 3.3.3 (2026-02-10, license docs), 3.3.2 (2024-08-28, disabled "action required" warning), 3.3.1 (2023-09-27, removed redundant code), 3.3.0 (2023-08-02, 3rd party security fix), 3.2.0 (2023-06-05, page explorer caption update, dark mode icons), 3.1.0 (2021-12-23, dark mode structure preview), 3.0.1 (2021-12-03, design mode fix), 3.0.0 (2021-09-28, toolbox category and tile images), 2.1.0 (2021-07-07, structure preview added).

**3. What part of behavior can be documented from this file?**
- v3.3.2 disabled the "action required" warning in Studio Pro — previously the widget may have shown a spurious warning when no action was configured.
- v3.3.1 removed redundant code for browser load time improvements — performance-motivated cleanup.
- v3.2.0 changed the page explorer caption to `[type, current value]` format (confirmed by `getCustomCaption()`).
- No functional behavior changes since v2.1.0 (2021) — only maintenance and IDE integration improvements.

**4. Is it user-facing?**
The changelog is publicly visible on the Mendix Marketplace.

**5. What new did you learn from this file?**
The widget has been stable since v2.1.0 (July 2021). All post-v3.0 changes are IDE improvements, security updates, or maintenance — no runtime behavior changes. The widget is mature and its core functionality (SVG circle animation, three value modes, four label types) has not changed.
