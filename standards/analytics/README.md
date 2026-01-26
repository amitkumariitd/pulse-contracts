# Analytics Standard

**Domain**: Analytics Tracking  
**Purpose**: Provider-agnostic analytics interface for tracking user behavior

---

## Overview

This standard defines a **provider-agnostic analytics interface** that:
- Tracks user behavior and interactions
- Correlates analytics events with distributed tracing
- Supports multiple analytics providers (Google Analytics, PostHog, custom backend)
- Automatically includes tracing context in all events

---

## Analytics Service Interface

All analytics providers MUST implement this interface:

```typescript
interface IAnalyticsService {
  /**
   * Initialize the analytics service
   */
  init(): void;

  /**
   * Track an event
   * @param event - Event name (e.g., 'order_placed', 'button_clicked')
   * @param properties - Event properties/metadata
   */
  track(event: string, properties?: Record<string, any>): void;

  /**
   * Identify a user
   * @param userId - User ID
   * @param properties - User properties (email, name, etc.)
   */
  identify(userId: string, properties?: UserProperties): void;

  /**
   * Track a page view
   * @param page - Page path
   * @param properties - Additional properties
   */
  page(page: string, properties?: Record<string, any>): void;

  /**
   * Reset user identity (on logout)
   */
  reset(): void;
}
```

---

## Event Structure

### AnalyticsEvent

```typescript
interface AnalyticsEvent {
  /** Event name (e.g., 'order_placed', 'button_clicked') */
  event: string;
  
  /** Event properties/metadata */
  properties?: Record<string, any>;
  
  /** User ID (if available) */
  userId?: string;
  
  /** Session ID */
  sessionId?: string;
  
  /** Timestamp (auto-generated if not provided) */
  timestamp?: string;
}
```

### UserProperties

```typescript
interface UserProperties {
  userId: string;
  email?: string;
  name?: string;
  [key: string]: any;
}
```

---

## Integration with Tracing

All analytics events MUST automatically include tracing context:

```typescript
{
  event: 'order_placed',
  properties: {
    // User data
    order_id: '123',
    instrument: 'NSE:RELIANCE',
    
    // Automatically added from tracing context
    trace_id: 't1735228800a1b2c3d4e5f6',
    request_id: 'r1735228800f6e5d4c3b2a1',
    session_id: 'session_1735228800_abc',
    timestamp: '2024-01-26T10:30:00.000Z',
    url: 'http://localhost:5173/orders',
    referrer: 'http://localhost:5173/',
  }
}
```

**Benefits:**
- Link analytics events to API calls via `trace_id`
- Correlate user actions with backend logs
- Debug issues by following trace from UI → API → DB

**Reference:** `contracts/standards/tracing/README.md`

---

## Common Events

### Order Management
- `order_placed` - Order created
- `order_cancelled` - Order cancelled
- `order_modified` - Order modified
- `order_retry` - Order retry after error

### User Interactions
- `button_click` - Button clicked
- `form_submit` - Form submitted
- `form_field_change` - Form field changed
- `error_interaction` - Error retry/dismiss

### Navigation
- `page_view` - Page viewed
- `navigation` - Route changed

---

## Supported Providers

### 1. Console (Development)
- Logs events to browser console
- Perfect for development
- No configuration required

### 2. Google Analytics 4
- Google Analytics 4 (GA4)
- Requires: GA tracking ID

### 3. PostHog
- PostHog (open-source analytics)
- Requires: PostHog API key and host

### 4. Custom Backend
- Send events to custom backend API
- Requires: Backend endpoint

---

## Standards Compliance

### Tracing Standard
**Reference:** `contracts/standards/tracing/README.md`

- MUST include `trace_id` in all events
- MUST include `request_id` in all events
- MUST include `session_id` in all events
- MUST include `timestamp` in all events

### Privacy
- MUST NOT track PII (passwords, credit cards, etc.)
- MUST respect user privacy settings
- MUST allow opt-out

### Performance
- MUST NOT block UI rendering
- MUST send events asynchronously
- MUST fail silently (never break user experience)

---

## Related Standards

- `contracts/standards/tracing/README.md` - Distributed tracing standard
- `contracts/standards/user-actions/README.md` - User action context standard

