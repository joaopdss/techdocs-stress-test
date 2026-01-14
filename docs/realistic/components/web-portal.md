# Web Portal

## Description

The Web Portal is TechCorp's main frontend application, offering a complete e-commerce experience for customers. This single-page application (SPA) allows product browsing, cart management, checkout, and order tracking.

The interface was developed following responsive design principles, ensuring a consistent experience across desktop, tablet, and mobile devices. The portal implements Server-Side Rendering (SSR) for SEO optimization and initial load performance.

The frontend architecture uses a micro-frontends approach, allowing different teams to develop and deploy features independently. Each module can be updated without impacting the rest of the application.

## Owners

- **Team:** Customer Experience Frontend
- **Tech Lead:** Patricia Gomes
- **Slack:** #cx-web-portal

## Technology Stack

- Framework: Next.js 14
- Language: TypeScript 5.3
- UI Library: React 18
- State Management: Zustand
- Styling: Tailwind CSS + shadcn/ui
- Testing: Jest + React Testing Library + Playwright

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `NEXT_PUBLIC_API_URL` | API base URL | - |
| `NEXT_PUBLIC_CDN_URL` | Assets CDN URL | - |
| `NEXT_PUBLIC_GA_ID` | Google Analytics ID | - |
| `NEXT_PUBLIC_SENTRY_DSN` | Sentry DSN | - |
| `API_SECRET_KEY` | Key for server-side calls | - |
| `REVALIDATE_INTERVAL` | ISR revalidation interval (seconds) | `60` |

### Build Configuration

```javascript
// next.config.js
module.exports = {
  images: {
    domains: ['cdn.techcorp.com'],
    formats: ['image/avif', 'image/webp'],
  },
  i18n: {
    locales: ['en-US', 'pt-BR', 'es-ES'],
    defaultLocale: 'en-US',
  },
  experimental: {
    serverActions: true,
  },
}
```

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/web-portal.git
cd web-portal

# Install dependencies
npm install

# Configure environment variables
cp .env.example .env.local
# Edit .env.local with development values

# Start in development mode
npm run dev

# Access application
open http://localhost:3000
```

### Run Tests

```bash
# Unit tests
npm run test

# Integration tests
npm run test:integration

# E2E tests
npm run test:e2e

# Coverage
npm run test:coverage
```

## Directory Structure

```
src/
├── app/                    # App Router (Next.js 14)
│   ├── (auth)/            # Authenticated routes group
│   ├── (public)/          # Public routes group
│   ├── api/               # API Routes
│   └── layout.tsx         # Root layout
├── components/
│   ├── ui/                # Base components (shadcn)
│   ├── features/          # Feature components
│   └── layouts/           # Layout components
├── hooks/                 # Custom hooks
├── lib/                   # Utilities and configurations
├── services/              # API calls
├── stores/                # Zustand stores
└── types/                 # TypeScript types
```

## Main Features

### 1. Product Catalog

- Listing with filters and sorting
- Full-text search with autocomplete
- Product pages with image gallery
- Ratings and reviews

### 2. Shopping Cart

- Local persistence (localStorage)
- Sync with backend on login
- Real-time shipping calculation
- Coupon application

### 3. Checkout

- Step-by-step checkout (address, payment, confirmation)
- Multiple payment methods
- Payment gateway integration via secure iframe
- Real-time form validation

### 4. Customer Area

- Order history
- Address management
- Account preferences
- Delivery tracking

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/web-portal
- **Sentry:** https://sentry.techcorp.internal/web-portal
- **Web Vitals:** https://analytics.techcorp.internal/web-vitals

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `LCP` | Largest Contentful Paint | > 2.5s |
| `FID` | First Input Delay | > 100ms |
| `CLS` | Cumulative Layout Shift | > 0.1 |
| `TTFB` | Time to First Byte | > 800ms |
| `error_rate` | JavaScript error rate | > 1% |

### Configured Alerts

- **WebVitalsLCPHigh:** LCP above 2.5s for 10 minutes
- **WebPortalErrorRateHigh:** JS error rate above 1%
- **WebPortalAPILatencyHigh:** API latency above 2s

## Troubleshooting

### Issue: Page loading slowly

**Cause:** Bundle too large, non-optimized images, or slow API.

**Solution:**
1. Analyze bundle: `npm run analyze`
2. Check component lazy loading
3. Confirm images use next/image
4. Check API latency in Network tab

### Issue: Hydration errors

**Cause:** Difference between server and client render.

**Solution:**
1. Check usage of `window` or `document` without verification
2. Use `useEffect` for client-only code
3. Check consistent initial states

### Issue: State not persisting between pages

**Cause:** Store not configured correctly or SSR interfering.

**Solution:**
1. Check Zustand persist configuration
2. Use `skipHydration` if necessary
3. Check if provider is being used correctly

### Issue: Checkout freezing

**Cause:** Validation error or failure in communication with payment-service.

**Solution:**
1. Check console for validation errors
2. Check network for request failures
3. Verify payment token hasn't expired

## Related Links

- [API Gateway](api-gateway.md) - Request backend
- [Auth Service](auth-service.md) - User authentication
- [Mobile App](mobile-app.md) - Mobile version
- [Admin Dashboard](admin-dashboard.md) - Administrative panel
