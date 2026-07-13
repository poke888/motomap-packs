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

## Data attribution

Routing data is processed from [OpenStreetMap](https://www.openstreetmap.org)
extracts (via [Geofabrik](https://download.geofabrik.de/)) — © OpenStreetMap
contributors, licensed under the
[Open Database License (ODbL)](https://opendatacommons.org/licenses/odbl/).
