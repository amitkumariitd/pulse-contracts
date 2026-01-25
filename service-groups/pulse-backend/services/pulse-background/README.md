# Pulse Background Service

**Type**: Background worker (no HTTP endpoints)  
**Purpose**: Async order processing and execution

## Overview

The pulse-background service is a background worker that processes orders asynchronously. It does not expose HTTP endpoints but shares the same codebase and database with pulse-api.

## Responsibilities

### 1. Order Splitting

Processes PENDING orders and splits them into child order slices.

**Input**: Orders with `order_queue_status = 'PENDING'`  
**Output**: Order slices with scheduled execution times  
**Status**: Updates order to `IN_PROGRESS` or `DONE`

**Algorithm:**
- Divides total quantity into N slices
- Calculates execution times over duration
- Applies randomization if configured
- Creates order_slices records in database

### 2. Order Slice Execution

Executes scheduled order slices at their designated times.

**Input**: Order slices with `stage = 'SCHEDULED'` and `scheduled_at <= NOW()`  
**Output**: Broker orders placed via broker API  
**Status**: Updates slice to `IN_PROGRESS` → `PROCESSED`

**Process:**
- Polls for due slices
- Places orders with broker (Zerodha)
- Records broker order ID
- Updates slice status

### 3. Timeout Monitoring

Monitors and recovers from stuck or failed operations.

**Monitors:**
- Orders stuck in `IN_PROGRESS` for too long
- Slices stuck in `IN_PROGRESS` for too long
- Failed broker API calls

**Actions:**
- Marks timed-out operations as failed
- Logs errors for investigation
- Updates status to allow retry or skip

## Configuration

Background workers are configured via environment variables:

```bash
# Polling intervals
ORDER_SPLITTER_POLL_INTERVAL_SECONDS=5
SLICE_EXECUTOR_POLL_INTERVAL_SECONDS=1

# Timeout thresholds
ORDER_SPLIT_TIMEOUT_MINUTES=5
SLICE_EXECUTION_TIMEOUT_MINUTES=2

# Broker configuration
BROKER_TYPE=mock  # or 'zerodha'
ZERODHA_API_KEY=your_api_key
ZERODHA_ACCESS_TOKEN=your_access_token
```

See: `doc/guides/config.md` for full configuration reference.

## Workers

### OrderSplitter

**File**: `pulse/workers/order_splitter.py`  
**Frequency**: Every 5 seconds (configurable)

Picks up PENDING orders and splits them into slices.

### SliceExecutor

**File**: `pulse/workers/slice_executor.py`  
**Frequency**: Every 1 second (configurable)

Executes scheduled slices by placing broker orders.

### TimeoutMonitor

**File**: `pulse/workers/timeout_monitor.py`  
**Frequency**: Every 30 seconds (configurable)

Monitors and recovers from timeouts.

## Deployment

### Standalone

```bash
python -m pulse.background
```

### Docker

```bash
docker-compose up pulse-background
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pulse-background
spec:
  replicas: 1  # Single instance for now
  template:
    spec:
      containers:
      - name: pulse-background
        image: pulse-backend:latest
        command: ["python", "-m", "pulse.background"]
```

## Concurrency Safety

All background workers follow strict concurrency safety rules:

- **Pessimistic locking**: Use `SELECT ... FOR UPDATE` to claim work
- **Idempotency**: All operations are idempotent
- **Atomic transitions**: State changes are atomic
- **No incremental updates**: Always recalculate from source of truth

See: `doc/guides/concurrency.md` for detailed patterns.

## Monitoring

### Logs

All workers log structured JSON with:
- `request_id` - Unique request identifier
- `trace_id` - Distributed trace identifier
- `order_id` - Order being processed
- `worker` - Worker name (splitter, executor, monitor)

### Metrics (Future)

- Orders processed per minute
- Slices executed per minute
- Average split time
- Average execution time
- Error rates

## Testing

### Unit Tests

```bash
pytest tests/unit/workers/
```

### Integration Tests

```bash
pytest tests/integration/workers/
```

See: `doc/guides/testing.md` for testing standards.

## Related Documentation

- Order splitting algorithm: `doc/requirements/002.split_order_feature.md`
- Order slice execution: `doc/requirements/003.order_slice_execution_feature.md`
- Concurrency safety: `doc/guides/concurrency.md`
- Broker integration: `doc/guides/zerodha_integration.md`

