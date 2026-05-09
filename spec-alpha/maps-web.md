# Maps

## Purpose

The Maps widget renders an interactive map with configurable markers on a Mendix page. It supports four map providers — Google Maps, OpenStreetMap, Mapbox, and HERE Maps — and accepts both static markers (configured at design time) and dynamic markers (driven by a Mendix data source). The widget resolves address-based markers via geocoding, optionally detects the user's current location, and provides interactive marker click actions.

## User Scenarios

### [P1] Display dynamic markers from a data source
**Given** a Mendix page with a Maps widget bound to a data source entity containing latitude, longitude, and optional title attributes  
**When** the page loads  
**Then** the map renders all data source items as markers at their respective positions, using the map provider configured in Studio Pro

#### Edge Cases
- If the data source status is not `Available` (loading or error), no markers are shown until data loads.
- Custom marker images in a dynamic list are shared across all items in that list; all items display the same icon.
- Dynamic latitude/longitude attributes must be `Decimal` type (`Big`); integer or string types are not supported.

### [P2] Display static address-based markers
**Given** static markers configured with address strings rather than explicit lat/lng coordinates  
**When** the widget initializes  
**Then** each address is geocoded via the Google Geocoding API and a marker is rendered at the resolved position

#### Edge Cases
- Geocoding results are cached globally per browser session (`window.mxGMLocationCache`); the same address is never geocoded twice.
- If an address cannot be geocoded, the marker is silently dropped (no error shown to the end user).
- A Geocoding API key (`geodecodeApiKey` or `geodecodeApiKeyExp`) is mandatory when any marker uses address-based location, regardless of which map provider is displayed.

### [P3] Show user's current location
**Given** `showCurrentLocation` is enabled and the browser grants geolocation permission  
**When** the map renders  
**Then** a fixed blue pin icon appears at the user's current GPS position

#### Edge Cases
- If geolocation is unsupported or the user denies permission, the current location marker is silently omitted.
- The current location icon is a hardcoded base64 PNG and is not configurable.
- There is no geolocation timeout; the browser default applies.

### [P4] Marker click action with info popup
**Given** a static or dynamic marker is configured with a title and/or an onClick action  
**When** the user clicks a Leaflet-based marker (OpenStreetMap, Mapbox, HERE Maps)  
**Then** a popup appears showing the marker title; clicking the title text within the popup fires the configured onClick action (two-click interaction)

#### Edge Cases
- For Google Maps, clicking an `AdvancedMarker` with both a title and an onClick toggles an `InfoWindow`; clicking the title text (with pointer cursor) fires the action.
- A marker with neither a title nor an onClick is non-interactive (Leaflet `interactive: false`).
- Leaflet markers with ARIA role `"button"` are keyboard-accessible when `interactive` is true.

## Functional Requirements

- FR-001: The system MUST support four map providers: Google Maps, OpenStreetMap, Mapbox, and HERE Maps.
- FR-002: OpenStreetMap MUST function without an API key; all other providers require one.
- FR-003: The widget MUST support static markers (lat/lng or address) and dynamic markers (data source with `Decimal` lat/lng attributes).
- FR-004: Address-based markers MUST be geocoded via the Google Geocoding API regardless of the chosen map provider.
- FR-005: Geocoding results MUST be cached in `window.mxGMLocationCache` for the duration of the browser session.
- FR-006: API keys MUST support both a static string and a Mendix expression variant; the expression variant takes precedence (`apiKeyExp?.value ?? apiKey`).
- FR-007: The widget MUST support five named zoom levels: world (1), continent (5), city (10), street (15), buildings (20). When `zoom` is `"automatic"`, bounds-fitting is used instead.
- FR-008: When `autoZoom` is true, both Google Maps and Leaflet MUST fit all marker bounds; the fallback center is Rotterdam (51.906688, 4.48837).
- FR-009: Google Maps MUST use `AdvancedMarker` (not the deprecated `Marker`); a `mapId` prop is required.
- FR-010: Mapbox tiles MUST use the `streets-v11` style; the tile zoom offset MUST be corrected (`zoomOffset: -1`, tile size 512).
- FR-011: HERE Maps MUST support both legacy `"app_id,app_code"` (comma-delimited) and modern single-`apiKey` authentication formats.
- FR-012: Custom Leaflet marker icons MUST be centered via CSS `transform: translate(-50%, -50%)` on a zero-dimension `DivIcon` container to avoid hitbox misalignment.
- FR-013: Mapbox and HERE Maps providers MUST display a warning in Studio design mode that preview is not available without an API key.

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|
| `mapProvider` | `"googleMaps" \| "openStreet" \| "mapBox" \| "hereMaps"` | — | Map provider | Selects the tile/rendering library. Hidden in Studio without advanced mode. |
| `apiKey` | `string` | `""` | API key | Static API key for Google Maps, Mapbox, or HERE Maps. |
| `apiKeyExp` | `DynamicValue<string>?` | — | API key (expression) | Expression-based API key; takes precedence over `apiKey`. |
| `geodecodeApiKey` | `string` | `""` | Geo Location API key | API key for the Google Geocoding API; required when any marker uses an address. |
| `geodecodeApiKeyExp` | `DynamicValue<string>?` | — | Geo Location API key (expression) | Expression-based geocode key; takes precedence. Visible only when address markers are present. |
| `googleMapId` | `string` | — | Map ID | Required for Google Maps `AdvancedMarker`. Mandatory from v4.0.0. |
| `markers` | `MarkersType[]` | `[]` | Static markers | List of markers with static coordinates or addresses. |
| `dynamicMarkers` | `DynamicMarkersType[]` | `[]` | Dynamic markers | List of marker data sources; lat/lng must be `Decimal` attributes. |
| `zoom` | `ZoomEnum` | — | Zoom | Named zoom level (world/continent/city/street/buildings/automatic). |
| `autoZoom` | `boolean` | — | Auto zoom | Derived from `zoom === "automatic"`; triggers bounds-fitting. |
| `optionZoomControl` | `boolean` | — | Show zoom controls | Displays zoom in/out controls on the map. |
| `optionDrag` | `boolean` | — | Draggable | Enables map drag (`gestureHandling:"auto"` for Google; `dragging` for Leaflet). |
| `optionScroll` | `boolean` | — | Scroll to zoom | Enables scroll wheel zoom. |
| `showCurrentLocation` | `boolean` | `false` | Show current location | Shows a blue pin at the user's GPS location if browser permission is granted. |
| `attributionControl` | `boolean` | — | Show attribution | Shows tile attribution. Available for Leaflet providers only. |
| `markerStyle` | `"default" \| "image"` | — | Marker style | Hidden in Studio without advanced mode. |
| `customMarker` | `DynamicValue<string>?` | — | Custom marker image | Image URL for the default custom marker icon. Visible when `markerStyle === "image"`. |

## Changelog

**v4.1.0 (2025-10-29):** Fixed rendering failures in some cases. Fixed array-index list rendering bug causing issues when filtering large marker lists. Fixed console warning for titled Leaflet markers.

**v4.0.0 (2024-05-02):** Fixed comma-decimal parsing in lat/lng values. Replaced deprecated Google Maps `Marker` with `AdvancedMarker`. Removed `mapsStyles` property. Added required `mapId` property for Google Maps (breaking change).

**v3.0.1 (2021-10-13):** "Enable advanced options" restricted to Studio only (not Studio Pro). Renamed "Google maps API Key" to "Geo Location API key". Made Geo Location API key required for address markers.

## Open Questions
> Could not be determined from source code alone — requires human review
- [ ] Is there a supported limit on the number of simultaneous markers before performance degrades?
- [ ] Does Mapbox `streets-v11` remain the intended style, or is the style configurable via a future prop?
- [ ] What happens to HERE Maps legacy authentication when the app_id/app_code scheme is deprecated by HERE?
