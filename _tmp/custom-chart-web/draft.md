# Draft: custom-chart-web

Extracted from `packages/pluggableWidgets/custom-chart-web/` on 2026-05-08.

---

## src/CustomChart.xml

**1. Purpose:** Declares widget metadata and configurable properties. This widget accepts raw Plotly JSON — it is a low-level escape hatch for charts not supported by other chart widgets.

**2. Logic described:** Five property groups: "Data" (dataStatic multiline string, dataAttribute String attribute, sampleData, showPlaygroundSlot/playground), "Layout options" (layoutStatic, layoutAttribute, sampleLayout), "Configuration options" (configurationOptions JSON), "Dimensions" (widthUnit/width/heightUnit/height/minHeightUnit/minHeight/maxHeightUnit/maxHeight/OverflowY), "Events" (onClick action, eventDataAttribute String attribute). `heightUnit` has 4 values: percentageOfWidth (Auto), pixels, percentageOfParent (Percentage), percentageOfView (Viewport). There are also min/max height constraints with the same 4 units (plus "none"). A `devMode` property is commented out.

**3. Documentable behavior:** `dataAttribute` merges with and overwrites `dataStatic`. `sampleData` is used in preview or at runtime when no `dataAttribute` is set. Same pattern for layout: `layoutAttribute` overwrites `layoutStatic`; `sampleLayout` is fallback. `eventDataAttribute` stores raw Plotly click event data as JSON string. `configurationOptions` is raw Plotly Config JSON. The widget is `offlineCapable=true` but has NO `needsEntityContext` — it is standalone, not requiring a surrounding data view.

**4. User-facing:** Yes. Studio Pro property panel.

**5. New learnings:** This is the only chart widget that accepts raw Plotly JSON as a string — all other chart widgets use structured Mendix props. The `devMode` property was commented out (apparently removed before release). Min/max height constraints and viewport-relative height are unique to this widget vs. the structured chart widgets. The `eventDataAttribute` writes Plotly event data (curveNumber, pointNumber, x, y, z, lat/lon, etc.) as a JSON string for server-side processing.

---

## typings/CustomChartProps.d.ts

**1. Purpose:** Auto-generated TypeScript types from CustomChart.xml.

**2. Logic described:** `dataStatic`, `sampleData`, `layoutStatic`, `sampleLayout`, `configurationOptions` are all `string` (non-optional, empty string by default). `dataAttribute` and `layoutAttribute` are `EditableValue<string>` (optional). `onClick` is `ActionValue` (optional). `eventDataAttribute` is `EditableValue<string>` (optional). `HeightUnitEnum` has 4 values including "percentageOfView". Separate `MinHeightUnitEnum` and `MaxHeightUnitEnum` types with "none" as a valid option. `OverflowYEnum` is "auto" | "scroll" | "hidden".

**3. Documentable behavior:** Both `dataAttribute` and `layoutAttribute` are `EditableValue<string>` — they can be written as well as read, though in practice only read. `eventDataAttribute` being `EditableValue<string>` is intentional — the widget writes click event data into it. The overflow control (OverflowY) suggests the chart can overflow its container vertically and scroll.

**4. User-facing:** No. Internal TypeScript contract.

**5. New learnings:** Unlike other chart widgets, `customLayout` and `customConfigurations` are plain strings (not optional), meaning they always have a value (possibly empty). The separate min/max height types (vs. column-chart which doesn't have them) indicate this widget has more flexible layout control.

---

## src/CustomChart.tsx

**1. Purpose:** Entry-point container component. Renders the chart into a plain `<div>` using MobX-based controllers for chart lifecycle management.

**2. Logic described:** Uses `observer` from `mobx-react-lite` for reactivity. `getPlaygroundContext()` from shared-charts provides a MobX-based context for the playground slot. `useCustomChart` hook provides `playgroundData` and `ref`. `constructWrapperStyle` from shared-charts computes CSS dimensions. The chart mounts into a plain `<div className="widget-custom-chart">` via the `ref` callback — Plotly renders directly into the DOM element (imperative API, not React-managed).

**3. Documentable behavior:** The chart is rendered by Plotly into a vanilla DOM element — React does not manage the chart content. The `PlaygroundContext.Provider` wraps `props.playground` to enable the playground slot to communicate with the chart's store. The `tabIndex` is passed to the wrapper div, enabling keyboard focus. `constructWrapperStyle` applies width/height CSS.

**4. User-facing:** No. Internal rendering layer.

**5. New learnings:** Using Plotly's imperative API (not `react-plotly.js`) is intentional — this widget uses MobX `autorun` to detect data changes and imperatively call `Plotly.react()`. This avoids React re-rendering overhead for chart updates. The `observer` wrapper ensures the component re-renders when observable MobX state changes.

---

## src/hooks/useCustomChart.ts

**1. Purpose:** React hook that bridges Mendix props to the MobX controller architecture and manages component lifecycle.

**2. Logic described:** Creates a `GateProvider` (stable across re-renders) that wraps props for the controller. `useSetup` initializes `CustomChartControllerHost` with the gate. Updates gate props on every render via `useEffect(() => gateProvider.setProps(props))`. Computes `containerStyle` from width/height props (handles percentageOfWidth/pixels/percentageOfParent). Builds `playgroundData` as a MobX `computed` value with chart store and data. Returns merged refs from resize controller and Plotly controller.

**3. Documentable behavior:** `percentageOfWidth` height mode: uses `paddingBottom` trick (percentage-based responsive height). `pixels` mode: explicit `height` in px. `percentageOfParent`: `height: N%`. The resize controller and chart controller both need access to the DOM element via ref — `mergeRefs` combines them. Props are re-synced to gate on every render (not just on change), ensuring MobX observables always see current prop values.

**4. User-facing:** No. React hook internal to the widget.

**5. New learnings:** The `paddingBottom` trick for `percentageOfWidth` height mode is a CSS pattern that allows responsive square/ratio-based charts. There is a TODO comment: "replace with get-dimensions from widget-plugin-platform" — meaning this dimension calculation is a known code smell/tech debt pending cleanup.

---

## src/controllers/ChartPropsController.ts

**1. Purpose:** MobX observable controller that computes Plotly `data`, `layout`, and `config` from Mendix props and handles click events.

**2. Logic described:** Uses MobX `makeAutoObservable`. Computes `data` via `parseData(dataStatic, dataAttribute, sampleData)`. Computes `layout` by parsing and merging static + attribute layout, then injecting responsive dimensions from `sizeProvider` (width, height from ResizeController). Layout font/legend/axis sizes are scaled proportionally: `scale = Math.max(base * (width / 1000), min)`. Default margin: `{l: 60, r: 60, t: 60, b: 60, pad: 10}`. Config: merges parsed config with `{displayModeBar: false}`. Click handler: extracts point data (curveNumber, pointNumber, x, y, z, lat, lon, location, pointNumbers) from the first clicked point and serializes to JSON in `eventDataAttribute`.

**3. Documentable behavior:** `displayModeBar: false` by default — hides the Plotly toolbar (zoom, pan, etc.). Font scales with chart width — at 1000px, it's `12px`; at 500px, it's `8px` (minimum enforced). Layout is always merged with actual rendered dimensions (width/height from ResizeController), overriding any user-set values. Click data captures the first point only (`data.points[0]`). `executeAction(onClick)` fires after setting the event data attribute.

**4. User-facing:** No. Controller logic. The click behavior and layout appearance are user-visible effects.

**5. New learnings:** The responsive font scaling formula `max(base * (width/1000), min)` means charts on narrow screens get smaller text. The `displayModeBar: false` default means users cannot zoom or pan the chart — it's display-only by default (can be overridden via `configurationOptions`). The event data JSON includes geographic coordinates (lat/lon) and 3D coordinates (z), confirming this widget supports all Plotly chart types.

---

## src/controllers/PlotlyController.ts

**1. Purpose:** Manages the Plotly chart lifecycle — initialization, updates, and cleanup — using MobX `autorun`.

**2. Logic described:** `setChart` ref callback: on mount, creates a `PlotlyChart` instance and sets up `autorun(() => chart.update(props.get()), { delay: 100 })` — the 100ms delay debounces rapid prop updates. On unmount, disposes the autorun and calls `chart.destroy()`. Uses `ComputedAtom` from `widget-plugin-mobx-kit` to track props.

**3. Documentable behavior:** Chart updates are debounced by 100ms via MobX `autorun` delay. When the component unmounts (e.g., conditional visibility hiding the chart), Plotly's `purge()` is called to clean up. This controller + autorun pattern means chart re-renders when any MobX-observed value in the `ChartPropsController` changes (data, layout, config).

**4. User-facing:** No. Internal chart lifecycle management.

**5. New learnings:** The 100ms debounce prevents chart from re-rendering on every keystroke if data comes from a user-input attribute. The CHANGELOG v6.3.0 fix for "conditional visibility" charts not re-drawing was likely fixed here (purge + re-init on mount).

---

## src/components/PlotlyChart.ts

**1. Purpose:** Plain class wrapping Plotly.js imperative API for chart initialization, updates, and cleanup.

**2. Logic described:** Uses `plotly.js-dist-min` (minimal distribution). `init()` calls `Plotly.newPlot(element, data, layout, config)` and attaches `plotly_click` event listener. `update()` updates internal data/layout/config and calls `Plotly.react()` (efficient diff-update). `destroy()` calls `Plotly.purge()` to remove chart and free memory.

**3. Documentable behavior:** `Plotly.react()` is used for updates (not `Plotly.newPlot` again) — this is Plotly's recommended efficient update method that preserves zoom state and only re-renders changed traces. Click events are bound to `plotly_click`, not the DOM `click` event. Only ONE click listener is attached (at init time) — it cannot be updated without recreating the chart.

**4. User-facing:** No. Internal Plotly wrapper.

**5. New learnings:** Using `plotly.js-dist-min` (minimal build) reduces bundle size vs full `plotly.js`. The single `plotly_click` listener limitation means if `onClick` callback changes (e.g., different action), the old listener still fires. This is a behavioral constraint — the onClick action is effectively frozen at chart creation time until the chart is recreated.

---

## src/utils/utils.ts

**1. Purpose:** Parses and merges Plotly data, layout, and config from static strings and dynamic attribute strings.

**2. Logic described:** `deepmergePlotly` uses `deepmerge` with `arrayMerge: (_target, src) => src` — arrays are replaced (not concatenated). `sharesDataKeys` checks if two traces both have plotable data keys (x, y, z, values, labels, lat, lon, etc.) — if both have data, they are treated as independent traces. `parseData`: merges static and dynamic traces by index; if same-index traces share data keys, adds both as separate traces; otherwise merges them. `parseLayout`/`parseConfig`: simple JSON parse with deepmerge fallback.

**3. Documentable behavior:** Data merge strategy (v6.3.0 breaking change): traces are merged BY INDEX — trace 0 of static + trace 0 of attribute = one merged trace (attribute overrides static). Exception: if both traces have their own data keys (e.g., both have `x` and `y`), they are added as separate traces. This allows: (a) static to define trace styling + attribute to provide data points, or (b) both to be independent traces. JSON parse errors fall back to empty arrays/objects with console.error — the chart renders empty rather than crashing.

**4. User-facing:** No. Utility functions. Data merge behavior directly affects user-visible chart content.

**5. New learnings:** The "arrays replaced, not concatenated" merge is critical for Plotly data — concatenating `x: [1,2]` + `x: [3,4]` would give `[1,2,3,4]`, but replacing gives `[3,4]`. This is the correct behavior for dynamic data overriding static data. The `DATA_KEYS` set includes financial chart keys (`open`, `high`, `low`, `close`) and polar keys (`r`, `theta`), confirming the widget supports candlestick, OHLC, and polar charts.

---

## CHANGELOG.md

**1. Purpose:** Release history from v1.0.0 to v6.3.0 (latest 2026-02-17).

**2. Logic described:** v6.3.0 (2026-02-17): Breaking change — changed data merge strategy from "append traces as separate elements" to "merge by index". v1.2.3: event data now returns more properties (curveNumber, pointNumber, coordinates) beyond bbox. v1.2.2: fixed eventDataAttribute setValue not working. v1.2.1: fixed parsing/merging of layout and data. v1.2.0: shared code update. v1.0.1: fixed static layout not applying. v1.0.0 (2025-02-28): introduced custom chart.

**3. Documentable behavior:** This is a relatively new widget (v1.0.0 in Feb 2025). The v6.3.0 merge strategy change is a BREAKING CHANGE for existing users — apps using this widget before v6.3.0 that relied on the "append" behavior must update their data configuration. The event data fix (v1.2.2) confirms that the `setValue` on `eventDataAttribute` was previously broken. The conditional visibility fix (v6.3.0) is critical for widgets shown/hidden via visibility rules.

**4. User-facing:** No. Developer documentation.

**5. New learnings:** The version jump from v1.2.3 to v6.3.0 suggests the widget was re-versioned to align with the shared charts version family. The breaking change in v6.3.0 requires documentation as a constraint: users upgrading must verify their data/attribute merge configuration.
