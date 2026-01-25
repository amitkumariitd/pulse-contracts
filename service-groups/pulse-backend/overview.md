# Pulse Backend Service Group

**Repository**: `pulse-backend`  
**Domain**: Trading order management and execution

## Overview

The pulse-backend service group contains all services related to trading order processing, including:
- External API gateway (GAPI)
- Internal order management API (Pulse)
- Background workers for async order processing

## Services

### 1. gapi-api

**Type**: HTTP API (FastAPI)  
**Port**: 8000  
**Purpose**: External-facing gateway API for client applications

**Responsibilities:**
- Public API endpoints (`/api/*`)
- Authentication and authorization (JWT)
- Input validation
- Request routing to internal services
- Internal service endpoints (`/internal/*`)

**Contract**: [services/gapi-api/api.md](services/gapi-api/api.md)  
**OpenAPI**: [services/gapi-api/openapi.yaml](services/gapi-api/openapi.yaml)

---

### 2. pulse-api

**Type**: HTTP API (FastAPI)  
**Port**: 8001  
**Purpose**: Internal trading order domain API

**Responsibilities:**
- Order creation and persistence
- Order state management
- Internal-only endpoints (`/internal/*`)
- Service-to-service communication with GAPI

**Contract**: [services/pulse-api/api.md](services/pulse-api/api.md)  
**OpenAPI**: [services/pulse-api/openapi.yaml](services/pulse-api/openapi.yaml)

---

### 3. pulse-background

**Type**: Background worker (no HTTP)  
**Purpose**: Async order processing

**Responsibilities:**
- Order splitting (parent → child orders)
- Order slice execution (scheduled order placement)
- Timeout monitoring and recovery
- Broker integration (order placement)

**Documentation**: [services/pulse-background/README.md](services/pulse-background/README.md)

---

## Service Boundaries

### GAPI ↔ Pulse: Strict HTTP-only boundary

- GAPI and Pulse communicate ONLY via HTTP APIs
- GAPI MUST NOT import code from Pulse
- Pulse MUST NOT import code from GAPI
- All communication goes through defined API contracts

### Pulse API ↔ Pulse Background: Shared codebase

- Both are deployables of the same `pulse` service
- They CAN share domain logic, repositories, and utilities
- Only entry points differ (HTTP vs background worker)

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    pulse-backend repo                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐         ┌──────────────┐             │
│  │  gapi-api    │  HTTP   │  pulse-api   │             │
│  │  (port 8000) │────────▶│  (port 8001) │             │
│  │              │         │              │             │
│  │ - /api/*     │         │ - /internal/*│             │
│  │ - /internal/*│         │              │             │
│  └──────────────┘         └──────┬───────┘             │
│                                   │                      │
│                                   │ Shared DB            │
│                                   │                      │
│                          ┌────────▼────────┐            │
│                          │ pulse-background │            │
│                          │                  │            │
│                          │ - Order splitter │            │
│                          │ - Slice executor │            │
│                          └──────────────────┘            │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Deployment

All services are deployed together from this repository:

```bash
# Start all services
docker-compose up

# Or run individually
uvicorn main:app --reload --port 8000  # GAPI + Pulse API
python -m pulse.background              # Background workers
```

See: `doc/deployment.md` for detailed deployment instructions.

---

## Data Flow

### Order Creation Flow

1. Client → `POST /api/orders` → gapi-api
2. gapi-api validates request
3. gapi-api → `POST /internal/orders` → pulse-api
4. pulse-api creates order in database (status: PENDING)
5. pulse-api returns order_id
6. gapi-api returns 202 Accepted to client
7. pulse-background picks up PENDING order
8. pulse-background splits order into slices
9. pulse-background schedules slice execution

---

## Future: Multi-Repo Split

When splitting this service group into multiple repos:

**Option 1: Split by domain**
- `gateway-backend` repo → gapi-api service
- `pulse-backend` repo → pulse-api, pulse-background services

**Option 2: Keep together**
- `pulse-backend` repo → all three services (current structure)

The contract structure supports both options.

