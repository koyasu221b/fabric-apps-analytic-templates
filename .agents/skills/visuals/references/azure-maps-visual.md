# Azure Maps Visual

Use `azure-maps-control`, the Azure Maps Web SDK, to build local geographic maps from a user-supplied Fabric semantic model. Unlike `VegaVisual` and `DataGrid`, the SDK manages a map inside a DOM container; it is not a React component and does not accept a `DataTable` prop or Vega specification.

**Scope:** a local app with a server-side subscription-key proxy. No Maps user sign-in, app registration, or deployment is required. Fabric model access still requires an authorized identity. Preserve existing charts and grids.

This reference describes what the agent should generate. Do not assume the template already contains a map component, local backend, or `dev:local` command. Planning requests must not trigger implementation or resource creation.

## Package and imports

Add `azure-maps-control` to the generated app's dependencies if missing, keeping the package manifest and lockfile consistent. Do not add a separate sign-in library for this workflow.

```ts
import * as atlas from "azure-maps-control";
```

Load the SDK stylesheet once in the app's global CSS:

```css
@import "azure-maps-control/dist/atlas.min.css";
```

Use a mounted container with nonzero width and an explicit height, for example `style={{ width: "100%", height: 448 }}`. WebGL support is required. The examples below share one setup scope; they are SDK patterns, not a complete React component or backend. For API details, consult the installed package's public types or the [Azure Maps SDK reference](https://learn.microsoft.com/en-us/javascript/api/azure-maps-control/).

## Data format

The basemap and the business-data overlay have separate sources:

```text
Azure Maps -> local authenticated proxy -> SDK basemap (roads, labels, tiles)
Fabric model -> local query backend -> validated GeoJSON -> SDK overlay layer
```

The semantic model stays in Fabric; the local app retrieves query results, not a copy of the model. To display model data, use GeoJSON features in an `atlas.source.DataSource`, rather than `DataTable`.

Before implementing the visual:

1. Confirm the workspace/model IDs and registered connection alias. Follow [schema-discovery](../../schema-discovery/SKILL.md) to discover relevant fields, measures, and relationships using `fabric-app-data query`.
2. Confirm the requested geographic grain, layer, measures, filters, labels, and interactions. Coordinates support points; lines/polygons require suitable geometry. Names alone require approved geocoding or another location source. Never silently add a source or substitute points for requested polygons.
3. Execute the actual DAX and verify its exact output names/types. Keep DAX in `.dax` files, query factories in `.ts`, and result mapping in a typed adapter. Reuse model measures; do not compute map aggregates from a truncated detail table.
4. Validate coordinates as finite WGS84 longitude/latitude values, using **`[longitude, latitude]`** order. Require longitude in `[-180, 180]` and latitude in `[-90, 90]`, stable unique IDs, and valid measure values. Report exclusions; never coerce missing coordinates to zero. Preserve valid zero values.

This example shape is the adapter's output, not an assumed model schema:

```ts
type MapPoint = {
    id: number | string;
    longitude: number;
    latitude: number;
    label: string;
    formattedValue: string;
    color: string;
};
```

Resolve `color` from the requested measure/categories and the app's palette. Format `formattedValue` using the same units and precision as the accompanying table. Do not hardcode a particular dataset's fields, legend, or geographic region.

## Local authentication and backend

Have the user enter an authorized Azure Maps key in an ignored file:

```dotenv
# .env.local-preview.local
AZURE_MAPS_SUBSCRIPTION_KEY=<enter-the-key-locally>
```

Never request the key in chat or use a `VITE_` prefix. Environment files do not hide values that are bundled into browser code. Add `.env.*.local` to `.gitignore` and publish only blank placeholders in an environment example. Avoid `.env.local` when existing template commands regenerate it.

Generate these development-only surfaces:

| Surface | Required behavior |
| --- | --- |
| Start command | Define `dev:local` as `vite --mode local-preview --host localhost --port 5173 --strictPort`, without provisioning or Rayfin environment-generation hooks. |
| Vite backend | Enable only for `command === "serve"` and `mode === "local-preview"`. Read the key with `loadEnv(mode, root, "AZURE_MAPS_")`; never expose it through client configuration or `define`. |
| Local bootstrap | Gate with `import.meta.env.DEV && import.meta.env.MODE === "local-preview"`. Use the local data transport and avoid requiring Fabric-host bootstrap in this mode only. Preserve normal template behavior elsewhere. |
| Status endpoint | Return only whether a key is configured. Missing configuration must show an actionable message, without disabling the data table. |
| Maps endpoint | Accept GET requests for allowlisted Azure Maps rendering resources. Add the key in the upstream `subscription-key` header and return resource bytes/content type. Never return the key to the browser. |
| Query endpoint | Accept named query operations and validated filters; construct DAX server-side from the approved factories for the supplied model. Do not accept arbitrary DAX, model IDs, or shell commands from the browser. |

**Backend safeguards:**

- Bind to loopback only; validate Host and Origin, reject cross-site requests, disable cross-origin access, and require a per-run random request token on API calls. That browser-visible token is a local request guard, not the Maps key.
- Restrict upstream origins to `https://atlas.microsoft.com` and, if needed, the exact `https://us.atlas.microsoft.com` and `https://eu.atlas.microsoft.com` origins. Allow required `/map/` and `/styling/` paths, not arbitrary destinations.
- Reject embedded credentials, strip client-supplied authentication, disable upstream redirects, set timeouts and request limits, and do not forward arbitrary client headers. Return sanitized errors, not upstream error bodies or secret-bearing process details. Do not log keys or echo them in responses.
- Keep model data transient and preview responses uncached. Do not expose the server through a tunnel or network interface. These guards do not isolate the key from other software running under the user's account.

Run real model queries through the installed Fabric CLI or an existing approved server-side query helper, using the user's Azure CLI sign-in (`az login` if needed). Invoke the CLI without a shell and validate its JSON status even when the process exits successfully. Surface authentication/query failures explicitly. Preserve empty results and reject detected truncation or a reached output limit.

Document `npm run dev:local`, direct navigation to **http://localhost:5173**, restart after key changes, and Ctrl+C to stop. This mode needs neither a Fabric portal iframe nor deployment. Maps requests may incur account usage charges.

## Basemap

The SDK automatically fetches styles, sprites, glyphs, and tiles. Do not download a separate basemap file or manually draw tiles.

Before running this example, provide `container` (an `HTMLDivElement` already mounted in the page), `initialCenter` (a `[number, number]` pair in longitude/latitude order), `initialZoom` (a numeric starting zoom level), and an application-defined `reportError(message)` function that displays errors in the UI. The generated development backend must supply `previewRequestToken` and implement `/api/local-preview/maps`; neither the backend nor this endpoint is built into the template.

```ts
function localMapsRequest(value: string) {
    const url = new URL(value);
    const origins = [
        "https://atlas.microsoft.com",
        "https://us.atlas.microsoft.com",
        "https://eu.atlas.microsoft.com",
    ];
    if (!origins.includes(url.origin)) {
        throw new Error("Only Azure Maps resources are allowed.");
    }
    url.searchParams.delete("subscription-key");
    return {
        url: `${window.location.origin}/api/local-preview/maps?url=${encodeURIComponent(url.href)}`,
        headers: { "x-local-preview-token": previewRequestToken },
    };
}

const map = new atlas.Map(container, {
    center: initialCenter,
    zoom: initialZoom,
    style: "road",
    disableTelemetry: true,
    enableAccessibilityLocationFallback: false,
    transformRequest: localMapsRequest,
    authOptions: {
        authType: atlas.AuthenticationType.subscriptionKey,
        subscriptionKey: "local-preview-proxy", // Public routing marker only.
    },
});
map.events.add("error", () => {
    reportError("Azure Maps could not load. Check the local backend, key, and network.");
});
```

Start the map after the status endpoint confirms configuration. The public marker is not a credential: the proxy strips it and supplies the real key upstream. The backend must independently enforce its allowlist. Disabling accessibility location fallback prevents additional geocoding requests; retain SDK accessibility and an accessible table.

## Data source and layer

Add sources and layers after `ready`. For point maps, use a `BubbleLayer`; select an appropriate layer for other requested geometry. Keep the latest validated data if a query finishes before the map is ready.

```ts
const source = new atlas.source.DataSource();
const popup = new atlas.Popup();
let ready = false;
let disposed = false;
let latestPoints: MapPoint[] = [];

function updatePoints(points: MapPoint[], fitCamera = false) {
    if (disposed) return;
    latestPoints = points;
    if (!ready) return;
    try {
        popup.close();
        const features = points.map(point => new atlas.data.Feature(
            new atlas.data.Point([point.longitude, point.latitude]),
            { label: point.label, value: point.formattedValue, color: point.color },
            point.id,
        ));
        source.setShapes(features);
        if (fitCamera && features.length) {
            map.setCamera({
                bounds: atlas.data.BoundingBox.fromData(features),
                padding: 45,
                maxZoom: 10,
            });
        }
    } catch {
        reportError("Map data could not be displayed.");
    }
}

map.events.add("ready", () => {
    if (disposed) return;
    try {
        map.sources.add(source);
        const layer = new atlas.layer.BubbleLayer(source, "model-points", {
            radius: 8,
            color: ["get", "color"],
            strokeWidth: 2,
        });
        map.layers.add(layer);
        map.events.add("click", layer, event => {
            const shape = event.shapes?.[0];
            if (!shape) return;
            const properties = shape instanceof atlas.Shape
                ? shape.getProperties() : shape.properties;
            if (!properties) return;
            const content = document.createElement("div");
            content.textContent = `${properties.label}: ${properties.value}`;
            popup.setOptions({ content, position: event.position });
            popup.open(map);
        });
        ready = true;
        updatePoints(latestPoints, true);
    } catch {
        reportError("The map data layer could not initialize.");
    }
});
```

Call `updatePoints(validatedPoints, true)` for successful filter changes when fitting the results is desired. Call `updatePoints([])` while loading, on query errors, or for empty results; show query errors separately. Ignore out-of-order responses so older queries cannot restore stale points.

## Interactivity, theme, and lifecycle

- **Selection:** layer clicks can return either `atlas.Shape` or GeoJSON `Feature`. Use `textContent`, not model-provided HTML, for popup content. Coordinate selections and filters through dashboard state. SDK layers do not expose VegaVisual/DataGrid's `onInteraction` prop.
- **Theme:** read the shared app theme and call `map.setStyle({ style: isDark ? "night" : "road" })`. Resolve overlay colors from shared tokens and update them without resetting the user's camera. Follow [formatting](formatting.md) for semantic colors and readable values; Vega's automatic theme and `columnMetadata` formatting do not apply to the SDK. Keep meaningful measure legends even for a single point layer.
- **Container:** use the same card/title conventions as other visuals. The SDK does not automatically provide [VisualContainer](visual-container.md) chrome. Do not promise Copy/image export for WebGL until it has been verified.
- **Accessibility:** preserve attribution, provide meaningful labels and an accessible table, and keep it usable when Maps fails.
- **React:** create the map once per mounted container, update its source on data changes, and call hooks unconditionally. If lazily importing the SDK, check for unmount before constructing it. Report initialization/WebGL errors and dispose partially initialized resources if setup fails.

Observe container resizing and release resources in the owning effect's cleanup:

```ts
const observer = new ResizeObserver(() => map.resize());
observer.observe(container);

function dispose() {
    disposed = true;
    observer.disconnect();
    popup.close();
    map.dispose();
}
```

## Validation

For generated application changes, run relevant unit tests, TypeScript checking, lint, and build. Run `npx tsc -b` separately: the template's build command uses `--noCheck`, so a successful build does not establish type correctness. Azure Maps does not require a Vega-Lite specification and is not validated by `validate:visual`; continue running that command for existing Vega charts as required by `AGENTS.md`.

After generating the local mode and backend, run `npm run dev:local` and validate directly at `http://localhost:5173` with real queries and Maps responses. These browser checks supplement unit tests and headless Vega chart validation; they do not depend on the removed `app-validation` skill.

| Check | Expected result |
| --- | --- |
| Basemap | Successful style/tile requests and visible geography; a canvas or `ready` event alone is insufficient. |
| Overlay | Expected source/rendered features for the chosen view; popup values agree with model results and the table. |
| Filters | Correct date bounds, escaped filter values, latest-response handling, and no stale points/popups. |
| Empty/invalid data | Clear overlay for zero results; report invalid records and schema errors. A successful empty CLI result may omit column metadata. |
| Failures | Actionable missing/invalid-key, network, proxy, WebGL, and Fabric query errors; no mock data or silent success. |
| UX | Theme, resize, navigation, accessible table, and preserved chart/grid behavior. |
| Local isolation | Key file ignored and blocked from HTTP download; no real key in browser requests, served JavaScript, logs, or production assets. Local backend/bootstrap disabled in production. |

Test proxy request restrictions and data/lifecycle edge cases with unit tests; do not mock authentication or model data to claim live browser success. Finally, try the skill in a fresh template checkout without the demo files. Do not claim template reusability solely because one hand-built demo works. If access or configuration blocks a check, state the blocker. Deployment and Maps user-sign-in validation are outside this local-only milestone.
