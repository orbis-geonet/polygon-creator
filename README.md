# orbisv2-polygon-creator

> **Polygon Calculation Background Service**

---

## Overview

`orbisv2-polygon-creator` is a dedicated Java Spring Boot background service responsible for computing and persisting polygon data for the Orbis platform. It is intentionally designed as a standalone worker process — not an API server.

> ⚠️ **Important:** This service does **not** expose a public-facing REST API. It runs on a separate port from the main Orbis Java backend and should never be confused with or replace the primary application server. Its sole responsibility is to run scheduled polygon calculation jobs in the background.

---

## Architecture & Port Isolation

The service is architected for strict separation of concerns:

- The main Orbis backend API runs on its own port (typically `8080`).
- `orbisv2-polygon-creator` runs on a **different port** to avoid conflicts.
- Both services connect independently to the same MongoDB instance.
- The polygon creator operates in a fire-and-forget loop, paginating through records and computing polygons according to its schedule configuration.

Configure the service port in `application.properties` or your environment-specific YAML file:

```properties
server.port=8081   # Use any port not occupied by the main backend
```

---

## Tech Stack

- Java (Spring Boot)
- MongoDB (via Spring Data MongoDB)
- Docker & Docker Compose (deployment)
- YAML-based environment configuration

---

## Prerequisites

To build and run locally you need:

- JDK 11 or later
- Gradle (wrapper included)
- A running MongoDB instance
- Docker + Docker Compose (for containerized deployment)

---

## Configuration

Configuration is managed through Spring Boot YAML profiles. Below is the reference for the production-local profile (`application-prod-local.yml`):

```yaml
spring:
  data:
    mongodb:
      database: orbis_clone
      uri: mongodb://localhost:27017/?retryWrites=true

logging:
  level:
    org:
      springframework:
        data:
          mongodb:
            core:
              MongoTemplate: DEBUG
            repository:
              query: DEBUG

app:
  polygon-calculation:
    enable: true
    test-controller-enable: false
    delay-in-sec: 1
    start-page: 0
    page-size: 3
```

### `app.polygon-calculation` properties

| Property | Default | Description |
|---|---|---|
| `enable` | `true` | Master switch. Set to `false` to disable all polygon calculation jobs. |
| `test-controller-enable` | `false` | Exposes a test HTTP endpoint when `true`. Keep `false` in production. |
| `delay-in-sec` | `1` | Pause in seconds between processing iterations. |
| `start-page` | `0` | Pagination start page when scanning records. |
| `page-size` | `3` | Number of records processed per page/batch. |

## Deploying on Any Server

This service has no OS-specific dependencies beyond a JRE (or Docker). To deploy on any Linux server:

1. Ensure Java 11+ is installed, or use a Docker JRE image.
2. Copy the JAR to a target directory on the server.
3. Create an application YAML for the target environment (database URI, port, profile).
4. Run the JAR directly with `java -jar` or wrap it in a systemd service / Docker Compose stack.
5. Confirm the chosen port is open in any firewall rules and does not clash with the main API.

Systemd service example for non-Docker deployments:

```ini
[Unit]
Description=Orbis Polygon Creator
After=network.target

[Service]
ExecStart=java -jar /opt/orbis/orbisv2-polygon-creator-0.0.1-SNAPSHOT.jar
Environment=SPRING_PROFILES_ACTIVE=prod-local
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

*orbisv2-polygon-creator — Internal background service. Not a public API.*
