# Pulse Contracts - AI Assistant Guide

## Project Identity

This is the **API Contracts & Documentation Repository** for the Pulse trading platform (NOT blockchain smart contracts). It serves as the single source of truth for:
- API specifications (OpenAPI + Markdown)
- Shared schemas across services
- Implementation standards
- Product documentation and architecture

## Architecture Quick Reference

### Services
1. **GAPI API** (port 8000) - External-facing gateway
2. **Pulse API** (port 8001) - Internal trading domain service
3. **Pulse Background** (no HTTP) - Async workers for order processing

### Critical Service Boundary Rule
**GAPI ↔ Pulse communication is HTTP-only (no shared code/direct function calls)**

This separation enables future multi-repo split.

### Tech Stack
- Framework: FastAPI (Python)
- Database: PostgreSQL
- Testing: pytest with unittest.mock
- Standards: OpenTelemetry tracing, W3C Trace Context, structured JSON logging

## Repository Structure

```
pulse-contracts/
├── service-groups/pulse-backend/services/  # API specs (api.md + openapi.yaml)
│   ├── gapi-api/                           # External gateway
│   ├── pulse-api/                          # Internal order API
│   └── pulse-background/                   # Async workers
├── schemas/                                # Shared schemas (common.yaml)
├── standards/                              # Cross-service standards
│   ├── tracing/, context/, logging/
│   ├── concurrency/, testing/, config/
│   └── ui-events/, user-actions/
└── product/                                # Product docs and specs
    ├── features/                           # Feature specifications
    └── architecture/                       # Architecture decisions
```

## Critical Standards to Follow

### 1. Distributed Tracing
**Reference**: `standards/tracing/README.md`

**ID Formats**:
- `trace_id`: `t{10-digit-timestamp}{12-hex-chars}` (e.g., `t1735228800a1b2c3d4e5f6`)
- `request_id`: `r{10-digit-timestamp}{12-hex-chars}` (e.g., `r1735228800f6e5d4c3b2a1`)
- `span_source`: `SERVICE:METHOD/path->SERVICE:METHOD/path`

**Required HTTP Headers**:
- Request: `X-Trace-Id`, `X-Request-Id`, `X-Trace-Source`, `X-Request-Source`
- Response: Always include `X-Request-Id` and `X-Trace-Id`

### 2. Request Context
**Reference**: `standards/context/README.md`

- Immutable dataclass containing: `trace_id`, `request_id`, `span_source`, `trace_source`, `request_source`
- Pass explicitly through all function calls (no global state)
- **Never include sensitive data** (no tokens, passwords, business data)

### 3. Structured Logging
**Reference**: `standards/logging/README.md`

- JSON format with required fields: `timestamp`, `level`, `logger`, `message`
- Auto-inject tracing fields when context passed to logger
- **Never log sensitive data** (credentials, auth headers, PII)

### 4. Concurrency Safety
**Reference**: `standards/concurrency/README.md`

**Patterns**:
- **Idempotent operations**: Use unique constraints for deduplication
- **Pessimistic locking**: `SELECT FOR UPDATE SKIP LOCKED` for exclusive processing
- **Optimistic locking**: First-come-first-served with version checks
- **Recalculate aggregates**: Never increment counters (query from source)
- **Timeout monitors**: Detect and recover from crashes

### 5. Testing Standards
**Reference**: `standards/testing/README.md`

**Organization**:
```
tests/
├── unit/          # <100ms, no external deps
└── integration/   # <5s, use TestClient
```

**Requirements**:
- AAA pattern (Arrange-Act-Assert)
- Minimum coverage: Success path + validation failures + edge cases
- Framework: pytest with unittest.mock
- CI/CD: Tests run on every commit/PR; build fails if tests fail

## API Contract Standards

### Error Response Format (Mandatory)
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": {}
  }
}
```

### Common Error Codes
- **4xx Validation**: `INVALID_REQUEST`, `INVALID_INSTRUMENT`, `INVALID_QUANTITY`, `INVALID_SPLIT_CONFIG`
- **4xx Business**: `INSUFFICIENT_FUNDS`, `MARKET_CLOSED`
- **4xx Conflict**: `DUPLICATE_ORDER_UNIQUE_KEY`, `UNAUTHORIZED`
- **5xx System**: `INTERNAL_ERROR`, `DATABASE_ERROR`, `BROKER_ERROR`

### Required Headers
- **Request**: `Content-Type: application/json`, optional: `X-Request-Id`, `X-Trace-Id`
- **Response**: Always include `X-Request-Id` and `X-Trace-Id`

## Naming Conventions

### Identifiers
- `order_id`: `ord_{random}` (e.g., `ord_abc123`)
- `order_unique_key`: User-provided unique key for deduplication

### Service Names
- Uppercase: `GAPI`, `PULSE`, `ORDER_SERVICE`
- Call format: `SERVICE:METHOD/path` (e.g., `GAPI:POST/api/orders`)

### Enums
- `OrderSide`: `BUY`, `SELL`
- `OrderQueueStatus`: `PENDING`, `IN_PROGRESS`, `DONE`, `SKIPPED`
- `OrderSliceStage`: `SCHEDULED`, `IN_PROGRESS`, `PROCESSED`

## File Navigation Guide

- **API Specs**: `service-groups/pulse-backend/services/{service-name}/api.md`
- **OpenAPI Specs**: `service-groups/pulse-backend/services/{service-name}/openapi.yaml`
- **Standards**: `standards/{topic}/README.md`
- **Product Specs**: `product/features/{feature-name}.md`
- **Architecture**: `product/architecture/{topic}.md`

## Development Rules

### Mandatory
1. **API-first**: Define all APIs in contracts BEFORE implementation
2. **Dual formats**: Both OpenAPI (YAML) and Markdown required; OpenAPI is source of truth
3. **Service boundaries**: Respect GAPI ↔ Pulse HTTP-only boundary (no code sharing)
4. **Explicit propagation**: Propagate tracing/context explicitly through all calls
5. **Standard errors**: Follow error response format
6. **JSON logs**: All logs must be valid JSON
7. **Immutable context**: Request context must be immutable
8. **Idempotency**: All concurrent operations must be idempotent

### Forbidden
1. Direct function calls across service boundaries (GAPI ↔ Pulse)
2. Shared in-memory state between services
3. Incrementing counters (recalculate from source instead)
4. Logging sensitive data (credentials, tokens, PII)
5. Global state in request context

## Common Workflows

### Adding a New API Endpoint
1. Define in OpenAPI spec (`openapi.yaml`)
2. Document in human-readable format (`api.md`)
3. Update common schemas if needed (`schemas/common.yaml`)
4. Implement with proper context propagation
5. Add unit tests (success + validation + edge cases)
6. Add integration test with TestClient

### Adding a New Standard
1. Create `standards/{topic}/README.md`
2. Document context, requirements, examples
3. Update relevant service documentation
4. Add to CLAUDE.md if widely applicable

## Key Product Context

**Vision**: Build a trading platform enabling sophisticated order execution strategies

**Core Feature**: Split Orders (v1)
- Divide large orders into N smaller slices
- Execute over specified duration
- Optional randomization of size/timing
- Reduce market impact and avoid detection

**API**: `POST /api/orders`
- Deduplication via `order_unique_key`
- Response: `202 Accepted` with `order_id`
- See: `product/features/split-orders.md`

## Success Metrics
- Order placement latency: <100ms (p95)
- System uptime: 99.9%
- Error rate: <0.1%

## Important References

When working on code, always check these standards first:
- Tracing: `standards/tracing/README.md`
- Context: `standards/context/README.md`
- Logging: `standards/logging/README.md`
- Concurrency: `standards/concurrency/README.md`
- Testing: `standards/testing/README.md`
