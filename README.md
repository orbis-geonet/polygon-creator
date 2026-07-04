# polygon-creator

**The territory-fusion engine of an Orbis deployment.**

A Java Spring Boot background worker that reads geo-tagged *places* from MongoDB, fuses same-tribe places into organic territory polygons, and writes them back for the main Orbis backend to serve. It is a worker, not a public API.

This service recalculates polygons in the background and the map becomes a cheap read

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
