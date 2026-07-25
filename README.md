# motomap-packs

Pre-built on-device routing packs for the MotoMap (Switchback) iOS app —
Valhalla routing-tile archives, one per region, published as release assets.

## How the app uses this

The app fetches [`catalog.json`](catalog.json) from this repo's `main`
branch (`https://raw.githubusercontent.com/poke888/motomap-packs/main/catalog.json`)
to discover available regions, then downloads a region's `url` (a release
asset) when a planned route needs its coverage.

## Format

`catalog.json` (formatVersion 1): `engineVersion`, `updatedAt`, and
`regions[]` of `{id, name, country, sizeBytes, bbox [w,s,e,n], version,
url}`. Each pack tar contains a Valhalla tile extract (`index.bin` +
`<level>/<dir>/<id>.gph`) — the exact artifact layout the app's
`RoutingPackStore` expects.

## Engine version pin — IMPORTANT

Packs are built with **Valhalla 3.5.1** (digest-pinned image) to match the
engine vendored in the app. The on-disk tile format is version-coupled: do
NOT publish packs built with a different Valhalla version without bumping
the app's vendored engine in the same release cycle. The catalog's
`engineVersion` field is a hard filter — the app ignores regions whose
engineVersion mismatches its own engine.

## Rebuilding / adding regions

The build pipeline lives in the `moto-routes` repo:
`scripts/build-region-pack.sh` + `scripts/build-packs-catalog.sh`, with the
full recipe in `docs/region-packs.md` (Geofabrik extract → digest-pinned
one-shot Valhalla container → tar → catalog → `gh release upload`).

## Merged multi-province packs (`maritimes`)

Valhalla can't stitch two separate packs at route time — a trip needs ONE
pack whose graph covers every point end to end. `maritimes` (NB + NS + PEI,
116.6 MB, built 2026-07-25) exists so a cross-province trip like
Fredericton → Halifax has a single covering pack to route through.

Built by feeding all three provinces' `.osm.pbf` extracts to
`valhalla_build_tiles` in one `docker run` (same digest-pinned 3.5.1 image,
same `build_elevation=True`/`build_tar=True` settings as every other pack) —
**not** an `osmium merge`. Valhalla's own build log warns
("Tile building using more than one osm.pbf extract is discouraged... See
valhalla/valhalla#3925") and reports several thousand "possible duplicate"
edges across the three levels; empirically this did NOT break connectivity
at the NB/NS border — a live smoke-test route from St. George, NB to
Halifax, NS (motorcycle costing) returned a real 484 km / 274 min route
through `Fort Lawrence Road` → `NS 104` → `Highway 102`, i.e. it genuinely
crosses the isthmus using both provinces' road data in one pack. If a
future rebuild of this region ever produces a broken/disconnected route
near a province border, merge the extracts with `osmium merge` first
(Valhalla's own recommendation) rather than trusting the multi-pbf path —
this repo's build only skipped that step because the smoke test passed.

**Known app-side gap (not fixed here, not this repo's code): the
catalog-region "smallest bbox wins" selection used for the CONTEXTUAL
pack-download offer can under-offer `maritimes`.** Geofabrik extract bboxes
are padded rectangles, not real coverage shapes — New Brunswick's bbox
rectangle happens to reach Halifax's longitude/latitude, so for a rider who
already has `nb` installed, the app's offer logic sees "the smallest
catalog region covering both trip endpoints is already installed" and
never prompts to download `maritimes`, even though `nb`'s actual road graph
has no Nova Scotia roads. The on-device ROUTER's own retry ladder is fine —
it tries covering packs smallest-bbox-first and falls through to the next
one on a genuine no-edges failure, so routing succeeds once `maritimes` (or
`ns`) is actually installed. Riders who already have `nb` currently need to
add `maritimes` manually from Settings → Offline Maps rather than being
offered it inline. This is a `PackOfferCoordinator`/`decide()` selection
question in the iOS app, out of scope for this repo.

## Data attribution

Routing data is processed from [OpenStreetMap](https://www.openstreetmap.org)
extracts (via [Geofabrik](https://download.geofabrik.de/)) — © OpenStreetMap
contributors, licensed under the
[Open Database License (ODbL)](https://opendatacommons.org/licenses/odbl/).
