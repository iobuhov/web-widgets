# Draft: maps-web

Widget package: `packages/pluggableWidgets/maps-web`

---

## src/Maps.tsx

**1. What is the purpose of this file?**
This is the root React component exported by the widget. It is the entry point that Mendix calls when rendering the Maps widget. It orchestrates prop reception, location resolution (geocoding), current-user-location detection, and delegates rendering to `MapSwitcher`.

**2. What kind of logic is described in this file?**
Two side-effect hooks: (a) `useLocationResolver` converts raw marker props (static + dynamic) into resolved `Marker[]` objects (geocoding address-based markers if needed); (b) a `useEffect` that calls `getCurrentUserLocation()` when `showCurrentLocation` is true and stores the result in state. The resolved locations and optional current-location marker are passed as props to `MapSwitcher`.

**3. What part of behavior can be documented from this file?**
- The widget resolves both static and dynamic markers before rendering.
- Current-location detection is conditional on `showCurrentLocation` prop; errors are silently swallowed (console.error only).
- API key falls back: `apiKeyExp?.value ?? apiKey` (expression takes precedence over static key). Same fallback for geodecode key.
- Zoom is translated via `translateZoom(props.zoom)` — the string enum value is converted to a numeric level.
- The widget imports `leaflet/dist/leaflet.css` globally, so Leaflet CSS is always loaded regardless of the chosen map provider.

**4. Is it user-facing?**
Indirectly — it is the root component but renders no DOM itself. All visible output comes from `MapSwitcher` and its children. The user-facing effect is map rendering with markers.

**5. What new did you learn from this file?**
Expression-based API keys take precedence over static string keys (`apiKeyExp?.value ?? apiKey`). The geodecode key follows the same fallback pattern, allowing dynamic key injection at runtime via Mendix expressions.

---

## typings/MapsProps.d.ts

**1. What is the purpose of this file?**
Auto-generated TypeScript typings derived from `Maps.xml`. Defines all props accepted by the widget container (`MapsContainerProps`) and design-mode preview component (`MapsPreviewProps`), plus nested types for static markers (`MarkersType`), dynamic markers (`DynamicMarkersType`), and all enums.

**2. What kind of logic is described in this file?**
No logic — purely type declarations. Defines enums: `LocationTypeEnum` ("address" | "latlng"), `MarkerStyleEnum` ("default" | "image"), `ZoomEnum` (automatic/world/continent/city/street/buildings), `MapProviderEnum` (googleMaps/openStreet/mapBox/hereMaps), `WidthUnitEnum`, `HeightUnitEnum`.

**3. What part of behavior can be documented from this file?**
- Static markers (`MarkersType`) accept either an address (text template → `DynamicValue<string>`) or lat/lng (text templates → `DynamicValue<string>`), not both simultaneously.
- Dynamic markers (`DynamicMarkersType`) use a `ListValue` data source; latitude/longitude are `ListAttributeValue<Big>` (Decimal Mendix type).
- There are two API key props for each key (`apiKey`/`apiKeyExp` and `geodecodeApiKey`/`geodecodeApiKeyExp`) — the expression variant is optional and takes precedence.
- `MapsPreviewProps` notes that `className` is deprecated since Mendix 9.18.0; use `class` instead.

**4. Is it user-facing?**
No — this is a type declaration file only visible to the developer configuring the widget. It constrains what Mendix Studio/Studio Pro exposes in the widget editor.

**5. What new did you learn from this file?**
Dynamic marker latitude/longitude attributes must be `Decimal` type (backed by `Big`), not integer or string. Static marker lat/lng come in as string text templates and are parsed in `data.ts`.

---

## typings/shared.d.ts

**1. What is the purpose of this file?**
Defines shared TypeScript interfaces used across multiple components: `ModeledMarker` (intermediate representation before geocoding), `Marker` (resolved, renderable marker with numeric lat/lng), and `SharedProps` (common props for both `GoogleMap` and `LeafletMap`).

**2. What kind of logic is described in this file?**
No runtime logic — pure type definitions. `Marker` requires numeric `latitude`, `longitude`, and `url` (marker icon URL, may be empty string). `SharedProps` composes `Dimensions` from `@mendix/widget-plugin-platform` with map-specific controls.

**3. What part of behavior can be documented from this file?**
- Both map implementations share the same set of interaction props: `autoZoom`, `optionZoomControl`, `zoomLevel`, `optionDrag`, `optionScroll`, `showCurrentLocation`, `currentLocation?`, and `locations`.
- `mapsToken` (the API key passed to the map library) is optional — OpenStreetMap works without one.
- `ModeledMarker` includes an `action` callback (optional) which is the marker click handler before it's resolved; `Marker` uses `onClick` after resolution.

**4. Is it user-facing?**
No — internal type contracts between components.

**5. What new did you learn from this file?**
The `Marker.url` field is always a string (never undefined) but may be empty string — the rendering components check for truthiness to decide whether to render a custom image or a default pin.

---

## src/components/MapSwitcher.tsx

**1. What is the purpose of this file?**
A thin routing component that selects the appropriate map implementation based on `mapProvider` prop. If `googleMaps`, renders `GoogleMapContainer`; otherwise renders `LeafletMap` (covering openStreet, mapBox, hereMaps).

**2. What kind of logic is described in this file?**
A single conditional expression: `props.mapProvider === "googleMaps" ? <GoogleMapContainer {...props} /> : <LeafletMap {...props} />`. The component accepts a union of `GoogleMapsProps` and `LeafletProps` via `SwitcherProps extends GoogleMapsProps, LeafletProps`.

**3. What part of behavior can be documented from this file?**
All three non-Google providers (OpenStreet, Mapbox, HERE Maps) are rendered through the same `LeafletMap` component — provider-specific tile URLs are handled inside `LeafletMap` via the `baseMapLayer` utility. Google Maps uses an entirely separate implementation (`@vis.gl/react-google-maps`).

**4. Is it user-facing?**
Not directly. This is internal routing logic with no DOM output.

**5. What new did you learn from this file?**
The architectural split is binary: one code path for Google Maps (uses Google Maps JS SDK via `@vis.gl/react-google-maps`), another for everything else (Leaflet-based). OpenStreetMap, Mapbox, and HERE Maps all share the Leaflet rendering path and differ only in tile URL configuration.

---

## src/components/GoogleMap.tsx

**1. What is the purpose of this file?**
Implements the Google Maps rendering path. `GoogleMapContainer` wraps the map in an `APIProvider` (which loads the Google Maps JS SDK). The inner `GoogleMap` component renders the map canvas with all markers and handles auto-zoom. `GoogleMapsMarker` renders individual markers using `AdvancedMarker` and optionally shows an `InfoWindow` popup when the marker has a title.

**2. What kind of logic is described in this file?**
- Auto-zoom: on every change to `locations` or `currentLocation`, computes `LatLngBounds` from all markers and calls `map.fitBounds(bounds)` when `autoZoom` is true; otherwise centers on the bounds center. Falls back to Rotterdam coordinates if bounds is empty.
- Marker click behavior: clicking a marker with a title toggles an `InfoWindow`; clicking a marker with an `onClick` fires the action. Both can be combined.
- Shows a loading spinner (`<div className="spinner" />`) while the Google Maps API is loading (`isLoaded` from `useApiIsLoaded`).
- Default center is hardcoded to Rotterdam (51.906688, 4.48837).

**3. What part of behavior can be documented from this file?**
- Google Maps uses `AdvancedMarker` (not the deprecated `Marker`). The `mapId` prop is required for `AdvancedMarker` to function — this is a behavioral constraint introduced in v4.0.0.
- When a marker has a custom image URL, it renders as `<img src={marker.url} />` inside the `AdvancedMarker`; otherwise a `<Pin />` icon is used.
- The `InfoWindow` title text has a pointer cursor only if an `onClick` action is also set.
- Map interaction controls: `gestureHandling` is set to "auto" when `draggable` is true, "none" when false. `scrollwheel`, `zoomControl`, `streetViewControl`, `mapTypeControl`, `fullscreenControl`, `rotateControl` are passed through directly.
- Zoom bounds: `minZoom=1`, `maxZoom=20`.

**4. Is it user-facing?**
Yes — this produces the visible Google Maps canvas and all markers rendered on it.

**5. What new did you learn from this file?**
The `mapId` prop (`googleMapId` in container props) is a hard requirement for Google Maps v4+. Without it, `AdvancedMarker` cannot be rendered. This is a breaking change from v3 where the legacy `Marker` worked without a map ID.

---

## src/components/LeafletMap.tsx

**1. What is the purpose of this file?**
Implements the Leaflet-based map rendering for OpenStreetMap, Mapbox, and HERE Maps providers. Uses `react-leaflet` (`MapContainer`, `Marker`, `Popup`, `TileLayer`). A helper `SetBoundsComponent` handles auto-zoom by calling `map.flyToBounds()` or `map.panTo()`.

**2. What kind of logic is described in this file?**
- Custom icon workaround: due to a long-standing `react-leaflet` issue (#453) with default icon URLs, the component always sets the `icon` prop explicitly on each marker — using a constructed `LeafletIcon` with bundled image paths for the default, or a `DivIcon` wrapping an `<img>` for custom markers.
- Custom marker CSS: `DivIcon` sets zero height/width on the container element (`.custom-leaflet-map-icon-marker`) so the hitbox matches only the icon image, avoiding misaligned click areas.
- Auto-zoom: `flyToBounds` with `padding: [0.5, 0.5]` and `animate: false` when `autoZoom` is true; `panTo` to bounds center otherwise.
- Marker interactivity: `interactive` is true only if the marker has a title OR an onClick action. If only an onClick (no title), the click event fires directly; if a title exists, a `Popup` is shown on click.

**3. What part of behavior can be documented from this file?**
- The `attributionControl` prop (visible only for non-Google providers) controls whether map credits are shown.
- Leaflet map zoom bounds: `minZoom=1`, `maxZoom=18` (vs Google's maxZoom=20).
- A marker is non-interactive (cannot be clicked or focused) if it has neither a title nor an onClick action.
- Popup title text has pointer cursor only when an onClick is also set.
- `optionDrag` maps to Leaflet's `dragging`; `optionScroll` to `scrollWheelZoom`.
- Default center: same Rotterdam coordinates as Google Maps implementation.

**4. Is it user-facing?**
Yes — produces the visible Leaflet map canvas for OpenStreetMap, Mapbox, and HERE Maps.

**5. What new did you learn from this file?**
The custom icon workaround (always setting explicit `icon` prop) is a permanent architectural decision documented in a code comment referencing react-leaflet issue #453. This means the widget will not automatically benefit from any future react-leaflet fix to default icon handling.

---

## src/utils/geodecode.ts

**1. What is the purpose of this file?**
Provides address-to-lat/lng geocoding and the `useLocationResolver` hook. The hook converts raw `MarkersType[]` and `DynamicMarkersType[]` into resolved `Marker[]` by separating address-based markers (need geocoding) from lat/lng markers (passed through directly).

**2. What kind of logic is described in this file?**
- `convertAddressToLatLng`: splits markers into address-based vs lat/lng-based; address-based markers are geocoded via Google Geocoding API (`https://maps.googleapis.com/maps/api/geocode/json`). An API key is required if any address markers are present; throws an error without one.
- `geocode`: caches results in `window.mxGMLocationCache` — a global cache keyed by address string. Prevents duplicate API calls for the same address within a session.
- `useLocationResolver`: a React hook that memoizes marker data and triggers re-geocoding only when markers actually change (deep equality check via `deep-equal`, excluding `action` callbacks from comparison).
- Failed geocoding for individual markers is caught and logged, not propagated — the marker is simply omitted from the result.

**3. What part of behavior can be documented from this file?**
- Geocoding results are cached globally in `window.mxGMLocationCache` — the same address is never geocoded twice in the same browser session.
- If an address cannot be resolved, the marker is silently dropped from the map (console.error only).
- A Geocoding API key (`geodecodeApiKey`/`geodecodeApiKeyExp`) is required when any marker uses address-based location. An error is thrown at geocoding time (not at config validation time) if the key is absent.
- Marker equality is checked without action callbacks — changes to onClick actions alone do not trigger re-geocoding.

**4. Is it user-facing?**
Indirectly — this drives whether markers appear on the map. Address geocoding failures are not surfaced to the end user.

**5. What new did you learn from this file?**
The geocoding always calls the Google Geocoding API regardless of which map provider is selected. Even for an OpenStreetMap or HERE Maps widget, if address-based markers are used, a Google Geocoding API key is required. This is a cross-provider behavioral constraint not immediately obvious from the prop names.

---

## src/utils/data.ts

**1. What is the purpose of this file?**
Adapts raw Mendix widget prop types (`MarkersType`, `DynamicMarkersType`) to the internal `ModeledMarker` interface. Two functions: `convertStaticModeledMarker` for static markers (text template values), `convertDynamicModeledMarker` for dynamic markers (data source items).

**2. What kind of logic is described in this file?**
- `convertStaticModeledMarker`: reads `.value` from each dynamic value field; parses lat/lng strings via `parseNumber`.
- `convertDynamicModeledMarker`: returns empty array if data source is not yet `Available`; iterates items via `markersDS.items`.
- `parseNumber`: replaces comma with dot before parsing as float — handles locale-specific decimal separators (e.g., "51,906" → 51.906).
- For dynamic markers: `customMarkerDynamic` is shared across all items in the list (single image for the whole list, not per-item). `id` is set from the Mendix object `item.id`.

**3. What part of behavior can be documented from this file?**
- Latitude and longitude values for static markers are accepted as strings and comma-decimal notation is supported (e.g., "4,48837" is accepted and converted to 4.48837). This was a bug fix in v4.0.0.
- Dynamic markers only render when the data source status is `Available`; loading or unavailable data sources result in an empty marker list.
- Custom marker image for dynamic markers is a single widget-level image (not per-item from the data source).
- Dynamic marker title is per-item from a string attribute; if title attribute is not mapped, title defaults to empty string `""`.

**4. Is it user-facing?**
No — pure data transformation layer.

**5. What new did you learn from this file?**
Custom marker images in dynamic marker lists are shared across all items (the `customMarkerDynamic` property is on the `DynamicMarkersType` object, not on each data source item). All items in one dynamic marker list will display the same custom icon.

---

## src/utils/location.ts

**1. What is the purpose of this file?**
Provides `getCurrentUserLocation()`, which wraps the browser Geolocation API to obtain the user's current position as a `Marker` object. The current location marker uses a hardcoded base64-encoded PNG image as its icon.

**2. What kind of logic is described in this file?**
Wraps `navigator.geolocation.getCurrentPosition()` in a Promise. Rejects with `"Current user location is not available"` in two cases: geolocation API not supported, or the user denies permission. The returned `Marker` has the blue location-pin icon as a data URI.

**3. What part of behavior can be documented from this file?**
- `showCurrentLocation` requires browser geolocation support. If the browser does not support it or the user denies the permission prompt, the current location marker silently fails (promise rejection is caught in Maps.tsx with console.error).
- The current location marker uses a fixed, built-in icon (a blue pin PNG embedded as a base64 data URI) — it is not configurable.
- There is no timeout configured for geolocation; the browser default applies.

**4. Is it user-facing?**
Yes — the current location marker appears on the map when enabled and the browser grants permission.

**5. What new did you learn from this file?**
The current location marker icon is a hardcoded base64 PNG baked into the source, not a configurable or theme-able asset. Its visual appearance cannot be changed by widget configuration.

---

## src/utils/zoom.ts

**1. What is the purpose of this file?**
Translates the string zoom enum values to numeric Leaflet/Google Maps zoom levels.

**2. What kind of logic is described in this file?**
A switch statement mapping: world→1, continent→5, city→10, street→15, buildings→20. Returns 1 (world) as the default fallback for unrecognized values (which in practice includes "automatic", handled at the call site before calling this function).

**3. What part of behavior can be documented from this file?**
The named zoom levels correspond to these numeric levels:
- world = 1
- continent = 5
- city = 10 (also used as the default when autoZoom is true)
- street = 15
- buildings = 20

When `autoZoom` is true, both Google Maps and LeafletMap use `city` (10) as the starting zoom before `fitBounds`/`flyToBounds` adjusts it.

**4. Is it user-facing?**
Indirectly — the translated numeric values control the zoom level displayed on the map.

**5. What new did you learn from this file?**
The "automatic" zoom enum value is never passed to `translateZoom` — it is intercepted upstream (`props.zoom === "automatic"` → `autoZoom=true`) and the zoom logic is handled by bounds-fitting instead.

---

## src/utils/leaflet.ts

**1. What is the purpose of this file?**
Builds the Leaflet `TileLayerProps` object for the selected non-Google map provider. Handles provider-specific URL templates, attribution strings, and API key injection.

**2. What kind of logic is described in this file?**
- Mapbox: URL uses `{id}` parameter set to `"mapbox/streets-v11"`, with access token appended as query param (`?access_token=`). Tile size is 512 with `zoomOffset: -1` to correct the zoom level mismatch.
- HERE Maps: Supports two token formats — legacy `"app_id,app_code"` (comma-separated, split into `?app_id=&app_code=`) and modern single `apiKey` format. URL targets the `normal.day` tile style.
- OpenStreetMap: No API key required. Uses the standard osm.org tile URL.

**3. What part of behavior can be documented from this file?**
- Mapbox always uses the `streets-v11` style — no style customization is supported via props.
- HERE Maps API key supports two formats: legacy comma-delimited `"app_id,app_code"` or modern single `apiKey`. The widget detects which format is used by checking for a comma.
- OpenStreetMap requires no API key and has no token injection.
- Attribution strings are provider-specific and are hardcoded.

**4. Is it user-facing?**
No — tile layer configuration is internal plumbing. The attribution text that appears on the map is a side effect of this file.

**5. What new did you learn from this file?**
HERE Maps has a legacy authentication mode (app_id + app_code, comma-separated in the API key field) in addition to the modern apiKey format. Both are supported and detected dynamically. The widget checks for a comma in the token string to distinguish them.

---

## src/Maps.editorConfig.ts

**1. What is the purpose of this file?**
Provides three Studio/Studio Pro editor hooks: `getProperties` (controls prop visibility), `check` (validates prop values and returns errors), and `getPreview` (returns an SVG preview image for design mode).

**2. What kind of logic is described in this file?**
- `getProperties`: Hides mutually exclusive props (e.g., shows `apiKey` OR `apiKeyExp` but not both, based on which is filled). On web platform without advanced mode, hides `mapProvider`, `markerStyle`, `customMarker`. For non-Google providers, hides Google-specific controls (streetView, mapTypeControl, fullScreenControl, rotateControl, googleMapId). For OpenStreet specifically, also hides API key fields. For Google Maps, hides `attributionControl`. Hides geodecode key fields when no address markers are configured.
- `check`: Validates that non-OpenStreetMap providers have an API key, that address markers have a geodecode key, that each static marker has either address or lat/lng, and that custom-image markers have an image set.
- `getPreview`: Returns an SVG image corresponding to the selected provider (GoogleMaps.svg, Mapbox.svg, OpenStreetMap.svg, HereMaps.svg).

**3. What part of behavior can be documented from this file?**
- OpenStreetMap is the only provider that does not require an API key — all others (Google Maps, Mapbox, HERE Maps) require one, enforced by a validation error.
- The geodecode API key fields are only visible when at least one marker (static or dynamic) uses address-based location — they are hidden otherwise.
- In Studio (web platform), map provider selection is hidden unless "Enable advanced options" is turned on. In Studio Pro (desktop), all options are always visible and the `advanced` toggle itself is hidden.
- Attribution control prop is hidden for Google Maps (Google manages its own attribution); it is only visible for Leaflet-based providers.
- Custom marker image prop is only visible when marker style is "image".

**4. Is it user-facing?**
Yes — in the sense that developers configuring the widget in Studio/Studio Pro see the property panel shaped by this file. Validation errors from `check` appear as warnings in the editor.

**5. What new did you learn from this file?**
The `geodecodeApiKeyExp` and `geodecodeApiKey` props are dynamically hidden in the Studio editor when no address markers are configured. This means a developer only sees the geocoding key field when it is actually needed, reducing confusion.

---

## src/Maps.editorPreview.tsx

**1. What is the purpose of this file?**
Provides the React component rendered in Studio's design-mode canvas (`preview` function) and the CSS for the preview container (`getPreviewCss`).

**2. What kind of logic is described in this file?**
Renders a simple `<div className="widget-maps-preview" />`. When the provider is Mapbox or HERE Maps, prepends an Alert warning that preview is not possible without an API key. The preview CSS comes from `MapsPreview.scss`.

**3. What part of behavior can be documented from this file?**
- In Studio design mode, Mapbox and HERE Maps show a warning message: "Provider unavailable without API Key, preview is not possible at the moment." The actual map is not rendered in design mode for these providers.
- OpenStreetMap and Google Maps do not show the warning (Google Maps preview uses the SVG from `getPreview` in editorConfig; OpenStreet presumably renders a placeholder).

**4. Is it user-facing?**
Yes — visible to the developer in Studio design mode canvas.

**5. What new did you learn from this file?**
Design-mode preview is intentionally simplified — no live map tiles are loaded. Mapbox and HERE explicitly show a warning. The `getPreview` hook in editorConfig (returning SVGs) is used in "structure preview" mode distinct from the live React-based design mode preview here.

---

## src/ui/Maps.scss

**1. What is the purpose of this file?**
Defines CSS for both Google Maps and Leaflet map containers, including wrapper sizing, z-index layering, and the custom marker icon hack for Leaflet.

**2. What kind of logic is described in this file?**
- `.widget-maps`: `position: relative` — serves as the positioning context for both implementations.
- `.widget-google-maps-wrapper` and `.widget-leaflet-maps-wrapper`: `100% width/height` to fill the parent container.
- Both map divs are positioned `absolute` with `top:0, bottom:0, left:0, right:0` to fill the wrapper.
- `.custom-leaflet-map-icon-marker`: `height: 0px; width: 0px` — collapses the DivIcon container so the hitbox matches only the image, not the phantom container dimensions.
- `.custom-leaflet-map-icon-marker-icon`: `transform: translate(-50%, -50%)` — centers the image on the marker coordinate.

**3. What part of behavior can be documented from this file?**
The custom Leaflet marker icon is centered via CSS transform (-50%, -50%), not via Leaflet's iconAnchor property. The DivIcon container has its size zeroed out to prevent click area mismatches. This is a workaround for a Leaflet default behavior where the DivIcon container's hitbox includes padding.

**4. Is it user-facing?**
Yes — these styles directly affect the visual appearance and interaction behavior of the map.

**5. What new did you learn from this file?**
The marker centering for custom Leaflet icons is achieved entirely through CSS (`translate(-50%, -50%)`) rather than Leaflet's built-in `iconAnchor` parameter. The `!important` overrides on height, width, and margin reset Leaflet's own inline styles that would otherwise interfere.

---

## src/components/__tests__/GoogleMap.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `GoogleMapContainer` component using Jest and Testing Library. Tests structural rendering with different dimension unit combinations and with markers/current-location active.

**2. What kind of logic is described in this file?**
Snapshot tests for five dimension configurations: (percentageOfWidth + pixels), (pixels + pixels), (percentageOfWidth + percentage), (percentageOfParent + percentage), plus two behavioral snapshots: rendering with markers and rendering with current location. Uses `@googlemaps/jest-mocks` to mock the Google Maps API.

**3. What part of behavior can be documented from this file?**
- All three `heightUnit` values (`percentageOfWidth`, `pixels`, `percentageOfParent`) combined with both `widthUnit` values are unit-tested — these dimension combinations are confirmed to render valid structure.
- `showCurrentLocation: true` with a `currentLocation` object is a supported and tested state.
- The widget renders correctly with multiple simultaneous markers (`locations` array with two entries).
- The test default uses `mapId: "DEMO_MAP_ID"` — confirming mapId is always required for test setup.

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The Google Maps mock (`@googlemaps/jest-mocks`) is initialized before each test (`beforeEach(() => { initialize(); })`), confirming the widget requires the Google Maps JS API to be loaded to function. The mock replaces `google.maps.*` in the test environment.

---

## src/components/__tests__/LeafletMap.spec.tsx

**1. What is the purpose of this file?**
Unit tests for the `LeafletMap` component. Tests structural rendering with different dimension units, all three non-Google providers, attribution control, markers, and current location.

**2. What kind of logic is described in this file?**
Snapshot tests covering dimension units (same four combinations as Google Maps tests), provider variants (`hereMaps`, `mapBox` — OpenStreet is the default), `attributionControl: true`, rendering with two markers, and rendering with current location.

**3. What part of behavior can be documented from this file?**
- All three Leaflet-based providers (openStreet, hereMaps, mapBox) are unit-tested and confirmed to produce valid render output.
- `attributionControl: true` is a distinct rendering state — the attribution display is confirmed to change structure.
- Current location and markers can coexist (current location marker is added to the `locations` array rendering path).

**4. Is it user-facing?**
No — test file only.

**5. What new did you learn from this file?**
The `mapProvider` prop on `LeafletMap` affects the `TileLayer` URL (via `baseMapLayer`), which is reflected in the snapshot — confirming that different providers produce different tile configurations in the render output.

---

## e2e/google.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for Google Maps rendering. Tests screenshot baseline, DOM rendering presence, and marker count for three scenarios: default (mixed), static markers only, and datasource markers only.

**2. What kind of logic is described in this file?**
Three test groups (default, static, datasource). Each confirms the `.widget-google-maps` element is visible. Marker count is verified by counting `.GMAMP-maps-pin-view` elements — 3 in default, 1 in static, 2 in datasource. A screenshot baseline test verifies overall visual correctness.

**3. What part of behavior can be documented from this file?**
- Google Maps markers are rendered with the CSS class `.GMAMP-maps-pin-view` (from `@vis.gl/react-google-maps` AdvancedMarker implementation).
- The default test page shows 3 markers (mixed static + dynamic).
- Static markers page (`/p/google-static`) shows 1 marker.
- Datasource markers page (`/p/google-datasource`) shows 2 markers.
- Session logout after each test is required due to Mendix's 5-session license limit.

**4. Is it user-facing?**
The tested behaviors are user-facing (map renders, markers appear). The test file itself is not.

**5. What new did you learn from this file?**
The Google Maps AdvancedMarker element uses the CSS class `GMAMP-maps-pin-view` for its DOM representation. This is the selector used to count/locate markers in e2e tests and may be relevant for CSS customization.

---

## e2e/openstreet.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for OpenStreetMap rendering. Tests rendering, marker count (mixed/static/datasource), screenshot baseline, and marker click behavior.

**2. What kind of logic is described in this file?**
Four test groups: rendering (screenshot baseline), mixed rendering (3 markers), static locations (1 marker), datasource locations (2 markers), on-click. The on-click test clicks the first `.leaflet-marker-icon` twice, then verifies a modal dialog appears containing "Clicked on static marker".

**3. What part of behavior can be documented from this file?**
- OpenStreetMap markers use CSS class `.leaflet-marker-icon`.
- Clicking a marker fires the configured onClick action. In the e2e test, this opens a Mendix modal dialog — confirming the action type "open modal popup" is supported and e2e-verified.
- The click test requires two clicks (first opens the Leaflet popup, second triggers the action from within the popup's span element). This is the behavioral interaction model for Leaflet markers with both title and onClick.
- Mixed page (`/p/osm`) shows 3 markers; static (`/p/osm-static`) shows 1; datasource (`/p/osm-datasource`) shows 2.

**4. Is it user-facing?**
The tested behaviors (map rendering, marker click → modal) are user-facing.

**5. What new did you learn from this file?**
For Leaflet markers with a title, the onClick action fires from within the Popup — so clicking the marker first opens the popup, then clicking the title text inside the popup fires the action. This is a two-click interaction, not a single-click interaction.

---

## e2e/here.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for HERE Maps rendering. Tests screenshot baseline, rendering, and marker count (mixed/static/datasource), and marker click behavior.

**2. What kind of logic is described in this file?**
Four test groups mirroring the OpenStreetMap tests. Several tests include `test.skip(browserName === "firefox", ...)` for marker count checks — indicating HERE Maps marker loading is unreliable in Firefox in the test environment. The on-click test verifies that clicking the first marker twice triggers a modal with "Clicked on static marker".

**3. What part of behavior can be documented from this file?**
- HERE Maps marker count tests are skipped for Firefox — a known compatibility constraint in the e2e environment (not necessarily a production limitation).
- Mixed page (`/p/here`) shows 3 markers; static (`/p/here-static`) shows 1; datasource (`/p/here-datasource`) shows 2.
- The onclick behavior (two-click pattern, modal confirmation) is identical to OpenStreetMap — both use the Leaflet rendering path.

**4. Is it user-facing?**
The tested behaviors are user-facing.

**5. What new did you learn from this file?**
HERE Maps e2e tests skip marker count assertions in Firefox. This is a test-environment limitation, not a widget limitation, but suggests HERE Maps tile loading may be slower or behave differently in Firefox.

---

## e2e/mapbox.spec.js

**1. What is the purpose of this file?**
End-to-end Playwright tests for Mapbox rendering. Structure mirrors HERE Maps and OpenStreetMap tests: rendering, marker count (mixed/static/datasource), screenshot baseline, and on-click.

**2. What kind of logic is described in this file?**
Four test groups. Marker count tests for static and datasource are skipped for Firefox (`test.skip(browserName === "firefox", ...)`). On-click test uses `page.getByRole("button", { name: "Marker" })` to locate the marker (role=button with accessible name "Marker"), clicks it twice, and verifies modal body contains "Clicked on static marker".

**3. What part of behavior can be documented from this file?**
- Mapbox markers have ARIA role "button" with accessible name "Marker" — confirming Leaflet markers are keyboard-accessible when interactive.
- Mixed page (`/p/mapbox`) shows 3 markers; static (`/p/mapbox-static`) shows 1; datasource (`/p/mapbox-datasource`) shows 2.
- The two-click interaction pattern for titled Leaflet markers applies identically across OpenStreetMap, HERE Maps, and Mapbox.
- Marker count assertions for Mapbox static/datasource are skipped on Firefox (same limitation as HERE Maps).

**4. Is it user-facing?**
The tested behaviors are user-facing.

**5. What new did you learn from this file?**
Interactive Leaflet markers (those with `interactive=true`) have role="button" with accessible name "Marker" — this confirms Leaflet provides built-in keyboard accessibility for interactive map markers.

---

## CHANGELOG.md

**1. What is the purpose of this file?**
Documents all notable changes to the maps-web widget across versions, following Keep a Changelog format with semantic versioning.

**2. What kind of logic is described in this file?**
Version history from v2.0.1 to v4.1.0 (plus reference to older releases on the marketplace).

**3. What part of behavior can be documented from this file?**
Key behavioral changes per version:
- v4.1.0 (2025-10-29): Fixed rendering failures in some cases. Fixed array-index-based list rendering bug (could cause issues when filtering large location lists). Fixed console warning for markers with titles in LeafletMap.
- v4.0.0 (2024-05-02): Fixed comma-decimal in lat/lng values. Replaced deprecated Google Maps `Marker` with `AdvancedMarker`. Removed `mapsStyles` property. Added required `mapId` property for Google Maps.
- v3.2.1 (2023-08-11): Fixed map centering after zoom-to-marker.
- v3.1.1 (2022-01-31): Fixed OpenStreetMap styles causing "puzzled screen" display.
- v3.0.1 (2021-10-13): Made "Enable advanced options" only for Studio (not Studio Pro). Renamed "Google maps API Key" to "Geo Location API key". Made Geo Location API key required for address markers.
- v3.0.3 (2021-11-18): Design mode now renders SVG instead of live map tiles.
- v2.0.3 (2021-08-04): Renamed "Markers" to "Static markers" and "Marker list" to "Dynamic markers".
- v2.0.1 (2021-04-26): Added structure preview per provider. Custom marker icon anchor changed to center instead of top-left.

**4. Is it user-facing?**
Yes — version history is publicly visible on the Mendix Marketplace and in the repo.

**5. What new did you learn from this file?**
The v4.0.0 upgrade is a breaking change for Google Maps users: `mapsStyles` was removed, `AdvancedMarker` replaced the deprecated `Marker`, and `mapId` became required. Existing configurations using the old deprecated Marker would need to add a Google `mapId` after upgrading.
