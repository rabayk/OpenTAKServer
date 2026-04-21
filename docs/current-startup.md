# Current Startup

## Overview

In its current format, this repository starts as a small multi-process backend system rather than a single binary or a complete self-contained stack.

The usual runtime shape is:

1. PostgreSQL
2. RabbitMQ
3. Optional MediaMTX
4. The main `opentakserver` web and Socket.IO process
5. One or more `cot_parser` worker processes
6. One or more `eud_handler` listener processes
7. Optionally nginx in front of Marti and certificate-based HTTPS flows

## Runtime prerequisites

The code currently assumes the following are available:

- Python 3.10 to 3.14
- A working package install path for the backend, typically `poetry install` or `pip install -e .`
- PostgreSQL reachable from the configured `SQLALCHEMY_DATABASE_URI`
- RabbitMQ reachable from `OTS_RABBITMQ_SERVER_ADDRESS`
- OpenSSL available for certificate-related flows
- Optional MediaMTX if video streaming features are enabled
- Optional nginx if you need the current Marti HTTPS and forwarded client-certificate behavior

## Important runtime state

The application uses `OTS_DATA_FOLDER` for persistent local state. By default this resolves to `~/ots`.

Important files and directories under that folder include:

- `config.yml`
- CA and certificate material
- log files
- uploads
- MediaMTX-related files such as `mediamtx/mediamtx.yml` and recording output

On first run, the main app and the auxiliary processes can generate `config.yml` if it does not exist.

## Configuration sources

Configuration currently comes from two places:

- Environment variables, via `opentakserver/defaultconfig.py`
- `config.yml` inside `OTS_DATA_FOLDER`, loaded after defaults

Important defaults already present in code:

- Main app listener: `OTS_LISTENER_ADDRESS=127.0.0.1`, `OTS_LISTENER_PORT=8081`
- Marti ports used in generated config and certificates: `OTS_MARTI_HTTP_PORT=8080`, `OTS_MARTI_HTTPS_PORT=8443`
- TAK TCP listener: `OTS_TCP_STREAMING_PORT=8088`
- TAK SSL listener: `OTS_SSL_STREAMING_PORT=8089`
- RabbitMQ host: `127.0.0.1`
- Database URI default: PostgreSQL on `127.0.0.1` with database `ots`

Before starting the system, review at least:

- `SQLALCHEMY_DATABASE_URI`
- `OTS_RABBITMQ_SERVER_ADDRESS`
- `OTS_RABBITMQ_USERNAME`
- `OTS_RABBITMQ_PASSWORD`
- `OTS_DATA_FOLDER`
- `OTS_MEDIAMTX_ENABLE`
- `OTS_ENABLE_LDAP`
- `OTS_ENABLE_MESHTASTIC`
- `OTS_ENABLE_MUMBLE_AUTHENTICATION`

## Recommended startup order

### 1. Install the backend from the local checkout

Example options:

```powershell
poetry install
```

or

```powershell
pip install -e .
```

This makes the console scripts from `pyproject.toml` available:

- `opentakserver`
- `cot_parser`
- `eud_handler`

### 2. Prepare PostgreSQL

Create and make reachable:

- a PostgreSQL server
- a database for OTS
- credentials that match `SQLALCHEMY_DATABASE_URI`

The current default config expects PostgreSQL rather than SQLite.

### 3. Prepare RabbitMQ

Start RabbitMQ and ensure the configured host and credentials are reachable by all OTS processes.

The main app creates the exchanges it needs during initialization. The parser and socket handlers then use RabbitMQ for CoT processing and event distribution.

### 4. Generate config if you want a file before first boot

You can let the app create `config.yml` automatically on first run, or generate it explicitly:

```powershell
flask --app opentakserver.app ots generate-config
```

Review the generated `config.yml` in `OTS_DATA_FOLDER` and replace placeholder values, especially the database password in the default URI.

### 5. Create certificate authority material if needed

The main app calls CA creation during initialization, but the repo also exposes a CLI command if you want to do this explicitly:

```powershell
flask --app opentakserver.app ots create-ca
```

### 6. Start the main app

```powershell
opentakserver
```

What happens during startup:

- `config.yml` is loaded or created
- Alembic migrations are applied automatically
- Flask extensions are initialized
- RabbitMQ exchanges are declared
- the certificate authority is checked and created if missing
- default roles and groups are created
- an `administrator` user is created if no admin exists
- the gevent-backed Flask and Socket.IO process starts on `OTS_LISTENER_PORT`

The main health endpoint is:

- `/api/health`

### 7. Start the CoT parser worker

```powershell
cot_parser
```

Notes:

- The parser consumes from RabbitMQ and writes parsed CoT data into the database.
- `OTS_COT_PARSER_PROCESSES` controls how many worker children it spawns.
- The implementation uses `os.fork()`, so the current parser runtime is effectively Linux-oriented.

### 8. Start the TAK socket listener

TCP listener:

```powershell
eud_handler --no-ssl
```

SSL listener:

```powershell
eud_handler --ssl
```

You may run one or both, depending on how clients connect.

The status endpoint exposed by `eud_handler` is:

- `/status`

## Reverse proxy and Marti HTTPS assumptions

The codebase contains multiple references to nginx forwarding a verified client certificate in the `X-Ssl-Cert` header. That means the current Marti HTTPS and some mission or enrollment flows are not documented in this repo as direct app-native TLS termination only. The current architecture appears to assume a reverse proxy in front of the Python app for those client-certificate workflows.

`ProxyFix` is also enabled in the main app, which further suggests deployment behind a proxy.

## First-run behavior and sharp edges

Be aware of the current operational behavior:

- The main app runs database migrations during startup.
- If no admin user exists, startup creates an `administrator` account with password `password`.
- On first run the app attempts to download icon data from GitHub into `OTS_DATA_FOLDER`.
- If `OTS_MEDIAMTX_ENABLE` is true, the app expects a writable `mediamtx/mediamtx.yml` under `OTS_DATA_FOLDER`.
- `cot_parser` uses `os.fork()`, which is not Windows-friendly.
- The top-level Dockerfile healthcheck uses `curl`, but the image does not install `curl`.
- Existing Dockerfiles install from GitHub rather than building the current local checkout.

## Minimal manual verification

After startup, verify at least:

- the main app responds on `/api/health`
- the database schema has been migrated
- RabbitMQ exchanges exist and are reachable
- `cot_parser` is consuming without errors
- `eud_handler` is listening on the expected TCP and or SSL port
- `OTS_DATA_FOLDER` contains the expected config, logs, and certificate state
