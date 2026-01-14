# Notification Service

## Description

The Notification Service is TechCorp's centralized microservice for sending notifications. This component manages communication dispatch through multiple channels: email, SMS, push notifications, and in-app messages.

The service implements a template system that allows message personalization by channel and language. It also offers advanced features such as scheduled sending, frequency control (throttling), and per-user opt-out preferences.

To ensure delivery, the notification-service uses queues with exponential retry and fallback between providers. All notifications are logged for auditing and engagement metrics.

## Owners

- **Team:** Customer Communications
- **Tech Lead:** Amanda Souza
- **Slack:** #comms-notifications

## Technology Stack

- Language: Node.js 20
- Framework: NestJS 10
- Database: PostgreSQL 15
- Cache: Redis 7
- Message Broker: RabbitMQ 3.12
- Template Engine: Handlebars

## Supported Channels

| Channel | Primary Provider | Fallback Provider |
|---------|------------------|-------------------|
| Email | SendGrid | Amazon SES |
| SMS | Twilio | Zenvia |
| Push (iOS) | APNs | - |
| Push (Android) | FCM | - |
| In-App | Internal WebSocket | - |

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `NOTIFICATION_PORT` | Service HTTP port | `3005` |
| `DATABASE_URL` | PostgreSQL connection string | - |
| `REDIS_URL` | Redis URL | - |
| `RABBITMQ_URL` | RabbitMQ URL | - |
| `SENDGRID_API_KEY` | SendGrid API key | - |
| `TWILIO_ACCOUNT_SID` | Twilio Account SID | - |
| `TWILIO_AUTH_TOKEN` | Twilio Auth Token | - |
| `FCM_SERVER_KEY` | Firebase Cloud Messaging key | - |
| `APNS_KEY_ID` | Apple Push Key ID | - |
| `APNS_TEAM_ID` | Apple Push Team ID | - |

### Rate Limiting Configuration

```yaml
# config/throttling.yaml
throttling:
  email:
    per_user_per_hour: 10
    per_user_per_day: 50
    global_per_minute: 1000
  sms:
    per_user_per_hour: 5
    per_user_per_day: 20
    global_per_minute: 500
  push:
    per_user_per_hour: 20
    per_user_per_day: 100
    global_per_minute: 5000
```

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/notification-service.git
cd notification-service

# Install dependencies
npm install

# Start dependencies
docker-compose up -d postgres redis rabbitmq

# Run migrations
npm run migration:run

# Start in development mode
npm run start:dev

# Verify health
curl http://localhost:3005/health
```

### Send Test Notification

```bash
# Send test email
curl -X POST http://localhost:3005/api/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "user_id": "uuid",
    "channel": "email",
    "template": "welcome",
    "data": {
      "user_name": "John"
    }
  }'
```

## Template System

### Template Structure

```
templates/
├── email/
│   ├── en-US/
│   │   ├── welcome.hbs
│   │   ├── order-confirmed.hbs
│   │   └── password-reset.hbs
│   └── pt-BR/
│       └── ...
├── sms/
│   ├── en-US/
│   │   ├── order-shipped.hbs
│   │   └── verification-code.hbs
│   └── pt-BR/
│       └── ...
└── push/
    └── ...
```

### Template Example

```handlebars
{{! templates/email/en-US/order-confirmed.hbs }}
<h1>Order Confirmed!</h1>
<p>Hello {{user_name}},</p>
<p>Your order <strong>#{{order_number}}</strong> has been confirmed.</p>
<p>Total amount: ${{format_currency total}}</p>
<p>Expected delivery: {{format_date delivery_date}}</p>
```

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/notification-service
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets
- **Logs:** Sent to Elasticsearch via Fluentd

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `notifications_sent_total` | Total notifications sent | - |
| `notifications_delivered_total` | Notifications delivered | - |
| `notifications_failed_total` | Send failures | > 5% |
| `notifications_latency_seconds` | Send latency | > 30s |
| `notifications_by_channel` | Sends by channel | - |

### Configured Alerts

- **NotificationDeliveryRateLow:** Delivery rate below 95% for 10 minutes
- **NotificationProviderDown:** Primary provider unavailable
- **NotificationQueueBacklog:** Queue with more than 10,000 pending messages
- **NotificationBounceRateHigh:** Email bounce rate above 5%

## Troubleshooting

### Issue: Notifications are not being sent

**Cause:** Queue stopped or worker crashing.

**Solution:**
1. Check queue status: `rabbitmqctl list_queues`
2. Check worker logs: `kubectl logs -l app=notification-worker`
3. Restart workers if necessary: `kubectl rollout restart deployment/notification-worker`

### Issue: Emails going to spam

**Cause:** Incorrect SPF/DKIM configuration or low IP reputation.

**Solution:**
1. Check DNS configuration (SPF, DKIM, DMARC)
2. Analyze IP reputation in SendGrid dashboard
3. Temporarily reduce volume if IP is blacklisted

### Issue: Push notifications not arriving on iOS

**Cause:** Expired device token or expired APNs certificate.

**Solution:**
1. Check APNs certificate validity
2. Force token refresh in the app
3. Check Apple feedback service for invalid tokens

### Issue: Rate limit reached for user

**Cause:** Throttling configuration too restrictive or send loop.

**Solution:**
1. Check user's send history: `GET /api/notifications?user_id=<id>`
2. Identify source of excessive sends
3. Adjust throttling if necessary or fix bug in source service

## Consumed Events

| Event | Source | Template |
|-------|--------|----------|
| `user.created` | user-service | welcome |
| `order.confirmed` | order-service | order-confirmed |
| `order.shipped` | order-service | order-shipped |
| `order.delivered` | order-service | order-delivered |
| `payment.failed` | payment-service | payment-failed |
| `auth.password.reset` | auth-service | password-reset |

## Published Events

| Event | Exchange | Description |
|-------|----------|-------------|
| `notification.sent` | `notification.events` | Notification sent |
| `notification.delivered` | `notification.events` | Delivery confirmed |
| `notification.failed` | `notification.events` | Send failure |
| `notification.opened` | `notification.events` | Email opened |
| `notification.clicked` | `notification.events` | Link clicked |

## Related Links

- [User Service](user-service.md) - User notification preferences
- [Order Service](order-service.md) - Order events
- [Queue Service](queue-service.md) - Queue infrastructure
- [Common Errors](../troubleshooting/common-errors.md) - Frequent issues
