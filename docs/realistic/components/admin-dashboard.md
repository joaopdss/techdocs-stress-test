# Admin Dashboard

## Description

The Admin Dashboard is TechCorp's administrative interface, used by internal teams to manage day-to-day operations. This web application offers functionalities for order management, products, users, reports, and system configurations.

The dashboard was designed to support different access profiles, from support agents to system administrators. Each profile has a specific set of permissions that determine which functionalities are available.

The interface prioritizes operational efficiency, with keyboard shortcuts, quick search, and batch actions for the most frequent operations. The dashboard also includes configurable widgets for real-time KPI monitoring.

## Owners

- **Team:** Internal Tools
- **Tech Lead:** Diego Martins
- **Slack:** #internal-admin

## Technology Stack

- Framework: React 18 + Vite
- Language: TypeScript 5.3
- UI Library: Ant Design 5
- State Management: React Query + Zustand
- Charts: Recharts
- Tables: TanStack Table

## Access Profiles

| Profile | Description | Main Permissions |
|---------|-------------|------------------|
| `support` | Customer service | View orders, users |
| `operations` | Fulfillment operations | Manage orders, inventory |
| `finance` | Finance team | Reports, payments |
| `product_manager` | Catalog management | Products, categories, prices |
| `admin` | System administrator | Full access |

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `VITE_API_URL` | API base URL | - |
| `VITE_AUTH_URL` | auth-service URL | - |
| `VITE_SENTRY_DSN` | Sentry DSN | - |
| `VITE_FEATURE_FLAGS_URL` | Feature flags service URL | - |
| `VITE_WEBSOCKET_URL` | WebSocket URL for real-time | - |

### Build Configuration

```javascript
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'antd'],
          charts: ['recharts'],
        },
      },
    },
  },
})
```

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/admin-dashboard.git
cd admin-dashboard

# Install dependencies
npm install

# Configure environment variables
cp .env.example .env.local

# Start in development mode
npm run dev

# Access application
open http://localhost:5173
```

### Test Users

| Email | Password | Profile |
|-------|----------|---------|
| support@techcorp.com | Test@123 | support |
| admin@techcorp.com | Test@123 | admin |

## Available Modules

### 1. Order Management

- Order list with advanced filters
- Order details with status timeline
- Actions: cancel, refund, resend
- Export to CSV/Excel

### 2. User Management

- Search by email, tax ID, phone
- User's order history
- Actions: block, unblock, reset password
- Merge duplicate accounts

### 3. Product Management

- Product and variation CRUD
- Bulk upload via CSV
- Category and attribute management
- Price and promotion control

### 4. Reports

- Sales by period
- Best selling products
- Conversion metrics
- Data export

### 5. Settings

- Permission management
- Integration configuration
- Audit logs
- Feature flags

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/admin-dashboard
- **Sentry:** https://sentry.techcorp.internal/admin-dashboard
- **Audit Logs:** https://kibana.techcorp.internal/admin-audit

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `admin_active_sessions` | Active sessions | - |
| `admin_actions_total` | Total actions performed | - |
| `admin_error_rate` | Error rate | > 1% |
| `admin_page_load_time` | Page load time | > 3s |

## Troubleshooting

### Issue: Order table loading slowly

**Cause:** Too many active filters or very long period.

**Solution:**
1. Reduce search period
2. Remove unnecessary filters
3. Use export for large analyses

### Issue: Action returning error 403

**Cause:** User without permission for the action.

**Solution:**
1. Check user's profile
2. Request permission elevation from admin
3. Verify if action is available in the profile

### Issue: WebSocket disconnecting frequently

**Cause:** Connection timeout or network issue.

**Solution:**
1. Check connection stability
2. Reload the page
3. Check if firewall is blocking WebSocket

### Issue: Outdated data on screen

**Cause:** React Query cache or lack of invalidation.

**Solution:**
1. Use refresh button in toolbar
2. Clear browser cache
3. Check if WebSocket is connected

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+K` | Quick search |
| `Ctrl+Shift+O` | Go to orders |
| `Ctrl+Shift+U` | Go to users |
| `Ctrl+Shift+P` | Go to products |
| `Esc` | Close modal |

## Audit

All administrative actions are logged for auditing:

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "user_id": "uuid",
  "user_email": "admin@techcorp.com",
  "action": "order.cancel",
  "resource_type": "order",
  "resource_id": "uuid",
  "details": {
    "reason": "Customer requested cancellation"
  },
  "ip_address": "10.0.0.1"
}
```

## Related Links

- [Auth Service](auth-service.md) - Authentication and permissions
- [User Service](user-service.md) - User data
- [User Management](user-management.md) - User management guide
- [Order Service](order-service.md) - Order data
