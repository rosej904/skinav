# skinav

Voice-first ski navigation: "I'm at the bottom of Crest, how do I get to Hard Pan?"

## Layout

| Path | Role |
|---|---|
| `internal/ingest` | OpenSkiMap GeoJSON → routing graph (offline job) |
| `internal/graph` | Versioned graph artifact (what lands in S3) |
| `internal/engine` | Pure routing core: name matching, GPS origin, Dijkstra, voice summary |
| `cmd/skinav-ingest` | Build a resort graph + QA report |
| `cmd/skinav-route` | Local CLI for testing routes |
| `cmd/skinav-server` | MCP server (local and Lambda) |
| `internal/adapter/mcpserver` | MCP tools over stateless streamable HTTP |
| `internal/resorts` | Loads/caches one engine per resort (dir now, S3 later) |
| `internal/status` | Lift status providers: none, file, cached; status.live next |
| `resorts/<id>/bbox` | Overpass bounding box (south,west,north,east) |
| `resorts/<id>/aliases.json` | Hand-maintained name fixes, managed by PR |
| `resorts/<id>/pois.json` | Lodges/restrooms OSM is missing: `[{"name": "Milly Chalet", "kinds": ["lodge","food"], "lat": 40.58, "lon": -111.60}]`, or `{"name": "X", "remove": true}` |
| `resorts/<id>/features.json` | Feature tag fixes: `{"Deer Park": ["-terrain_park"], "Candyland": ["terrain_park"]}` |
| `resorts/<id>/links.json` | Hand-curated connections the map is missing, managed by PR |

Adapters (MCP now; REST or a phone line later) live in `internal/adapter/*`
and call `engine.Route`. Status providers and the group
store plug in behind interfaces; the engine itself never does I/O.

## Try it

```sh
make test
make fetch     # Overpass: lifts + downhill runs inside resorts/<id>/bbox
make graphs    # builds graphs/brighton.json, breckenridge.json, vail.json
go run ./cmd/skinav-route -graph graphs/brighton.json -from "Crest" -to "Great Western" -max blue
```

To add a resort, draw a box on openstreetmap.org (Export > "Manually select a
different area"), copy the four numbers into `resorts/<id>/bbox` as
`south,west,north,east`, and add `{}` to `resorts/<id>/aliases.json`.
The Breckenridge and Vail boxes are rough estimates; widen them if the QA
report is missing lifts.

You can also paste the output of `skinav-ingest -query <bbox>` into
overpass-turbo.eu to see exactly what gets pulled.

Overpass data has no elevation, so direction relies on the OSM convention
(lifts drawn bottom to top, pistes drawn downhill). The OpenSkiMap GeoJSON
path (`-lifts/-runs/-area`) is still supported if you get those files.

## MCP server

Requires Go 1.25+ (the official MCP Go SDK). After unzipping, run `go mod tidy` once.

```sh
make serve                              # http://localhost:8080/mcp
make serve STATUS=status.example.json   # pretend Crest is on wind hold
```

Tools: `find_route` (resort, from/at or lat/lon, to, max_difficulty, avoid),
`find_nearby` (closest restrooms, lodges, food, bars, terrain parks, glades,
moguls, groomers, ungroomed, or a color, ranked by travel time, with directions), `locate` (fuzzy name lookup),
`list_resorts`. Responses carry a terse `spoken`
line for voice plus full steps; server instructions tell the model to relay
`spoken` nearly verbatim.

It runs stateless streamable HTTP with JSON responses, so every request is
independent: no sessions, a good fit for Lambda.

Test locally with the MCP Inspector:

```sh
npx @modelcontextprotocol/inspector   # transport: Streamable HTTP, URL http://localhost:8080/mcp
```

Or test from your phone before any AWS work:

```sh
make serve HOSTS='*.trycloudflare.com'
cloudflared tunnel --url http://localhost:8080
```

and add the printed `https://<random>.trycloudflare.com/mcp` as a custom
connector in Claude (no auth). Without `HOSTS`, the SDK's DNS-rebinding
protection rejects tunneled requests with a 403.

Env vars (same binary on Lambda): `PORT`, `MCP_PATH` (use a long random
segment in prod), `ALLOWED_HOSTS`, `GRAPHS_DIR`, `STATUS_FILE`.

## Reading the QA report

* **dead-end nodes**: a run end or lift top with no way out usually means two
  features didn't snap. Look at the coordinates on openskimap.org, then tweak
  `-snap` or fix the data upstream in OSM (benefits everyone).
* Dead ends right at the edge of your bbox are usually runs that continue
  outside the box (e.g. into Solitude or Park City) and can be ignored.
* **missing connections**: if a route fails or detours oddly, add a link:
  `[{"from": "Crest Express@bottom", "to": "Great Western Express@bottom", "bidirectional": true, "note": "skate across base"}]`
* **unnamed**: unnamed run segments are read as "the connecting trail".
* Compare the lift list to the resort's status page and add aliases for
  mismatches, e.g. `{"Crest Express": "Crest"}`.

## Feature tags (parks, glades, moguls)

Runs get tags from OSM (`piste:type=snow_park`, `gladed=yes`,
`piste:grooming=mogul|backcountry|...`), from names ("Gary's Glade",
"Terrain Park"), and from park/glade polygons that runs pass through. The QA
report lists what got tagged per resort. OSM coverage is thin here, so expect to
curate `features.json`: add tags to runs you know (`"Pine Martin": ["glades"]`)
and remove wrong ones (`"Terrain Park": ["-moguls"]`). Requires re-running
`make fetch` (the query now includes `snow_park`).

## Glades and parks: confidence tiers

Tree skiing and parks are poorly tagged everywhere, so runs carry evidence
instead of a yes/no:

| Tier | From |
|---|---|
| confirmed | OSM tags, names ("Gary's Glade"), `features.json`, or `"confirmed"` in `vision.json` |
| likely / possible | trail-map reading (`vision.json`), runs inside mapped forest |

`find_nearby` answers with the closest match and hedges when it isn't sure:
"Desperado looks like a glade run, about 4 minutes away. The trail map draws it
through the trees. ... If that's not it, Gary's Glade is a sure thing, about 12
minutes away."

**Forest signal.** Runs inside `natural=wood` / `landuse=forest` for most of
their length get a glades hint. The QA report says whether mappers carved runs
out of the forest at this resort; if they didn't, every run looks wooded and the
signal is automatically weakened.

**Trail-map vision pass.** Download the resort's trail maps (PNG/JPEG
preferred; several per resort is fine, e.g. Vail front side + back bowls) into
`resorts/<id>/maps/`, which is gitignored since the art belongs to the resort. Then:

```sh
make vision RESORTS=brighton DRY=tiles   # preview the tiles, no API calls
make vision RESORTS=brighton             # needs ANTHROPIC_API_KEY
make graphs RESORTS=brighton
```

Big maps are cut into overlapping tiles so run labels stay readable; each call
sends the whole map (for the legend) plus one zoomed tile. Parks are found by
their orange lines/park symbols, glades by runs drawn through trees. Output is
`resorts/<id>/vision.json`, meant to be reviewed in a PR: delete wrong entries,
set `"confidence": "confirmed"` on ones you know. `unmatched` lists names the
model read that aren't in OSM (often runs nobody has mapped yet). Hand edits in
`features.json` always win over `vision.json`.

## Places (lodges, food, bars, restrooms)

Pulled from OSM (`amenity=restaurant|cafe|bar|toilets|...`, `tourism=alpine_hut`,
buildings named lodge/chalet/hut/yurt) and attached to the nearest node on the
snow; anything over 200 m away (town, parking lots) is dropped. Restroom
searches also match lodges and restaurants, since restrooms inside them are
rarely mapped separately ("Closest restrooms are inside Brighton Lodge").
Lodge names work as `from`/`to` in `find_route` too. Check the QA report's
lodge/food/restroom lists and fill gaps in `pois.json`: dropping a pin in
Google Maps gives you lat/lon.

## How the graph is built

1. Keep lifts and `downhill` runs in the ski area; drop abandoned/proposed lifts.
2. Orient lifts uphill and runs downhill (using elevation when present).
3. Merge endpoints within 40 m into nodes; add nodes where runs cross.
4. Split runs wherever a node touches their interior (mid-run junctions, lifts beside a run).
5. Add short level connectors (skating between a run-out and a lift maze).
6. Link lift bottoms within 400 m and run ends within 150 m of a lift bottom:
   base areas are rarely mapped as pistes.
7. Apply `links.json`.

Edge cost = ride/ski time from rough speeds by lift type and difficulty,
plus a 90 s lift-line penalty per lift boarded and a 30 s penalty for switching
runs, so routes stay on one trail instead of zigzagging at every crossing.

## Known gaps / next up

- [x] MCP adapter (`find_route`, `locate`, `list_resorts`)
- [ ] Lambda + API Gateway + Cloudflare DNS (Terraform)
- [ ] Lift status provider (status.live) with name matching via aliases
- [ ] GPS snapping to edges (not just nodes) and describing the spot
- [ ] Phonetic matching for speech-to-text mishearings
- [ ] Elevation enrichment from a DEM (better orientation + connector checks)
- [ ] Bowl/area polygons (Vail Back Bowls, Breck Peak 8/9/10 bowls) as zones: chutes into them currently dead-end
- [ ] Gondolas as two-way (riding down) and mid-stations
- [ ] Mid-station lifts are treated as one segment
- [ ] Left/right cues ("skier's right off the top") from geometry
