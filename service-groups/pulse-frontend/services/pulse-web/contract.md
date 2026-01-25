# Pulse Web Application Contract

**Service**: pulse-web  
**Type**: Single Page Application (React)  
**Repository**: pulse-frontend

---

## Overview

pulse-web is the web-based user interface for the Pulse trading platform. It provides traders with tools to create split orders, monitor execution, and manage their trading activity.

---

## API Integration

### Backend API
- **Service**: GAPI API (pulse-backend)
- **Contract**: `contracts/service-groups/pulse-backend/services/gapi-api/`
- **Base URL**: Configurable via environment variable
- **Authentication**: JWT Bearer tokens

### Consumed Endpoints

#### Order Management
- `POST /api/orders` - Create split order
- `GET /api/orders` - List orders (future)
- `GET /api/orders/{order_id}` - Get order details (future)
- `DELETE /api/orders/{order_id}` - Cancel order (future)

#### Health Check
- `GET /health` - Backend health status

---

## Request Standards

### Headers (All Requests)

**Required:**
- `Content-Type: application/json`
- `Authorization: Bearer <jwt_token>`

**Optional (Tracing):**
- `X-Request-Id: <request_id>` - Auto-generated if not provided
- `X-Trace-Id: <trace_id>` - Auto-generated if not provided

### Tracing Implementation

Following `contracts/standards/tracing/`:

```typescript
// Generate request_id
function generateRequestId(): string {
  const timestamp = Math.floor(Date.now() / 1000);
  const randomHex = Array.from(crypto.getRandomValues(new Uint8Array(6)))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
  return `r${timestamp}${randomHex}`;
}

// Generate trace_id
function generateTraceId(): string {
  const timestamp = Math.floor(Date.now() / 1000);
  const randomHex = Array.from(crypto.getRandomValues(new Uint8Array(6)))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
  return `t${timestamp}${randomHex}`;
}

// Add to axios interceptor
axios.interceptors.request.use((config) => {
  if (!config.headers['X-Request-Id']) {
    config.headers['X-Request-Id'] = generateRequestId();
  }
  if (!config.headers['X-Trace-Id']) {
    config.headers['X-Trace-Id'] = generateTraceId();
  }
  return config;
});
```

---

## Response Handling

### Success Responses
- Parse response data according to contract schemas
- Extract tracing headers (`X-Request-Id`, `X-Trace-Id`)
- Update UI state with response data

### Error Responses

All errors follow contract format from `contracts/schemas/common.md`:

```typescript
interface ApiError {
  error: {
    code: string;
    message: string;
    details: Record<string, any>;
  };
}
```

**Error Handling Pattern:**

```typescript
try {
  const response = await apiClient.post('/api/orders', orderData);
  return response.data;
} catch (error) {
  if (axios.isAxiosError(error) && error.response) {
    const apiError = error.response.data as ApiError;
    const requestId = error.response.headers['x-request-id'];
    const traceId = error.response.headers['x-trace-id'];
    
    // Log error with tracing context
    console.error('API Error:', {
      code: apiError.error.code,
      message: apiError.error.message,
      requestId,
      traceId,
    });
    
    // Display user-friendly message
    showErrorMessage(apiError.error.code, requestId, traceId);
  }
  throw error;
}
```

---

## Error Code Handling

From `contracts/schemas/common.md`:

### Validation Errors (4xx)
- `INVALID_REQUEST` → "Please check your input and try again"
- `INVALID_INSTRUMENT` → "Invalid instrument format. Use EXCHANGE:SYMBOL (e.g., NSE:RELIANCE)"
- `INVALID_QUANTITY` → "Quantity must be greater than 0 and at least equal to number of splits"
- `INVALID_SPLIT_CONFIG` → "Invalid split configuration. Check num_splits (2-100) and duration (1-1440 minutes)"
- `DUPLICATE_ORDER_UNIQUE_KEY` → "This order has already been submitted with different parameters"
- `UNAUTHORIZED` → "Please log in to continue"

### Business Errors (4xx)
- `INSUFFICIENT_FUNDS` → "Insufficient funds to place this order"
- `MARKET_CLOSED` → "Market is currently closed. Please try again during market hours"

### System Errors (5xx)
- `INTERNAL_ERROR` → "An unexpected error occurred. Please try again"
- `DATABASE_ERROR` → "Database error. Please try again later"
- `BROKER_ERROR` → "Broker API error. Please contact support"

**Always include tracing IDs in error messages for support:**
```
Error: Market is currently closed
Request ID: r1735228800f6e5d4c3b2a1
Trace ID: t1735228800a1b2c3d4e5f6
```

---

## Type Definitions

### Request Types

```typescript
// From contracts/schemas/common.yaml
interface SplitConfig {
  num_splits: number;      // 2-100
  duration_minutes: number; // 1-1440
  randomize?: boolean;      // default: true
}

interface CreateOrderRequest {
  order_unique_key: string;
  instrument: string;       // Format: EXCHANGE:SYMBOL
  side: 'BUY' | 'SELL';
  total_quantity: number;
  split_config: SplitConfig;
}
```

### Response Types

```typescript
interface CreateOrderResponse {
  order_id: string;
  order_unique_key: string;
}
```

---

## Validation Rules

### Client-Side Validation

Before sending requests, validate:

1. **order_unique_key**: Non-empty string
2. **instrument**: Matches pattern `^(NSE|BSE):[A-Z0-9]+$`
3. **side**: Must be 'BUY' or 'SELL'
4. **total_quantity**: 
   - Must be > 0
   - Must be >= split_config.num_splits
5. **split_config.num_splits**: 2-100
6. **split_config.duration_minutes**: 1-1440

---

## Logging Standards

Following `contracts/standards/logging/`:

### Client-Side Logging

```typescript
interface LogEntry {
  timestamp: string;      // ISO 8601
  level: 'ERROR' | 'WARN' | 'INFO' | 'DEBUG';
  message: string;
  trace_id?: string;
  request_id?: string;
  data?: Record<string, any>;
}

// Example
logger.error('Order creation failed', {
  trace_id: 't1735228800a1b2c3d4e5f6',
  request_id: 'r1735228800f6e5d4c3b2a1',
  error_code: 'INVALID_QUANTITY',
  data: { quantity: 5, num_splits: 10 }
});
```

---

## Testing Standards

Following `contracts/standards/testing/`:

### Unit Tests
- Test components in isolation
- Mock API responses using contract examples
- Test error handling for all error codes
- Validate form validation logic

### Integration Tests
- Test API client with mock server
- Verify request format matches contracts
- Verify response parsing matches contracts
- Test tracing header propagation

### Contract Validation Tests
- Validate all request types match OpenAPI schemas
- Validate all response types match OpenAPI schemas
- Test error response format compliance

---

## Configuration Standards

Following `contracts/standards/config/`:

### Environment Variables

```typescript
interface AppConfig {
  API_BASE_URL: string;        // Backend API URL
  ENVIRONMENT: 'dev' | 'staging' | 'production';
  ENABLE_MOCK_API: boolean;    // Use mock APIs
  LOG_LEVEL: 'ERROR' | 'WARN' | 'INFO' | 'DEBUG';
}
```

### Configuration Files
- `.env.development` - Development config
- `.env.staging` - Staging config
- `.env.production` - Production config

---

## Security

### Authentication
- Store JWT tokens securely (httpOnly cookies preferred)
- Include Bearer token in Authorization header
- Handle 401 Unauthorized by redirecting to login
- Refresh tokens before expiry

### Data Protection
- Never log sensitive data (tokens, passwords)
- Sanitize user input before display
- Use HTTPS for all API calls
- Implement CSRF protection

---

## Performance

### Optimization
- Lazy load routes and components
- Debounce form inputs
- Cache API responses with React Query
- Minimize bundle size

### Monitoring
- Track API response times
- Monitor error rates
- Log performance metrics
- Track user interactions

---

## Related Documentation

- **GAPI API**: `contracts/service-groups/pulse-backend/services/gapi-api/api.md`
- **Common Schemas**: `contracts/schemas/common.md`
- **Tracing Standard**: `contracts/standards/tracing/README.md`
- **Logging Standard**: `contracts/standards/logging/README.md`
- **Testing Standard**: `contracts/standards/testing/README.md`
- **Config Standard**: `contracts/standards/config/README.md`

