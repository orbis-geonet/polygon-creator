# polygon-creator

**The territory-fusion engine of an Orbis deployment.**

A standalone Java Spring Boot background worker that computes the organic territory polygons on the Orbis map: it reads geo-tagged *places* from MongoDB, fuses same-tribe places into merged territory outlines, and writes the finished polygons back to MongoDB for the main Orbis backend to serve. It is a worker, not a public API — it runs on its own port (default `8090`), separate from the backend, against the same database.

## Why it exists

The service exists for one job: **recalculating polygons in the background so nothing else ever computes them at request time.**

- **Fusion is expensive.** One coverage circle loads up to 50,000 places (`PageRequest.of(0, 50000)` in `PlacesService`) and compares circle pairs for resizing and tangents — quadratic work, instrumented with per-coordinate timing and heap logging (`MemoryUtils`). Batch work, not request work.
- **Map reads must be cheap.** The map is the hottest read path in the app. Since polygons are precomputed into the `polygons` collection, a map view is a single geo query — zero geometry at request time.
- **Recalculation is incremental.** A `CHECKIN` coordinate recomputes only a small neighborhood (25 km by convention) around user activity; the 150 km `TRIGGER` grid is for deliberate full-map sweeps. One claim never recomputes the planet.
- **Heavy work is isolated.** A full-map sweep can saturate this worker's CPU and memory without touching the user-facing API, and its throughput is throttled independently (`delay-in-sec`, `page-size`, thread counts).

## How it works

### The fusion pipeline

Each place is a circle sized by its weight; per coverage circle the pipeline is:

1. **Resize** — overlapping circles are carved down, newest claim wins: on rival-tribe overlap the older circle shrinks by the overlap (`CircleCalculationPointUtils`); circles below the visibility threshold drop to radius 0.
2. **Connect** — external/internal tangent bridges are computed between same-tribe circles only (`TangentPointsUtils`); bridges that collide with another tribe's circles or bridges are broken (`LinesIntersectLinePointUtils`).
3. **Cluster** — connected same-tribe circles group into clusters (`PlaceIntersectionUtils`); each cluster becomes one polygon (in the code these merged shapes are called *palindromes*).
4. **Outline** — circle arcs and tangent lines are interpolated into point chains and merged into one closed outline (`PalindromeUtils.merge`); a single-place cluster is a full circle.
5. **Clean up** — polygons contained in another polygon are dropped, duplicates (same `placeKeys`) deduplicated keeping the newest, and radius-0 places attached to their tribe's containing polygon.
6. **Label** — the polygon center is the pole of inaccessibility, via a bundled Java port of Mapbox's [polylabel](https://github.com/mapbox/polylabel).

The resulting `Polygon` document holds the tribe (group) key, member place keys, outline points, and center — everything the map renderer needs.

### The scheduler

Work is queued as coverage circles in the `polygonSchedulerCoordinate` collection (status `NEW` → `IN_PROCESS` → `DONE`/`ERROR`):

| Type | Radius | Purpose |
|---|---|---|
| `TRIGGER` | 150 km | Full-map sweeps: `POST /polygon-calculations/trigger` clears the queue and seeds a hexagonally offset global grid (Antarctica excluded). |
| `CHECKIN` | 25 km | Local recalculation around user activity written by the main backend. |

`PolygonCalculationJob` runs two thread pools (defaults: 7 `TRIGGER`, 5 `CHECKIN`). Every `delay-in-sec` seconds each thread claims its own page of `NEW` coordinates (page index = thread number; claimed by flipping status to `IN_PROCESS`), runs the pipeline, then reconciles: polygons in the outer 25% of the coverage circle are treated as edge polygons and preserved so adjacent grid cells don't fight over seams, while non-edge polygons in the area are replaced by the fresh set. Coordinates are stamped with timing and place/polygon counts (making the collection a progress log for sweeps); a daily cron purges `DONE` entries older than a day and all `ERROR` entries. The worker is stateless apart from MongoDB — restart it any time and it resumes from the queue.

### Endpoints

`PolygonCalculationController` is an unauthenticated test/ops API, gated by `test-controller-enable` — keep it disabled in production. Spring Actuator exposes `health`, `info`, `metrics`, and `prometheus`.

| Endpoint | Method | Purpose |
|---|---|---|
| `/polygon-calculations/map-palindrome` | GET | Run the fusion pipeline on the fly for a lat/lon + distance (debugging; production serves from the DB). |
| `/polygon-calculations/map-palindrome-from-db` | GET | Return persisted polygons around a point. |
| `/polygon-calculations/trigger` | POST | **Destructive.** Clear the queue and seed the global `TRIGGER` grid. |
| `/polygon-calculations/trigger-one-point` | POST | Queue one `TRIGGER` coordinate at a lat/lon. |

## Tech stack

Java 17 · Spring Boot 2.5.5 · Spring Data MongoDB (`places`, `polygons`, `polygonSchedulerCoordinate`) · GeoTools 25 / JTS · Lombok + MapStruct · Micrometer/Prometheus · Docker multi-stage build (non-root, JRE-only image).

## Configuration

Spring profiles per environment (`application.yaml` base, plus `prod`, `prod-local`, `staging`). Under `app.polygon-calculation`:

| Property | Default | Description |
|---|---|---|
| `enable` | `true` | Master switch — the background job is not created when `false`. |
| `test-controller-enable` | `false` | Enables the HTTP test controller. Keep `false` in production. |
| `delay-in-sec` | `1` | Pause between scheduler polling iterations. |
| `start-page` | `0` | Page offset added to each thread's page index. |
| `page-size` | `3` | Coordinates claimed per thread per iteration. |
| `trigger-point-thread-number` | `7` | Threads for `TRIGGER` (full-map) work. |
| `check-in-point-thread-number` | `5` | Threads for `CHECKIN` (local) work. |

Standard Spring overrides apply, e.g. `SPRING_DATA_MONGODB_URI`, `SPRING_PROFILES_ACTIVE`, `PORT`.

## Build and run

```bash
./gradlew clean build
./gradlew bootRun
```

```bash
docker build -t orbis/polygon-creator .
docker run -d \
  -e SPRING_DATA_MONGODB_URI="mongodb://mongo:27017/?retryWrites=true" \
  -e SPRING_PROFILES_ACTIVE=prod \
  -p 8090:8090 \
  orbis/polygon-creator
```

## License

AGPL-3.0 — see [LICENSE](LICENSE).
