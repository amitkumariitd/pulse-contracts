# Pulse Frontend Service Group

**Repository**: `pulse-frontend`  
**Domain**: Trading platform web application

## Overview

The pulse-frontend service group contains the web application for the Pulse trading platform, providing a user interface for traders to:
- Create and manage split orders
- Monitor order execution status
- View order history and analytics
- Configure trading preferences

## Services

### 1. pulse-web

**Type**: Single Page Application (SPA)  
**Framework**: React 18 + TypeScript + Vite  
**Port**: 5173 (development), served via CDN/static hosting (production)  
**Purpose**: Web-based trading interface

**Responsibilities:**
- User interface for order creation and management
- Real-time order status monitoring
- Form validation and user input handling
- API client for GAPI backend
- Client-side state management
- Error handling and user feedback

**Contract**: [services/pulse-web/contract.md](services/pulse-web/contract.md)

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  pulse-frontend repo                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │              pulse-web (React SPA)                │  │
│  │                                                    │  │
│  │  Components:                                       │  │
│  │  ├─ Order Creation Form                           │  │
│  │  ├─ Order Status Dashboard                        │  │
│  │  ├─ Order History                                 │  │
│  │  └─ Settings                                      │  │
│  │                                                    │  │
│  │  API Client:                                       │  │
│  │  └─ HTTP calls to GAPI API (port 8000)           │  │
│  │                                                    │  │
│  └──────────────────────────────────────────────────┘  │
│                          │                              │
│                          │ HTTPS                        │
│                          ▼                              │
│              ┌────────────────────┐                     │
│              │   GAPI API         │                     │
│              │   (Backend)        │                     │
│              │   Port 8000        │                     │
│              └────────────────────┘                     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Technology Stack

### Core
- **React 18**: UI framework
- **TypeScript**: Type safety
- **Vite**: Build tool and dev server

### UI & Styling
- **Material-UI (MUI)**: Component library
- **CSS Modules**: Scoped styling

### State & Data
- **React Query (@tanstack/react-query)**: Server state management
- **Axios**: HTTP client

### Development
- **ESLint**: Code linting
- **TypeScript**: Type checking

---

## Backend Integration

### API Consumer
pulse-web consumes the GAPI API:
- **Base URL**: Configurable (env variable)
- **API Contract**: `contracts/service-groups/pulse-backend/services/gapi-api/`
- **Authentication**: JWT Bearer tokens
- **Tracing**: Propagates `X-Request-Id` and `X-Trace-Id` headers

### Contract Compliance
- **MUST** follow GAPI API contracts exactly
- **MUST NOT** invent endpoints or fields
- **MUST** use shared schemas from `contracts/schemas/`
- **MUST** handle all contract error codes

---

## Standards Compliance

### Tracing (`contracts/standards/tracing/`)
- Include `X-Request-Id` and `X-Trace-Id` in all API requests
- Capture tracing headers from responses
- Display tracing IDs in error messages for debugging
- Log tracing context for support

### Context Propagation (`contracts/standards/context/`)
- Maintain request context across API calls
- Propagate tracing headers consistently
- Include context in error reporting

### Logging (`contracts/standards/logging/`)
- Structured logging for client-side errors
- Include tracing information in logs
- Log API errors with request/response details
- Use consistent log levels (ERROR, WARN, INFO, DEBUG)

### Testing (`contracts/standards/testing/`)
- Unit tests for components
- Integration tests for API client
- Mock API responses using contract examples
- Validate request/response types match contracts
- Test error handling for all contract error codes

### Configuration (`contracts/standards/config/`)
- Environment-based configuration
- API base URL configurable
- Feature flags for gradual rollout
- Separate dev/staging/production configs

---

## Development Workflow

### 1. Contract-First Development
- Check GAPI API contracts before implementing features
- Use contract schemas for TypeScript types
- Never invent API fields or endpoints

### 2. Mock-First UI Development
- Build UI with mock APIs first (`src/api/mock/`)
- Get UI working and approved
- Swap to real API calls after validation

### 3. Type Safety
- Define types in `src/types/` based on contracts
- Use TypeScript strictly (no `any`)
- Validate API responses match expected types

### 4. Error Handling
- Handle all contract error codes
- Display user-friendly error messages
- Show tracing IDs for support
- Log errors with full context

---

## Deployment

### Development
```bash
npm run dev  # Start dev server on port 5173
```

### Production Build
```bash
npm run build   # Build static assets to dist/
npm run preview # Preview production build
```

### Hosting Options
- **Static Hosting**: Netlify, Vercel, AWS S3 + CloudFront
- **CDN**: Serve static assets via CDN
- **Docker**: Containerized deployment with nginx

---

## Future Enhancements

### Phase 2: Real-time Updates
- WebSocket connection for live order status
- Real-time notifications
- Live market data integration

### Phase 3: Advanced Features
- Order analytics and reporting
- Multi-account support
- Advanced charting
- Mobile responsive design

### Phase 4: Mobile App
- React Native mobile app
- Shared API client logic
- Native mobile features

---

## Related Documentation

- **GAPI API Contract**: `contracts/service-groups/pulse-backend/services/gapi-api/`
- **Common Schemas**: `contracts/schemas/`
- **Standards**: `contracts/standards/`
- **Product Overview**: `contracts/product/overview.md`

