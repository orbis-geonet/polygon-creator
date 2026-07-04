# polygon-creator

**The territory-fusion engine of an Orbis deployment.**

A Java Spring Boot background worker that reads geo-tagged *places* from MongoDB, fuses same-tribe places into organic territory polygons, and writes them back for the main Orbis backend to serve. It is a worker, not a public API — own port (default `8090`), same database as the backend.

## Why it exists

Fusion is far too heavy to run per map request, so this service recalculates polygons in the background and the map becomes a cheap read:

- One coverage circle loads up to 50,000 places and compares circle pairs — quadratic, batch-grade work.
- A map view is then a single geo query over the precomputed `polygons` collection.
- Recalculation is incremental: user activity recomputes a ~25 km neighborhood, never the planet; a 150 km grid handles deliberate full-map sweeps.
- The heavy work runs in its own process, so it can never starve the user-facing API.

## The fusion pipeline

Each place is a circle sized by its weight. Per coverage circle:

1. **Resize** — overlapping circles are carved down, newest claim wins; rival overlap shrinks the older circle, and sub-threshold circles drop to radius 0 (`CircleCalculationPointUtils`).
2. **Connect** — tangent bridges link same-tribe circles; bridges colliding with rival circles or bridges are broken (`TangentPointsUtils`, `LinesIntersectLinePointUtils`).
3. **Cluster** — connected circles form clusters, called *palindromes* in the code; one cluster = one polygon (`PlaceIntersectionUtils`).
4. **Outline** — arcs and bridges are interpolated into one closed outline; a lone circle stays a full circle (`PalindromeUtils.merge`).
5. **Clean up** — contained polygons dropped, duplicates deduplicated (newest kept), radius-0 places attached by point-in-polygon.
6. **Label** — the center is the pole of inaccessibility, via a Java port of Mapbox's [polylabel](https://github.com/mapbox/polylabel).

Each `Polygon` document stores the tribe key, member place keys, outline points, and center.

## The scheduler

Work is queued as coverage circles in the `polygonSchedulerCoordinate` collection (`NEW` → `IN_PROCESS` → `DONE`/`ERROR`):

| Type | Radius | Purpose |
|---|---|---|
| `TRIGGER` | 150 km | Full-map sweep — `POST /polygon-calculations/trigger` seeds a hexagonally offset global grid. |
| `CHECKIN` | 25 km | Local recalculation around user activity, queued by the main backend. |

Two thread pools (defaults: 7 `TRIGGER`, 5 `CHECKIN`) each claim their own page of `NEW` coordinates every `delay-in-sec` seconds and run the pipeline. Polygons in the outer 25% of a coverage circle are preserved as edge polygons so adjacent cells don't fight over seams; the rest are replaced. The worker is stateless apart from MongoDB — restart it any time and it resumes from the queue. A daily cron purges old processed coordinates.

## Endpoints

Unauthenticated test/ops API, gated by `test-controller-enable` — keep disabled in production. Actuator exposes `health`, `info`, `metrics`, `prometheus`.

| Endpoint | Method | Purpose |
|---|---|---|
| `/polygon-calculations/map-palindrome` | GET | Run fusion on the fly for a lat/lon (debugging only). |
| `/polygon-calculations/map-palindrome-from-db` | GET | Return persisted polygons around a point. |
| `/polygon-calculations/trigger` | POST | **Destructive.** Clear the queue, seed the global grid. |
| `/polygon-calculations/trigger-one-point` | POST | Queue one `TRIGGER` coordinate. |

## Tech stack

Java 17 · Spring Boot 2.5.5 · Spring Data MongoDB (`places`, `polygons`, `polygonSchedulerCoordinate`) · GeoTools / JTS · Lombok + MapStruct · Micrometer/Prometheus · Docker multi-stage build.

## Configuration

Spring profiles per environment (`application.yaml` base + `prod`, `prod-local`, `staging`). Under `app.polygon-calculation`:

| Property | Default | Description |
|---|---|---|
| `enable` | `true` | Master switch for the background job. |
| `test-controller-enable` | `false` | Enables the HTTP test controller. |
| `delay-in-sec` | `1` | Pause between polling iterations. |
| `start-page` | `0` | Page offset added to each thread's page index. |
| `page-size` | `3` | Coordinates claimed per thread per iteration. |
| `trigger-point-thread-number` | `7` | `TRIGGER` threads. |
| `check-in-point-thread-number` | `5` | `CHECKIN` threads. |

`app.keep-alive.admin-url` sets the admin endpoint pinged every 2 minutes by the keep-alive task. Standard Spring overrides apply: `SPRING_DATA_MONGODB_URI`, `SPRING_PROFILES_ACTIVE`, `PORT`.

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
