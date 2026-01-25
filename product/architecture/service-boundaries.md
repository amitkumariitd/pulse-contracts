# Service Boundaries

This document defines the boundaries between services and how they communicate.

## Overview

The trading platform is composed of multiple services that work together to provide order execution functionality. Each service has clear responsibilities and communicates with other services through well-defined APIs.

## Service Architecture

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

## Service Definitions

### gapi-api (Gateway API)

**Type**: HTTP API  
**Port**: 8000  
**Purpose**: External-facing gateway for client applications

**Responsibilities:**
- Public API endpoints (`/api/*`)
- Authentication and authorization
- Input validation
- Request routing to internal services
- Rate limiting and throttling
- API versioning

**Does NOT:**
- Contain business logic
- Access database directly
- Process orders

**Communication:**
- **Inbound**: Client applications (HTTP/REST)
- **Outbound**: pulse-api (HTTP/REST)

---

### pulse-api (Pulse API)

**Type**: HTTP API  
**Port**: 8001  
**Purpose**: Internal trading order domain API

**Responsibilities:**
- Order creation and persistence
- Order state management
- Domain business logic
- Database access via repositories
- Internal-only endpoints (`/internal/*`)

**Does NOT:**
- Handle authentication (trusts GAPI)
- Validate external inputs (GAPI does this)
- Execute orders (pulse-background does this)

**Communication:**
- **Inbound**: gapi-api (HTTP/REST)
- **Outbound**: Database (PostgreSQL)

---

### pulse-background (Background Workers)

**Type**: Background worker (no HTTP)  
**Purpose**: Async order processing and execution

**Responsibilities:**
- Order splitting (parent → child slices)
- Order slice execution (scheduled placement)
- Timeout monitoring and recovery
- Broker integration (order placement)

**Does NOT:**
- Expose HTTP endpoints
- Handle client requests
- Validate inputs (pulse-api does this)

**Communication:**
- **Inbound**: Database (polling for work)
- **Outbound**: Database (updates), Broker APIs (order placement)

---

## Communication Patterns

### 1. GAPI ↔ Pulse: Strict HTTP-only boundary

**Rule**: GAPI and Pulse communicate ONLY via HTTP APIs

**Why:**
- Enables future multi-repo split
- Clear service boundaries
- Independent deployment
- Language/framework independence

**Forbidden:**
- ❌ Direct function calls across services
- ❌ Shared in-memory state
- ❌ Direct database access from GAPI

**Allowed:**
- ✅ HTTP REST API calls
- ✅ Shared schemas (via contracts)
- ✅ Shared tracing headers

**Example:**
```python
# ❌ WRONG: Direct function call
from pulse.services.order_service import create_order
result = create_order(order_data)

# ✅ CORRECT: HTTP API call
response = await http_client.post(
    f"{pulse_api_base_url}/internal/orders",
    json=order_data
)
```

---

### 2. Pulse API ↔ Pulse Background: Shared codebase

**Rule**: Both are deployables of the same service, can share code

**Why:**
- Same domain (trading orders)
- Same database
- Same repository (pulse-backend)

**Allowed:**
- ✅ Shared domain logic
- ✅ Shared repositories
- ✅ Shared utilities
- ✅ Same database schema

**Communication:**
- Database is the integration point
- pulse-api writes orders (status: PENDING)
- pulse-background polls for PENDING orders
- pulse-background updates order status

---

## Data Flow

### Order Creation Flow

```
1. Client → POST /api/orders → gapi-api
   ↓
2. gapi-api validates request (auth, input)
   ↓
3. gapi-api → POST /internal/orders → pulse-api
   ↓
4. pulse-api creates order in DB (status: PENDING)
   ↓
5. pulse-api returns order_id
   ↓
6. gapi-api returns 202 Accepted to client
   ↓
7. pulse-background polls DB for PENDING orders
   ↓
8. pulse-background splits order into slices
   ↓
9. pulse-background schedules slice execution
```

---

## Future: Multi-Repo Split

When splitting into multiple repositories:

### Option 1: Split by domain

**Repo 1: gateway-backend**
- Service: gapi-api

**Repo 2: pulse-backend**
- Services: pulse-api, pulse-background

**Communication:**
- gapi-api → pulse-api (HTTP)
- Both repos reference `contracts/` as git submodule

### Option 2: Keep together

**Repo 1: pulse-backend**
- Services: gapi-api, pulse-api, pulse-background

**Why:**
- Simpler deployment
- Faster iteration
- Shared contracts in same repo

---

## Service Boundary Rules

### Mandatory Rules

1. **GAPI MUST NOT import code from Pulse**
2. **Pulse MUST NOT import code from GAPI**
3. **All cross-service communication MUST use HTTP APIs**
4. **All APIs MUST be defined in contracts/ before implementation**
5. **Shared schemas MUST come from contracts/schemas/**

### Best Practices

1. Use request/trace IDs for distributed tracing
2. Propagate context across service boundaries
3. Handle errors gracefully at boundaries
4. Use idempotency keys for duplicate prevention
5. Version APIs to support backward compatibility

---

## Related Documentation

- Service Contracts: [../../service-groups/pulse-backend/](../../service-groups/pulse-backend/)
- Product Overview: [../overview.md](../overview.md)
- Data Flow: [data-flow.md](data-flow.md)

