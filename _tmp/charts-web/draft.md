# charts-web — Source Extraction Draft

Package: `@mendix/charts-web` v6.3.0  
Type: **bundle package** (not an individual widget — bundles 10 chart widgets into a single MPK)  
Marketplace app #105695, min Mendix 10.6.0

---

## package.json

**1. Purpose:** Package manifest for the charts-web bundle. Defines it as a module-type `mxpackage` that aggregates 10 individual chart widget packages.

**2. Logic:** `mxpackage.type: "widget"` with `changelogType: "module"` — it's treated as a widget package but has module-level changelog. The `dependencies` field lists all 10 bundled widgets:
- `@mendix/area-chart-web`
- `@mendix/bar-chart-web`
- `@mendix/bubble-chart-web`
- `@mendix/column-chart-web`
- `@mendix/custom-chart-web`
- `@mendix/heatmap-chart-web`
- `@mendix/line-chart-web`
- `@mendix/pie-doughnut-chart-web`
- `@mendix/time-series-chart-web`
- `@mendix/chart-playground-web`

All use `workspace:*` — resolved from the local monorepo. Output artifact: `Charts.mpk`.

**3. Behavioral documentation:** This package has no runtime widget code of its own. Its sole job is to produce a combined `Charts.mpk` containing all 10 chart widgets, which is what gets distributed to the Mendix Marketplace. `reactReady: true` in marketplace config.

**4. User-facing?** No — build infrastructure.

**5. New learnings:** This "bundle" pattern aggregates independently versioned packages into a single distributable MPK. Each widget can be versioned independently (e.g., CustomChart 1.2.3 while AreaChart 6.2.1) and the bundle version tracks the max or an overall semantic versioning across all. Min Mendix version is 10.6.0 (newer than individual widgets like carousel-web which requires only 9.6.0).

---

## src/package.xml

**1. Purpose:** Mendix client module descriptor — the manifest embedded inside the final MPK that tells the Mendix runtime which widget XML files and file directories are included.

**2. Logic:** Lists 10 widget files (by path relative to their unpacked location):
- `AreaChart/AreaChart.xml`
- `BarChart/BarChart.xml`
- `BubbleChart/BubbleChart.xml`
- `ColumnChart/ColumnChart.xml`
- `CustomChart/CustomChart.xml`
- `HeatMap/HeatMap.xml`
- `LineChart/LineChart.xml`
- `PieChart/PieChart.xml`
- `TimeSeries/TimeSeries.xml`
- `ChartPlayground/ChartPlayground.xml`

And 10 corresponding `<file>` entries pointing to `com/mendix/widget/web/{widgetdir}` directories.

**3. Behavioral documentation:** This XML is what the Mendix client uses to register all 10 widgets. When the Charts.mpk is imported into a Mendix app, all 10 chart widgets become available in Studio Pro's widget toolbox.

**4. User-facing?** No — Mendix build artifact descriptor.

**5. New learnings:** All widgets use `com.mendix.widget.web` as their `packagePath` (the Mendix Java/JavaScript namespace). The `clientModule` version matches the bundle version (6.3.0).

---

## scripts/build.ts

**1. Purpose:** Build script that assembles `Charts.mpk` from the 10 individual widget MPKs.

**2. Logic:**
1. Lists all dependency packages via `listPackages(dependencies)`
2. For each dependency, reads `getWidgetInfo` (name, version, mpk path)
3. Creates a `tmp` directory
4. For each widget: unzips its MPK, copies the `com/` source tree into the shared tmp, then removes the individual `com/` and `package.xml` to flatten the structure
5. Copies the bundle's own `src/package.xml` into the merged tmp
6. Copies the LICENSE file
7. Zips the merged tmp into `Charts.mpk`

In dev mode (not production AND `MX_PROJECT_PATH` is set): copies the output MPK directly to the Mendix test project's `widgets/` directory.

**3. Behavioral documentation:** The assembly process merges all widget JS bundles and XML files into one MPK with a single `package.xml`. This means the final MPK is a flat merged archive — not nested MPKs inside a zip.

**4. User-facing?** No — build tooling.

**5. New learnings:** The build uses `@mendix/automation-utils` helpers (`unzip`, `zip`, `cp`, `mkdir`, `rm`). The hot-reload dev mode copies the MPK directly to a local Mendix project, enabling rapid iteration. The "repack" pattern (unzip → merge → rezip) is specific to bundle packages; individual widget builds use `pluggable-widgets-tools` directly.

---

## CHANGELOG.md

**1. Purpose:** Release history for the Charts bundle, tracking changes per sub-widget per release.

**2. Logic:** Key versions:
- **v6.3.0** (2026-02-17): CustomChart breaking change — "Static" and "Source attribute" trace data now merged by index (source overrides static at same index) rather than appended. Fixed conditional visibility re-draw.
- **v6.2.3**: CustomChart returns more event properties beyond bbox coordinates.
- **v6.2.2**: CustomChart fixed event data attribute value not setting.
- **v6.2.1**: All charts update shared-charts dep; HeatMap fixed onClick (datasource + selection); CustomChart fixed layout/data parsing.
- **v6.2.0**: Aggregation fix for Plotly 3.0 across all standard charts.
- **v6.0.0** (2025-02-28): **Plotly.js upgraded to v3.0** across all widgets. CustomChart widget introduced.
- **v5.0.0** (2024-06-14): "Enable developer mode" property removed; playground extracted to separate `chart-playground-web` widget. Migrated from Ace editor to CodeMirror.
- **v4.0.0** (2022-10-27): Removed all deprecated (legacy) chart widgets.
- **Unreleased**: CodeMirror disabled in playground for compatibility; only basic editor available.

**3. Behavioral documentation:** The bundle's changelog is composite — it tracks breaking changes and fixes per individual sub-widget. Aggregation was a recurring pain point across Plotly 2→3 migration. CustomChart is the newest addition (v6.0.0 = Feb 2025), with the most active recent development. The playground's development experience has been degraded (CodeMirror disabled) in the unreleased version.

**4. User-facing?** No — developer documentation.

**5. New learnings:** Individual widgets within the bundle have their own semantic version numbers that don't align with the bundle version (e.g., CustomChart started at 1.0.0 when bundle was 6.0.0). The Plotly 3.0 migration (v6.0.0 bundle) was significant enough to require a breaking-change version bump. The `fast-json-patch` dependency appears in older entries (v4.0.3), suggesting it was used for JSON merging before being replaced.

---

## README.md

**1. Purpose:** Minimal internal documentation for the bundle automation.

**2. Logic:** One paragraph: this is the automation package for the Charts module release. Points to CHANGELOG.md for change history.

**3. Behavioral documentation:** Not user documentation — internal note for maintainers.

**4. User-facing?** No.

**5. New learnings:** The README explicitly calls this "module automation" rather than a widget, confirming the bundle/meta-package nature. External users interact with individual chart widgets; the bundle is primarily a Marketplace distribution artifact.

---

## Summary

**Package:** charts-web v6.3.0 (`@mendix/charts-web`) — a **bundle/meta-package**, not a widget itself.

**Purpose:** Assembles 10 independently versioned Mendix chart widgets into a single `Charts.mpk` for Marketplace distribution:

| Widget | Notes |
|--------|-------|
| area-chart-web | Plotly area chart |
| bar-chart-web | Plotly bar chart |
| bubble-chart-web | Plotly bubble chart |
| column-chart-web | Plotly column chart |
| custom-chart-web | Raw Plotly JSON config (added v6.0.0) |
| heatmap-chart-web | Plotly heatmap |
| line-chart-web | Plotly line chart |
| pie-doughnut-chart-web | Plotly pie/donut |
| time-series-chart-web | Plotly time series |
| chart-playground-web | Developer mode editor (added v5.0.0) |

**Architecture:**
- No runtime code — purely a build/packaging artifact
- `scripts/build.ts`: unpacks each widget MPK → merges file trees → rezips as `Charts.mpk`
- `src/package.xml`: the merged Mendix module descriptor for all 10 widgets

**Key history:**
- v4.0.0: Removed legacy deprecated widgets
- v5.0.0: Extracted playground into its own widget (`chart-playground-web`); CodeMirror editor
- v6.0.0: Plotly.js v3.0 upgrade (breaking); CustomChart widget introduced
- v6.3.0 (latest): CustomChart merge behavior changed (index-based override vs. append)
- Unreleased: CodeMirror disabled for compatibility (basic editor only)

**No tests, no runtime source** — all behavior lives in the 10 sub-widget packages.
