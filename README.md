# Contract Documentation

This directory contains all API contracts, schemas, and product documentation for the trading platform.

## Structure

```
contracts/                                    # Root (git submodule)
├── README.md                                # This file
│
├── service-groups/                          # Service groups (each = one repo)
│   └── pulse-backend/                       # pulse-backend repo
│       ├── overview.md                      # Service group overview
│       └── services/                        # Deployables in this repo
│           ├── gapi-api/                    # GAPI API service (deployable)
│           │   ├── api.md                   # Human-readable API docs
│           │   └── openapi.yaml             # OpenAPI 3.0 spec
│           ├── pulse-api/                   # Pulse API service (deployable)
│           │   ├── api.md                   # Human-readable API docs
│           │   └── openapi.yaml             # OpenAPI 3.0 spec
│           └── pulse-background/            # Background workers (deployable)
│               └── README.md                # Worker documentation
│
├── schemas/                                 # Shared schemas across all services
│   ├── common.md                           # Human-readable shared schemas
│   ├── common.yaml                         # OpenAPI components (machine-readable)
│   └── domain/                             # Domain object schemas (future)
│
├── guides/                                  # Cross-service implementation guides
│   ├── tracing.md                          # Distributed tracing standards
│   ├── context.md                          # Request context propagation
│   ├── logging.md                          # Structured logging standards
│   ├── concurrency.md                      # Concurrency safety patterns
│   ├── config.md                           # Configuration standards
│   └── testing.md                          # Testing standards
│
└── product/                                # Product documentation
    ├── overview.md                         # Product vision and context
    ├── features/                           # Feature specifications
    │   └── split-orders.md                 # Split orders feature spec
    └── architecture/                       # Cross-service architecture
        └── service-boundaries.md           # Service boundaries and communication
```

**Note**: This `contracts/` folder is designed to be shared across all repositories as a git submodule.

## Terminology

| Term | Definition | Example |
|------|------------|---------|
| **Service Group** | Git repository containing related services | `pulse-backend` |
| **Service** | Deployable unit (container/process) | `gapi-api`, `pulse-api`, `pulse-background` |
| **Endpoint** | HTTP route within a service | `POST /api/orders` |

## Service Groups

### pulse-backend

Current repository containing trading domain services.

**Services:**
- `gapi-api` - External-facing gateway API (FastAPI, port 8000)
- `pulse-api` - Internal trading order API (FastAPI, port 8001)
- `pulse-background` - Background workers for async processing (no HTTP)

See: [service-groups/pulse-backend/overview.md](service-groups/pulse-backend/overview.md)

## Shared Schemas

Common schemas, enums, and error formats used across all services.

- `schemas/common.md` - Human-readable shared schemas
- `schemas/common.yaml` - OpenAPI components (machine-readable)
- `schemas/domain/` - Domain object schemas (Order, Instrument, etc.)

## Implementation Guides

Cross-service implementation standards that apply to all backend and frontend services.

- `guides/tracing.md` - Distributed tracing standards (trace_id, request_id, span hierarchy)
- `guides/context.md` - Request context propagation across service boundaries
- `guides/logging.md` - Structured logging format and standards
- `guides/concurrency.md` - Concurrency safety patterns (idempotency, locking, atomic operations)
- `guides/config.md` - Configuration management standards
- `guides/testing.md` - Testing standards and best practices

**Note**: Repo-specific guides (e.g., PostgreSQL setup, IDE configuration) remain in each repo's `doc/guides/`.

## Product Documentation

Product specifications and architecture documentation.

- `product/overview.md` - Product vision and context
- `product/features/` - Feature specifications
- `product/architecture/` - Cross-service architecture

## Contract Standards

### API Contract Rules

1. **All APIs MUST be defined here before implementation**
2. **Do NOT invent endpoints, fields, or error formats**
3. **Each service has both OpenAPI spec and human-readable docs**
4. **Shared schemas MUST come from `schemas/`**

### File Formats

- **OpenAPI YAML** (`openapi.yaml`) - Machine-readable, source of truth for contracts
- **Markdown** (`api.md`) - Human-readable, easier to review and understand

Both formats MUST be kept in sync. OpenAPI is the source of truth.

## Usage

### For Developers

1. Check service contract before implementing endpoints
2. Update OpenAPI spec when adding/changing endpoints
3. Update markdown docs to match OpenAPI spec
4. Reference shared schemas from `schemas/`

### For Product Managers

1. Read product documentation in `product/`
2. Review feature specifications in `product/features/`
3. Understand service boundaries in `product/architecture/`

### For API Consumers

1. Use OpenAPI specs to generate SDKs
2. Read markdown docs for human-friendly API documentation
3. Check `schemas/common.md` for shared types and error formats

## Future: Multi-Repo Split

When splitting into multiple repositories, each service group folder becomes its own repo:

```
Repo 1: pulse-backend (trading domain)
  └── services: pulse-api, pulse-background

Repo 2: gateway-backend (API gateway)
  └── services: gapi-api

Repo 3: analytics-backend (analytics domain)
  └── services: analytics-api, analytics-worker
```

Each repo would reference `contracts/` as a git submodule.

## Questions?

See `.augment/rules/rules.md` for mandatory API contract rules.

