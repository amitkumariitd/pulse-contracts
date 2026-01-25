# Split Orders Feature

## Overview

The Split Orders feature allows traders to divide a large order into multiple smaller slices that are executed over a specified time period. This helps minimize market impact and avoid detection by other market participants.

## User Story

**As a** trader  
**I want to** split a large order into smaller slices executed over time  
**So that** I can minimize market impact and get better execution prices

## Use Cases

### Use Case 1: Large Order Execution

**Scenario**: A trader wants to buy 1000 shares of RELIANCE but doesn't want to move the market.

**Solution**: Split the order into 10 slices of 100 shares each, executed over 60 minutes.

**Benefit**: Smaller orders have less market impact, potentially resulting in better average execution price.

### Use Case 2: Time-Weighted Average Price (TWAP)

**Scenario**: A trader wants to achieve a price close to the time-weighted average price over a trading session.

**Solution**: Split the order into many small slices executed at regular intervals throughout the session.

**Benefit**: Execution price approximates the average market price over the period.

### Use Case 3: Stealth Trading

**Scenario**: A trader wants to accumulate a position without alerting other market participants.

**Solution**: Split the order with randomization enabled to vary slice sizes and timing.

**Benefit**: Unpredictable order pattern makes it harder for others to detect and front-run.

## Feature Specification

### Input Parameters

| Parameter | Type | Required | Description | Constraints |
|-----------|------|----------|-------------|-------------|
| `instrument` | string | Yes | Trading symbol | Format: `EXCHANGE:SYMBOL` (e.g., `NSE:RELIANCE`) |
| `side` | enum | Yes | Order side | `BUY` or `SELL` |
| `total_quantity` | integer | Yes | Total shares to trade | Min: 1, Must be >= `num_splits` |
| `num_splits` | integer | Yes | Number of slices | Min: 2, Max: 100 |
| `duration_minutes` | integer | Yes | Total duration | Min: 1, Max: 1440 (24 hours) |
| `randomize` | boolean | No | Apply randomization | Default: `true` |
| `order_unique_key` | string | Yes | Deduplication key | Unique per order |

### Output

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | string | Unique identifier for the parent order |
| `order_unique_key` | string | Echoed from request |

**Response Code**: `202 Accepted` (order accepted for async processing)

### Behavior

1. **Validation**: System validates all input parameters
2. **Deduplication**: System checks `order_unique_key` to prevent duplicates
3. **Order Creation**: System creates parent order with status `PENDING`
4. **Async Processing**: Background worker picks up order and splits it
5. **Slice Creation**: System creates N child slices with calculated quantities and times
6. **Scheduling**: System schedules each slice for execution at its designated time

### Splitting Algorithm

**Quantity Distribution:**
- If `randomize = false`: Equal distribution (total_quantity / num_splits)
- If `randomize = true`: Random distribution with ±20% variance

**Time Distribution:**
- If `randomize = false`: Equal intervals (duration_minutes / num_splits)
- If `randomize = true`: Random intervals with ±20% variance

**Constraints:**
- All slices must fit within the duration window
- Sum of slice quantities must equal total_quantity
- Each slice must have quantity >= 1

See technical spec: `../../doc/requirements/002.split_order_feature.md` (in main repo)

## User Experience

### API Request

```bash
POST /api/orders
Authorization: Bearer <token>
Content-Type: application/json

{
  "order_unique_key": "my-order-123",
  "instrument": "NSE:RELIANCE",
  "side": "BUY",
  "total_quantity": 100,
  "split_config": {
    "num_splits": 5,
    "duration_minutes": 60,
    "randomize": true
  }
}
```

### API Response

```json
HTTP/1.1 202 Accepted
X-Request-Id: 550e8400-e29b-41d4-a716-446655440000
X-Trace-Id: 7c9e6679-7425-40de-944b-e07fc1f90ae7

{
  "order_id": "ord_abc123",
  "order_unique_key": "my-order-123"
}
```

### What Happens Next

1. Order is created with status `PENDING`
2. Background worker picks up order within ~5 seconds
3. Order is split into 5 slices
4. Slices are scheduled for execution over 60 minutes
5. Each slice is executed at its scheduled time
6. Order status updates to `IN_PROGRESS` → `DONE`

## Error Handling

### Validation Errors (400 Bad Request)

- Missing required fields
- Invalid instrument format
- Invalid quantity (must be > 0)
- Invalid split config (num_splits, duration_minutes out of range)
- total_quantity < num_splits (can't split into more slices than shares)

### Duplicate Order (409 Conflict)

- Same `order_unique_key` used with different order data
- Returns existing order_id if same key used with identical data

### Business Errors (422 Unprocessable Entity)

- Insufficient funds
- Market closed
- Instrument not tradable

### System Errors (500 Internal Server Error)

- Database errors
- Unexpected system failures

## Success Criteria

### Functional
- ✅ Order is split into correct number of slices
- ✅ Slice quantities sum to total quantity
- ✅ Slices are scheduled within duration window
- ✅ Randomization works as expected
- ✅ Deduplication prevents duplicate orders

### Non-Functional
- ✅ API responds within 100ms (p95)
- ✅ Order splitting completes within 5 seconds
- ✅ System handles 100 concurrent orders
- ✅ No data loss or corruption

## Future Enhancements

### Phase 2
- Real-time order status tracking
- Cancel pending slices
- Modify split configuration

### Phase 3
- Advanced splitting strategies (VWAP, Iceberg)
- Conditional splitting (price-based triggers)
- Multi-instrument orders

## Related Documentation

- API Contract: [../../service-groups/pulse-backend/services/gapi-api/api.md](../../service-groups/pulse-backend/services/gapi-api/api.md)
- Technical Spec: `../../doc/requirements/002.split_order_feature.md` (in main repo)
- Architecture: [../architecture/service-boundaries.md](../architecture/service-boundaries.md)

