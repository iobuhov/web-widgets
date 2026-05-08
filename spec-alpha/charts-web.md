# Charts (charts-web)

## Purpose

The charts-web package is a **bundle meta-package**, not an individual widget. It aggregates 10 independently versioned Mendix chart widgets into a single distributable `Charts.mpk` for the Mendix Marketplace (app #105695). The package contains no runtime widget code of its own; its sole function is to produce the combined MPK artifact used for Marketplace distribution and app import. Minimum Mendix version: 10.6.0.

## User Scenarios

### [P1] Import the Charts module into a Mendix app
**Given** a developer imports `Charts.mpk` into a Mendix Studio Pro project  
**When** the import completes  
**Then** all 10 chart widgets become available in the Studio Pro widget toolbox  

#### Edge Cases
- All 10 widgets are registered as a unit; there is no mechanism to import a subset of the bundle
- The bundle requires Mendix 10.6.0 or higher

### [P2] Use an individual chart widget from the bundle
**Given** the Charts module has been imported into a Mendix app  
**When** the developer drags a chart widget onto a page  
**Then** the widget behaves identically to if it had been installed as a standalone MPK  

#### Edge Cases
- Each widget within the bundle has its own semantic version independent of the bundle version
- Individual widget behavior is defined by the respective widget spec (e.g., `area-chart-web.md`)

## Functional Requirements

- FR-001: The package MUST produce a `Charts.mpk` archive that, when imported into Mendix Studio Pro, registers all 10 bundled widgets.
- FR-002: The bundle build MUST merge each widget's `com/` file tree into a single flat MPK using a repack process (unzip → merge → rezip).
- FR-003: The merged MPK MUST include a single `src/package.xml` that lists all 10 widget XML descriptors and their corresponding file directories.
- FR-004: In development mode (non-production build with `MX_PROJECT_PATH` set), the build MUST copy the output MPK directly to the Mendix test project's `widgets/` directory.
- FR-005: The package MUST declare `reactReady: true` in its Marketplace configuration.

## Bundled Widgets

| Widget | Description |
|--------|-------------|
| `area-chart-web` | Plotly area chart |
| `bar-chart-web` | Plotly bar chart |
| `bubble-chart-web` | Plotly bubble chart |
| `column-chart-web` | Plotly column chart |
| `custom-chart-web` | Raw Plotly JSON configuration chart (added v6.0.0) |
| `heatmap-chart-web` | Plotly heatmap chart |
| `line-chart-web` | Plotly line chart |
| `pie-doughnut-chart-web` | Plotly pie/donut chart |
| `time-series-chart-web` | Plotly time series chart |
| `chart-playground-web` | Developer-mode JSON editor overlay (extracted to own widget in v5.0.0) |

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| *(none)* | — | — | — | This package contains no configurable runtime widget. All properties belong to the individual bundled widgets. |

## Changelog

**6.3.0** (2026-02-17) — CustomChart: "Static" and "Source attribute" trace data now merged by index (source overrides static at the same index) rather than appended. Fixed conditional visibility re-draw issue.

**6.0.0** (2025-02-28) — Plotly.js upgraded to v3.0 across all widgets (breaking change). CustomChart widget introduced.

**5.0.0** (2024-06-14) — "Enable developer mode" property removed; playground extracted to the standalone `chart-playground-web` widget. Migrated from Ace editor to CodeMirror.

## Open Questions

> Could not be determined from source code alone — requires human review

- [ ] How are individual widget version numbers coordinated with the bundle version during a release?
- [ ] What is the policy for supporting older Mendix versions (currently requires 10.6.0) when individual widgets may support older versions?
- [ ] The unreleased changelog notes that CodeMirror is disabled in the playground for compatibility — what compatibility issue triggered this, and when will it be re-enabled?
