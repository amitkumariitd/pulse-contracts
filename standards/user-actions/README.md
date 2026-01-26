# User Action Context Standard

**Domain**: User Action Context Propagation  
**Purpose**: Pass UI action context to backend APIs via HTTP headers

---

## Overview

When a user triggers an API call (clicks a button, submits a form), the frontend MUST pass **user action context** to the backend via HTTP headers.

This enables the backend to:
1. **Log which UI action triggered the request** - Better debugging
2. **Apply action-specific logic** - Different rate limits, validation rules
3. **Detect suspicious patterns** - Security monitoring
4. **Track user behavior** - Analytics and audit trails
5. **Optimize processing** - Priority queues based on action type

---

## HTTP Headers

### Header Names

| Header Name       | Description                          | Required |
|-------------------|--------------------------------------|----------|
| X-User-Action     | Action identifier                    | Yes      |
| X-User-Component  | Component that triggered the action  | No       |
| X-User-Page       | Current page/route                   | No       |
| X-User-Metadata   | Additional metadata (JSON string)    | No       |

### Example

```http
POST /api/orders HTTP/1.1
Host: api.pulse.example.com
Content-Type: application/json

# Tracing headers (from contracts/standards/tracing/README.md)
X-Trace-Id: t1735228800a1b2c3d4e5f6
X-Trace-Source: PULSE-WEB:POST/api/orders
X-Request-Id: r1735228800f6e5d4c3b2a1
X-Request-Source: PULSE-WEB:POST/api/orders
X-Span-Source: PULSE-WEB:POST/api/orders

# User action headers (this standard)
X-User-Action: place_order
X-User-Component: OrderForm
X-User-Page: /trading/orders
X-User-Metadata: {"instrument":"NSE:RELIANCE","side":"BUY"}
```

---

## UserActionContext Type

```typescript
interface UserActionContext {
  /** Action identifier (e.g., 'place_order', 'cancel_order', 'retry_order') */
  action: string;
  
  /** Component that triggered the action */
  component?: string;
  
  /** Current page/route */
  page?: string;
  
  /** Additional metadata */
  metadata?: Record<string, any>;
}
```

---

## Standard Action Names

### Order Management

| Action              | When to Use                    |
|---------------------|--------------------------------|
| `place_order`       | Initial order placement        |
| `retry_order`       | Retry after error              |
| `cancel_order`      | Cancel single order            |
| `cancel_all_orders` | Cancel multiple orders         |
| `modify_order`      | Modify existing order          |

### Data Fetching

| Action                | When to Use                    |
|-----------------------|--------------------------------|
| `refresh_orders`      | Manual refresh button          |
| `auto_refresh_orders` | Auto-refresh/polling           |
| `load_more_orders`    | Pagination                     |
| `search_orders`       | Search/filter                  |

### Settings

| Action            | When to Use                    |
|-------------------|--------------------------------|
| `update_settings` | Change user settings           |
| `reset_settings`  | Reset to defaults           |

### Authentication

| Action          | When to Use                    |
|-----------------|--------------------------------|
| `login`         | User login                     |
| `logout`        | User logout                    |
| `refresh_token` | Token refresh                  |

---

## Integration with Tracing

User action context is sent **alongside** tracing headers:

```
Complete Request Headers:
┌─────────────────────────────────────────────────────────┐
│ Tracing Headers (contracts/standards/tracing/README.md) │
├─────────────────────────────────────────────────────────┤
│ X-Trace-Id: t1735228800a1b2c3d4e5f6                     │
│ X-Request-Id: r1735228800f6e5d4c3b2a1                   │
│ X-Trace-Source: PULSE-WEB:POST/api/orders               │
│ X-Request-Source: PULSE-WEB:POST/api/orders             │
│ X-Span-Source: PULSE-WEB:POST/api/orders                │
├─────────────────────────────────────────────────────────┤
│ User Action Headers (this standard)                     │
├─────────────────────────────────────────────────────────┤
│ X-User-Action: place_order                              │
│ X-User-Component: OrderForm                             │
│ X-User-Page: /trading/orders                            │
│ X-User-Metadata: {"instrument":"NSE:RELIANCE"}          │
└─────────────────────────────────────────────────────────┘
```

**Result:** Backend knows:
- **What**: place_order (user action)
- **Where**: OrderForm on /trading/orders (UI location)
- **When**: t1735228800... (trace_id timestamp)
- **Context**: instrument=NSE:RELIANCE (metadata)

---

## Backend Usage

### Extract Headers (Python Example)

```python
from fastapi import Request

def extract_action_context(request: Request) -> dict:
    """Extract user action context from headers"""
    return {
        'action': request.headers.get('X-User-Action'),
        'component': request.headers.get('X-User-Component'),
        'page': request.headers.get('X-User-Page'),
        'metadata': json.loads(request.headers.get('X-User-Metadata', '{}')),
    }
```

### Use in Endpoint

```python
@app.post("/api/orders")
async def create_order(
    request: Request,
    order_data: CreateOrderRequest,
):
    action_context = extract_action_context(request)
    
    # Log with action context
    logger.info(
        "Order creation request",
        trace_id=request.headers.get('X-Trace-Id'),
        user_action=action_context['action'],
        component=action_context['component'],
    )
    
    # Apply action-specific logic
    if action_context['action'] == 'retry_order':
        # More lenient validation for retries
        pass
    
    # Process order...
```

---

## Standards Compliance

### Tracing Standard
**Reference:** `contracts/standards/tracing/README.md`

- MUST send user action headers alongside tracing headers
- MUST include `trace_id` to correlate user actions with API calls
- MUST NOT replace or interfere with tracing headers

### Privacy
- MUST NOT include PII in metadata
- MUST NOT include sensitive data (passwords, tokens, etc.)

### Performance
- MUST NOT add significant overhead to requests
- Metadata SHOULD be small (< 1KB)

---

## Related Standards

- `contracts/standards/tracing/README.md` - Distributed tracing standard
- `contracts/standards/analytics/README.md` - Analytics tracking standard

