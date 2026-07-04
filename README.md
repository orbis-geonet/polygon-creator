# polygon-creator

**The territory-fusion engine of an Orbis deployment.**

`polygon-creator` (internally `orbisv2-polygon-creator`) is a standalone Java Spring Boot background worker that computes the organic, blob-shaped territory polygons you see on the Orbis map. It reads geo-tagged *places* from MongoDB, fuses places that belong to the same group into a single merged territory outline, and persists the resulting polygons back to MongoDB, where the main Orbis backend serves them to clients.

It is a worker process, not a public API. It runs on its own port (default `8090`), separate from the main Orbis backend, and connects independently to the same MongoDB instance.

---

## Why this service exists

Territory polygons could in principle be computed by the main backend at request time, every time a user opens the map. At Orbis scale that is the wrong trade-off, so this service exists for exactly one job: **recalculating polygons in the background so nothing else ever has to compute them.** The code makes the reasoning concrete:

- **Fusion is expensive.** A single coverage circle loads up to 50,000 places in one pass (`PageRequest.of(0, 50000)` in `PlacesService.findPlacesAndCreatePalindrome`), and tangent computation compares every same-tribe pair of circles — quadratic in the number of places (`TangentPointsUtils`). The code instruments itself accordingly: per-coordinate timing logs and heap snapshots before and after every computation (`MemoryUtils.printCurrentMemoryInfo`). This is batch work, not request work.
- **Map reads must be cheap.** The map is the hottest read path in the app. Because this worker persists finished polygons to the `polygons` collection, the backend answers a map view with a single geo query over precomputed documents (the same query this service exposes as `/map-palindrome-from-db`) — zero geometry is executed at request time.
- **Recalculation is incremental.** Territory only changes where activity happens, so the scheduler recomputes neighborhoods, not the planet: a check-in queues a 25 km `CHECKIN` circle around the event, while the 150 km `TRIGGER` grid exists for deliberate full-map sweeps. One user's claim never triggers a global recalculation.
- **Heavy work is isolated.** The recalculation runs in its own JVM, on its own port, with its own thread pools — a full-map sweep can saturate this worker's CPU and memory without ever starving or crashing the user-facing API. Throughput is throttled independently (`delay-in-sec`, `page-size`, thread counts), and the process can be restarted at any time and resume from the queue in MongoDB.

The `/map-palindrome` test endpoint, which runs the full fusion pipeline on the fly for a single point, exists precisely to contrast the two paths: on-demand computation is possible for debugging, but production always serves from the precomputed `polygons` collection.

---

## What it actually does

### 1. The fusion algorithm (circles, tangents, merge)

Every place is modeled as a circle on the map:

1. **Sizing** — each place's circle radius is derived from its weight/size and rescaled relative to neighboring places (`CircleCalculationPointUtils.calculateResizedRadius`). Places whose resized radius falls below the visibility threshold are set to radius 0 and absorbed later.
2. **Connection** — for every pair of circles belonging to the **same group**, external/internal tangent points are computed (`TangentPointsUtils`). A tangent "bridge" is only kept if it does not collide with circles or bridges of *other* groups (`LinesIntersectLinePointUtils.checkPalindromeCollisionAndBreakConnection`) — territories never swallow a rival group's territory.
3. **Clustering** — connected same-group circles are grouped into clusters (`PlaceIntersectionUtils`); each cluster becomes one polygon candidate. In the codebase these merged shapes are called *palindromes*.
4. **Outline generation** — for each cluster, circle arcs and tangent lines are interpolated into dense point chains (`CircleCreatorPointUtils`, `LinesCreatorPointUtils`) and merged into a single closed outline (`PalindromeUtils.merge`). A cluster with a single place simply becomes a full circle.
5. **Cleanup** — polygons fully contained inside another polygon are dropped, duplicate polygons (same `placeKeys`) are deduplicated, and zero-radius places are attached to whichever polygon of their group contains them (point-in-polygon test).
6. **Labeling** — each polygon's visual center is computed as the *pole of inaccessibility* using a bundled Java port of Mapbox's [polylabel](https://github.com/mapbox/polylabel) algorithm (`utils/points/points/polylabel`).

The final `Polygon` document stores the group key, the member place keys, the outline points, and the polylabel center — everything the map renderer needs.

### 2. The scheduler (how the whole planet gets covered)

Recalculation work is queued as `PolygonSchedulerCoordinate` documents in MongoDB. Each coordinate is a coverage circle (center + radius) with a status (`NEW` → `DONE`/`ERROR`) and one of two types:

| Type | Radius | Purpose |
|---|---|---|
| `TRIGGER` | 150 km | Full-map sweeps. `POST /polygon-calculations/trigger` wipes the queue and seeds a hexagonally offset global grid of coverage circles spanning all continents (Antarctica excluded). |
| `CHECKIN` | 25 km | Local recalculation around user activity (e.g. a check-in written by the main backend), so a single event only reprocesses its neighborhood. |

`PolygonCalculationJob` starts two dedicated thread pools (defaults: 7 TRIGGER threads, 5 CHECKIN threads). Every `delay-in-sec` seconds, each thread claims a page of pending coordinates (the page index is derived from the thread number, so threads never contend for the same page), loads up to 50,000 places within the coverage circle, runs the fusion pipeline, and reconciles the results:

- Polygons near the edge of the coverage circle (>75% of the radius from the center) are treated as *edge polygons* and left untouched, so adjacent grid cells do not fight over seam territory.
- Pre-existing polygons in the area that do not overlap an edge polygon are deleted and replaced by the freshly computed set.
- A final aggregation pass removes duplicate polygons, keeping the newest copy.

Processed coordinates are stamped with status, timing, and place/polygon counts, which makes the scheduler collection double as a progress/benchmark log for global sweeps.

### 3. HTTP endpoints (internal/test only)

`PolygonCalculationController` exposes a small unauthenticated API meant for testing and manual operation. It is guarded by `app.polygon-calculation.test-controller-enable` and must stay disabled in production:

| Endpoint | Method | Purpose |
|---|---|---|
| `/polygon-calculations/map-palindrome` | GET | Compute polygons on the fly for a lat/lon + distance (bypasses the DB queue). |
| `/polygon-calculations/map-palindrome-from-db` | GET | Return already-persisted polygons around a point. |
| `/polygon-calculations/trigger` | POST | **Destructive.** Clears the scheduler queue and seeds the global TRIGGER grid. |
| `/polygon-calculations/trigger-one-point` | POST | Queue a single TRIGGER coordinate at a lat/lon. |

Spring Actuator is enabled with `health`, `info`, `metrics`, and `prometheus` endpoints for monitoring.

---

## Tech stack

- Java 17, Spring Boot 2.5.5 (Gradle 7.5.1 wrapper)
- MongoDB via Spring Data MongoDB (documents: `places`, `polygons`, `polygonSchedulerCoordinates`)
- Geospatial: GeoTools 25, JTS, davidmoten/geo, bundled Mapbox polylabel port
- Lombok + MapStruct for boilerplate and DTO mapping
- Micrometer + Prometheus registry for metrics
- Docker multi-stage build (non-root runtime user, JRE-only final image)

## Configuration

Spring profiles select the environment (`application.yaml` is the base, plus `prod`, `prod-local`, and `staging` variants). Key settings under `app.polygon-calculation`:

| Property | Default | Description |
|---|---|---|
| `enable` | `true` | Master switch for the background job (`PolygonCalculationJob` is not created when `false`). |
| `test-controller-enable` | `false` | Enables the HTTP test controller. Keep `false` in production. |
| `delay-in-sec` | `1` | Pause between scheduler polling iterations. |
| `start-page` | `0` | Page offset added to each thread's page index. |
| `page-size` | `3` | Coordinates claimed per thread per iteration. |
| `trigger-point-thread-number` | `7` | Threads for TRIGGER (full-map) work. |
| `check-in-point-thread-number` | `5` | Threads for CHECKIN (local) work. |

Connection settings follow standard Spring conventions and can be overridden by environment variables, e.g. `SPRING_DATA_MONGODB_URI`, `SPRING_PROFILES_ACTIVE`, and `PORT` (`server.port` defaults to `8090`).

## Build and run

Local build and test:

```bash
./gradlew clean build
./gradlew bootRun   # uses the default profile; pass SPRING_PROFILES_ACTIVE to switch
```

Docker:

```bash
docker build -t orbis/polygon-creator .
docker run -d \
  -e SPRING_DATA_MONGODB_URI="mongodb://mongo:27017/?retryWrites=true" \
  -e SPRING_PROFILES_ACTIVE=prod \
  -p 8090:8090 \
  orbis/polygon-creator
```

Bare-metal (systemd) deployments only need a JRE 17+:

```ini
[Unit]
Description=Orbis Polygon Creator
After=network.target

[Service]
ExecStart=java -jar /opt/orbis/orbisv2-polygon-creator-0.0.1-SNAPSHOT.jar
Environment=SPRING_PROFILES_ACTIVE=prod
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Kick off a full-map computation by inserting TRIGGER coordinates (via the test controller in a non-production environment, or by writing `PolygonSchedulerCoordinate` documents directly). The worker picks them up on its next polling cycle.

## Operational notes

- The service is stateless apart from MongoDB; it can be restarted at any time and will resume from the scheduler queue.
- Do not run two instances with overlapping thread/page configurations against the same queue unless you adjust `start-page` so their page ranges do not overlap.
- Memory usage is logged before and after each coverage-circle computation (`MemoryUtils`); dense urban areas with tens of thousands of places are the expensive case.
- The port must not clash with the main Orbis backend when co-located on one host.

## License

AGPL-3.0 — see [LICENSE](LICENSE).

---

*polygon-creator — internal background service of the Orbis platform. Not a public API.*
