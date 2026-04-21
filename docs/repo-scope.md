# Repo Scope

## What this repository contains

This repository is the Python backend for OpenTAKServer. It contains the server runtime, API surface, background workers, database models, migrations, certificate management code, and optional integration points used by the platform.

It does not contain the separate Web UI frontend. The README links that UI as [OpenTAKServer-UI](https://github.com/brian7704/OpenTAKServer-UI).

## Main runnable processes

The `pyproject.toml` entrypoints define three backend processes:

- `opentakserver`
  - Main Flask application and Socket.IO server.
  - Bootstraps config, logging, migrations, security, plugins, scheduled jobs, certificate authority, and API blueprints.
- `cot_parser`
  - RabbitMQ consumer that parses incoming CoT payloads, persists data, and emits updates to Socket.IO clients.
  - Runs as a separate process from the web app.
- `eud_handler`
  - Raw TAK client socket listener for TCP or SSL client traffic.
  - Runs independently from the main Flask app and the parser.

## Backend subsystems in this repo

### Application bootstrap

`opentakserver/app.py` is the primary bootstrap module. It is responsible for:

- Loading default config from `opentakserver/defaultconfig.py`
- Loading `config.yml` from `OTS_DATA_FOLDER` when present
- Creating the initial config file on first run
- Running Alembic migrations at startup
- Initializing Flask extensions
- Registering the Marti-compatible, OTS, scheduler, and Socket.IO blueprints
- Creating default roles, groups, and the bootstrap admin account
- Starting the gevent-backed Socket.IO server

### API layers

The backend exposes two main HTTP API families:

- `opentakserver/blueprints/marti_api`
  - TAK/Marti-compatible endpoints for certificates, missions, groups, video, CoT, contacts, data sync, and related client flows.
- `opentakserver/blueprints/ots_api`
  - OTS-specific endpoints for health, user management, groups, packages, plugins, missions, markers, video, MediaMTX integration, Meshtastic integration, tokens, scheduler control, and related admin functionality.

There is also a Socket.IO blueprint used for live event delivery to UI clients.

### Data model and migrations

Persistent state is modeled in `opentakserver/models` and migrated through Alembic files in `opentakserver/migrations`.

The models cover:

- Users, roles, teams, groups, and tokens
- CoT events and derived objects such as points, markers, routes, alerts, CasEvac, and GeoChat
- Missions, mission content, invitations, roles, and logs
- Device profiles, packages, plugins, certificates, video streams, and recordings
- Meshtastic channel state and scheduled job storage

### Background and integration components

Other important components in this repo include:

- `opentakserver/eud_handler`
  - TAK socket handling and client session management.
- `opentakserver/cot_parser`
  - CoT parsing, normalization, database persistence, and downstream event fanout.
- `opentakserver/plugins`
  - Plugin base classes and runtime plugin loading.
- `opentakserver/certificate_authority.py`
  - Certificate authority creation, server certificate issuance, and enrollment support.
- `opentakserver/controllers`
  - RabbitMQ and Meshtastic-related controllers.
- `opentakserver/forms`
  - Forms used by portions of the backend API layer.
- `opentakserver/maps`
  - Map source definitions shipped with the backend.
- `opentakserver/translations`
  - Localization assets used by the server.

## External components assumed by this repo

The current code assumes several external services or runtime dependencies:

- PostgreSQL for the primary relational database
- RabbitMQ for messaging between backend processes and Socket.IO fanout
- Optional MediaMTX for video path management and recording workflows
- Likely an external nginx layer for Marti/client-certificate HTTPS flows
- Optional external systems such as LDAP, Meshtastic, Mumble, and TAK.gov integration points

## Current architectural shape

Today this repository is not structured as a single self-contained process. It is a small backend system composed of multiple cooperating Python services plus infrastructure dependencies. The main app, parser, and TAK socket listener are intended to run together against the same database, RabbitMQ broker, and shared `OTS_DATA_FOLDER` state.
