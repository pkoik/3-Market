# 3 — Market · Rive v2 Integration Handoff

Handoff updated: August 17, 2026

Reference repository: https://github.com/pkoik/3-Market

## Deliverables

| File | Purpose |
| --- | --- |
| `index.html` | Standalone responsive preview with the complete market block plus two labeled Graph/Jacket source groups |
| `3-market_v02.riv` | Current main Rive v2 source used by the complete block and the first pair of isolated artboards |
| `3-marketgraphs.riv` | Independent Graph-only Rive source |
| `3-marketjacket.riv` | Independent Jacket-only Rive source |
| `3-Market_v01.riv` | Previous source version retained for reference |
| `HANDOFF.md` | This integration document |

`index.html` embeds all three current Rive files as base64, so no manual file picker is required when opening the demo directly. The separate `.riv` files are included for development.

## Runtime

The preview uses the pinned WebGL2 runtime:

```text
https://unpkg.com/@rive-app/webgl2@2.38.4
```

WebGL2 is required because the asset uses blur effects. There is no automatic Canvas renderer fallback.

Every Rive instance uses:

- `autoplay: true`;
- `autoBind: true`;
- `useOffscreenRenderer: false`;
- `Fit.Layout`;
- `State Machine 1`.

## Rive v2 Contract

Do not rename these entities without updating the integration code:

| Preview | Source file | Artboard | Size | View Model | Instance | Entrance trigger |
| --- | --- | --- | --- | --- | --- | --- |
| Complete block | `3-market_v02.riv` | `3 - Market` | `1136 × 536` | `MarketVM` | `Instance` | `isIn` |
| Graph from main file | `3-market_v02.riv` | `Graph` | `265 × 204` | `GraphVM` | `Instance` | `isStatsGraph` |
| Jacket from main file | `3-market_v02.riv` | `Jacket` | `573 × 573` | `JacketVM` | `Instance` | `isJacketIn` |
| Independent Graph | `3-marketgraphs.riv` | `Graph` | `265 × 204` | `GraphVM` | `Instance` | `isStatsGraph` |
| Independent Jacket | `3-marketjacket.riv` | `Jacket` | `619 × 665` | `JacketVM` | `Instance` | `isJacketIn` |

The complete block still contains the nested Graph and Jacket components. The first isolated pair instantiates those artboards directly from the main file. The second pair uses the two independent exported files.

## Page Structure

- `.page-placeholder` simulates the page content before the Rive section.
- `.market-section` contains the complete `3 - Market` artboard.
- `.components-section` contains two clearly labeled source groups: artboards from the main file, followed by the independent files.
- Each group contains only a Graph and Jacket, with no market supporting text.
- On desktop, the Graph and Jacket in each group are displayed side by side.
- At widths up to 900 px, they stack vertically and trigger independently as they enter the viewport.

Remove or replace `.page-placeholder` during product integration.

## Scroll Triggers

Each preview has its own `IntersectionObserver`. Its entrance trigger fires once when at least 18% of that preview frame is visible:

```js
const TRIGGER_THRESHOLD = 0.18;
```

| Frame | Trigger fired |
| --- | --- |
| `#main-frame` | `MarketVM.isIn` |
| `#graph-main-frame` | `GraphVM.isStatsGraph` from `3-market_v02.riv` |
| `#jacket-main-frame` | `JacketVM.isJacketIn` from `3-market_v02.riv` |
| `#graph-file-frame` | `GraphVM.isStatsGraph` from `3-marketgraphs.riv` |
| `#jacket-file-frame` | `JacketVM.isJacketIn` from `3-marketjacket.riv` |

A trigger only fires after the corresponding Rive instance has loaded and its View Model has been bound. Browsers without `IntersectionObserver` use passive `scroll` and `resize` listeners.

## Responsive Behavior

The complete block keeps the v2 source-artboard ratio of `1136 / 536`.

Its scale is calculated from the current frame width:

```js
scale = frame.clientWidth / 1136;
```

Both Graph previews are rendered at their native `265 × 204` scale inside `360 × 300` working canvases. The Jacket from the main file is rendered at 50% scale inside a `760 × 900` working canvas displayed at `380 × 450`. The independent Jacket is rendered at 50% scale inside an `820 × 1000` working canvas displayed at `410 × 500`. On narrower screens all working canvases scale down proportionally.

Horizontal page margins:

- desktop and tablet: 24 px;
- mobile: 16 px;
- minimum supported page width: 320 px.

Each frame is observed with `ResizeObserver`. After meaningful size or DPR changes, the integration calls `resizeDrawingSurfaceToCanvas()`. A separate DPR watcher keeps the canvases sharp on Retina and HiDPI displays.

## Overflow-Safe Rendering

The Jacket intentionally extends beyond the complete block. To preserve it:

- the visible main frame follows `1136 × 536`;
- the main canvas working area remains `1238 × 620`;
- `.market-section` and `.market-frame` use `overflow: visible`;
- the main artboard uses `TopLeft` alignment;
- `body` uses only `overflow-x: hidden`;
- do not add `overflow: hidden`, `clip-path`, or `contain: paint` to the main frame or its product container.

The isolated previews use the same overflow-safe technique:

- Graph: `360 × 300` working canvas, native artboard scale (`1:1` on desktop);
- Jacket from the main file: `760 × 900` working canvas, displayed at `0.5` scale on desktop;
- independent Jacket: `820 × 1000` working canvas, displayed at `0.5` scale on desktop;
- both Jackets start at a `120` source-pixel vertical offset, producing `60 px` of visible top reserve at the desktop scale;
- both use `TopLeft` alignment with `Fit.Layout`;
- their frames and parent stage keep `overflow: visible` and do not use paint containment.

The larger WebGL2 drawing surfaces are required because CSS `overflow: visible` cannot restore pixels that were already clipped at the canvas boundary.

## View Model Resolution

For every preview, use the instance automatically attached to the artboard when it matches the configured View Model. Otherwise:

1. resolve the View Model by name;
2. try the named `Instance`;
3. fall back to `defaultInstance()` or `instance()`;
4. bind it with `bindViewModelInstance()`;
5. resolve the configured trigger and fail visibly if it is missing.

The existing string properties on `MarketVM` remain the content API for the complete block.

## Live Text Controls

Append `?dev=1` to enable the collapsible `TEXT CONTROLS` panel inside the complete market block. It is closed by default and does not exist as an active editing UI in normal preview mode.

Each field writes directly to its bound `MarketVM` string property on the `input` event, so changes appear in Rive while typing. `RESET ALL` restores the values loaded from the Rive instance for the current page session.

The panel exposes these 13 properties:

```text
processedSalesAmount
processedSalesDescription
developerRevenueAmount
developerRevenueDescription
directSalesStudiosCount
directSalesStudiosDescription
steamFeePercentage
steamFeeDescription
directTransactionsDescription
gamesAt1MCount
gamesAt1MLabel
gamesAt10MCount
gamesAt10MLabel
```

Text edits are preview-only and are not written back into `3-market_v02.riv` or persisted after reload.

## Product Integration

1. Copy the required section markup and styles from `index.html`.
2. Load `3-market_v02.riv` as a static asset or reuse the embedded buffer approach.
3. Create separate Rive instances for any artboards that must be rendered independently.
4. Keep the Artboard, View Model, Instance, and trigger names from the contract table.
5. Keep the per-frame scroll observer instead of using one shared trigger.
6. Keep `ResizeObserver`, DPR watching, and `resizeDrawingSurfaceToCanvas()`.
7. Call `player.cleanup()` and disconnect observers/listeners when a component unmounts.

To load the external asset instead of the embedded buffer, use:

```js
src: "./3-market_v02.riv"
```

## Diagnostics

Append `?dev=1` to the URL to display the status of all five previews. Successful triggers appear as:

```text
main: MarketVM.isIn fired
graph-main: GraphVM.isStatsGraph fired
jacket-main: JacketVM.isJacketIn fired
graph-file: GraphVM.isStatsGraph fired
jacket-file: JacketVM.isJacketIn fired
```

The console logs one `[Xsolla Market Rive v2]` object per instance with its Artboard, State Machine, View Model, trigger, and renderer.

## Verified

- v2 embedded data loads without a manual file picker;
- all five previews create independent WebGL2 instances;
- `MarketVM.isIn` and all four component triggers fire from their own viewport observers;
- both isolated source groups contain only the Graph and Jacket visuals and are labeled with their source files;
- the complete block retains the intended Jacket overflow;
- both Graph labels and both complete isolated Jackets render without canvas-edge clipping;
- the Graphs use their native scale and the Jackets use a 50% desktop scale with dedicated top reserve;
- all 13 `MarketVM` string properties update live through the collapsible dev panel;
- `RESET ALL` restores the initial Rive text values, while normal mode keeps the panel hidden;
- desktop layout works at `1280 × 720`;
- mobile layout works at `390 × 844`, with Graph and Jacket triggers firing independently;
- canvas resizing and DPR handling are active;
- no runtime warnings or errors were observed.

## Acceptance Criteria

- the complete block preserves the `1136 × 536` v2 artboard ratio;
- the complete block entrance starts only after `#main-frame` reaches approximately 18% visibility;
- the isolated Graph and Jacket show without the surrounding market text;
- each isolated component fires only its own trigger and only once per page load;
- all canvases scale without horizontal scrolling;
- the main and isolated overflow is not clipped;
- blur rendering matches the Rive preview;
- the console contains no load, WebGL2, View Model, or trigger errors.
