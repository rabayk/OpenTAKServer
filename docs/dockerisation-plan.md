# Dockerisation Plan

## Goal

Containerize the current backend shape in a way that is practical for local or lab deployment first, without pretending the system is a single-process app.

The initial target should be `docker compose`, not a production-hardened deployment.

## Proposed service layout

Use one backend image and run it with different commands for each Python process:

- `app`
  - Runs `opentakserver`
- `cot_parser`
  - Runs `cot_parser`
- `eud_handler_tcp`
  - Runs `eud_handler --no-ssl`
- `eud_handler_ssl`
  - Runs `eud_handler --ssl`
- `postgres`
  - Primary relational database
- `rabbitmq`
  - Messaging backbone for the app, parser, and socket handlers
- `mediamtx`
  - Optional service for video-related features
- `nginx`
  - Optional reverse proxy for Marti HTTPS and forwarded client-certificate flows

## Shared state and volumes

The Python services should share a persistent volume for `OTS_DATA_FOLDER` because that folder contains runtime state used across processes:

- `config.yml`
- certificates and CA material
- logs
- uploads
- MediaMTX-related files

PostgreSQL and RabbitMQ should use their own persistent volumes as normal.

## Configuration model

Move runtime configuration into compose-managed environment variables and mounted files:

- Keep common OTS settings in an `.env` or compose environment section
- Mount a persistent directory for `OTS_DATA_FOLDER`
- Keep secrets such as database credentials, RabbitMQ credentials, and CA passwords out of the image
- If `config.yml` remains part of the runtime model, treat it as generated or volume-backed state rather than baking it into the image

## Networking and dependency model

Expected dependency chain:

- `app` depends on `postgres` and `rabbitmq`
- `cot_parser` depends on `postgres`, `rabbitmq`, and the shared OTS data volume
- `eud_handler_*` depends on `postgres`, `rabbitmq`, and the shared OTS data volume
- `nginx` should front only the endpoints that require proxy behavior
- `mediamtx` is optional and should be enabled only when video features are needed

Expose at minimum:

- `8081` for the main app if running directly
- `8088` for TCP TAK traffic
- `8089` for SSL TAK traffic
- `8080` and `8443` only if the compose stack includes the Marti-facing proxy shape

## Image strategy

Replace the existing Dockerfiles that install from GitHub with a local-build image based on this checkout.

The shared backend image should:

- install system dependencies actually required at runtime
- install the local package from the current source tree
- avoid embedding mutable runtime state inside the image
- support changing the process via `command`

This keeps the app, parser, and handlers on the same source revision and reduces drift between containers.

## Recommended implementation sequence

### Phase 1

- Create a single backend `Dockerfile` that builds from the local checkout
- Add a `docker-compose.yml` with `app`, `cot_parser`, `eud_handler_tcp`, `postgres`, and `rabbitmq`
- Add a sample env file documenting required variables
- Mount a named volume for `OTS_DATA_FOLDER`
- Use healthchecks that match actual installed tools and exposed endpoints

### Phase 2

- Add `eud_handler_ssl` if SSL listener support is needed in-compose
- Add optional `mediamtx`
- Add optional `nginx` for Marti HTTPS and client-certificate forwarding
- Document how the reverse proxy populates `X-Ssl-Cert`

### Phase 3

- Tighten image size, startup ordering, restart policies, and observability
- Decide whether startup migrations stay in the app container or move to an explicit one-shot task
- Decide whether CA generation remains implicit or becomes an initialization step

## Constraints and known issues to carry into the container design

- `cot_parser` uses `os.fork()`, so target Linux containers for that service.
- The current code assumes a shared writable `OTS_DATA_FOLDER`.
- The current top-level Dockerfile includes a healthcheck that relies on `curl` even though `curl` is not installed.
- The current Dockerfiles install from GitHub instead of the checked-out source tree.
- Some Marti and mission flows appear to assume nginx-based client-certificate verification and header forwarding.
- MediaMTX support is optional, but the app expects local file paths under `OTS_DATA_FOLDER` when enabled.

## Out of scope for the first Docker pass

The initial compose design should not try to solve full production operations. Defer these to a later hardening pass:

- secret management beyond basic compose env handling
- external TLS certificate lifecycle
- backup and restore procedures
- HA or horizontal scaling strategy
- production monitoring and alerting
- resource tuning and isolation policies
