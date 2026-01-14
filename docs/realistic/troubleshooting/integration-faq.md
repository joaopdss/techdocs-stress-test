# Integration FAQ

## Overview

This document answers the most frequently asked questions about integrations with the TechCorp platform, including APIs, webhooks, and external systems.

## Authentication and Authorization

### How do I obtain API credentials?

To obtain API credentials for integration:

1. Access the Developer Portal: developers.techcorp.com
2. Create an account or log in
3. Register a new application
4. Obtain your `client_id` and `client_secret`

**Important:** The `client_secret` is displayed only once. Store it securely.

---

### What's the difference between client_credentials and authorization_code?

| Flow | Use | When to use |
|------|-----|-------------|
| `client_credentials` | Server-to-server | Backend integration without user |
| `authorization_code` | User delegation | App acting on behalf of user |

**client_credentials example:**
```bash
curl -X POST https://api.techcorp.com/v1/auth/oauth/token \
  -d grant_type=client_credentials \
  -d client_id=<your_client_id> \
  -d client_secret=<your_client_secret>
```

---

### My token expired, what do I do?

Access tokens expire in 30 minutes. Use the refresh token to obtain a new one:

```bash
curl -X POST https://api.techcorp.com/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "<your_refresh_token>"}'
```

If the refresh token also expired (7 days), you need to authenticate again.

---

### How do I test the API in sandbox environment?

Use the sandbox environment for testing:

- **Base URL:** `https://api.sandbox.techcorp.com/v1`
- **Portal:** `https://sandbox.techcorp.com`

Sandbox credentials are separate from production. Create an account on the sandbox portal.

---

## Webhooks

### How do I configure webhooks?

1. Access the Developer Portal
2. Go to Settings > Webhooks
3. Add your endpoint URL
4. Select the events you want to receive
5. Save and copy the `webhook_secret`

---

### What webhook events are available?

| Event | Description |
|-------|-------------|
| `order.created` | New order created |
| `order.paid` | Payment confirmed |
| `order.shipped` | Order shipped |
| `order.delivered` | Order delivered |
| `order.cancelled` | Order cancelled |
| `payment.confirmed` | Payment approved |
| `payment.failed` | Payment declined |
| `payment.refunded` | Refund processed |
| `user.created` | New user registered |
| `user.updated` | User data changed |

---

### How do I validate the webhook signature?

All webhooks include the `X-Webhook-Signature` header with HMAC-SHA256 signature:

```python
import hmac
import hashlib

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

# Usage
is_valid = verify_webhook(
    request.body,
    request.headers['X-Webhook-Signature'],
    WEBHOOK_SECRET
)
```

---

### Webhook is not arriving, what do I check?

Troubleshooting checklist:

1. **URL accessible?** Endpoint must be public (HTTPS)
2. **Fast response?** Respond with 2xx in less than 5 seconds
3. **Valid certificate?** SSL/TLS must be valid
4. **Firewall?** Whitelist our IPs (see below)
5. **Event configured?** Verify the event is selected

**IPs for whitelist:**
```
54.123.45.67
54.123.45.68
54.123.45.69
```

---

### What happens if my endpoint is down?

We implement automatic retry with exponential backoff:

| Attempt | Delay |
|---------|-------|
| 1 | Immediate |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 30 minutes |
| 5 | 2 hours |

After 5 failures, the webhook is moved to DLQ. You can reprocess via Portal.

---

## Payments

### What payment providers are supported?

| Provider | Methods | Status |
|----------|---------|--------|
| Stripe | Credit/debit card | Active |
| PagSeguro | Card, Boleto, PIX | Active |
| MercadoPago | Card, Boleto, PIX | Active |
| PayPal | PayPal Wallet | Active |

---

### How do I process payments in my integration?

To process payments via API:

1. **Tokenize the card** in frontend using our SDK
2. **Send the token** to your backend
3. **Call the payments API** with the token

```javascript
// Frontend: Tokenize card
const token = await TechCorpPay.createToken({
  number: '4242424242424242',
  exp_month: 12,
  exp_year: 2025,
  cvv: '123'
});

// Backend: Process payment
const payment = await fetch('/api/payments/process', {
  method: 'POST',
  body: JSON.stringify({
    order_id: 'uuid',
    amount: 29990,
    card_token: token.id
  })
});
```

---

### How do I test payments in sandbox?

Use the test cards:

| Number | Result |
|--------|--------|
| 4242424242424242 | Approved |
| 4000000000000002 | Declined |
| 4000000000009995 | Insufficient funds |
| 4000000000000069 | Expired card |

For test PIX, use the returned QR code - the payment is automatically confirmed in 5 seconds.

---

## Data and Synchronization

### How do I sync products with my system?

Synchronization options:

1. **Pull via API:** Call `GET /products` periodically
2. **Push via Webhook:** Receive `product.updated` events
3. **Full feed:** Download daily CSV from Portal

We recommend webhook for real-time updates + daily feed for reconciliation.

---

### What is the API rate limit?

| Type | Limit | Window |
|------|-------|--------|
| Authenticated API | 1000 req | 1 minute |
| Public API | 100 req | 1 minute |
| Search endpoints | 60 req | 1 minute |
| Write endpoints | 30 req | 1 minute |

Response headers indicate usage:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1705312260
```

---

### How do I paginate correctly?

Use cursor-based pagination for large volumes:

```bash
# First page
GET /api/orders?limit=100

# Response includes cursor
{
  "data": [...],
  "next_cursor": "eyJpZCI6IjEyMyJ9"
}

# Next page
GET /api/orders?limit=100&cursor=eyJpZCI6IjEyMyJ9
```

**Do not use** offset for large volumes - performance degrades.

---

## Errors and Debugging

### How do I debug integration errors?

1. **Check the error code:** Consult [Common Errors](common-errors.md)
2. **Check response headers:** `X-Request-Id` for support
3. **Check logs in Portal:** Call history for 7 days
4. **Contact support:** Include `X-Request-Id`

---

### Error 429: What to do?

You exceeded the rate limit. Solutions:

1. **Implement exponential backoff:**
```python
import time

def call_with_retry(func, max_retries=5):
    for i in range(max_retries):
        try:
            return func()
        except RateLimitError as e:
            wait = 2 ** i
            time.sleep(wait)
    raise Exception("Max retries exceeded")
```

2. **Optimize calls:**
   - Use local caching
   - Batch operations when possible
   - Avoid unnecessary polling

3. **Request limit increase:** Contact support with justification

---

### SSL certificate error, what to do?

If you're receiving SSL errors:

1. **Check date/time:** Your server must have correct time
2. **Update CA certificates:** `apt-get update && apt-get install ca-certificates`
3. **Check proxy:** Corporate proxies may interfere

**Never** disable certificate verification in production.

---

## Support

### How do I contact technical support?

| Channel | Use | SLA |
|---------|-----|-----|
| Developer Portal | General questions | 48h |
| developers@techcorp.com | Technical issues | 24h |
| Slack #api-support | Enterprise partners | 4h |

Always include:
- `X-Request-Id` from the error
- Problem timestamp
- Request/Response (without sensitive data)

---

### Where do I find complete documentation?

- **API Reference:** docs.techcorp.com/api
- **Guides:** docs.techcorp.com/guides
- **SDKs:** github.com/techcorp/sdks
- **Status:** status.techcorp.com

## Related Links

- [Common Errors](common-errors.md) - Error codes
- [Auth API](../apis/auth-api.md) - Auth endpoints
- [Orders API](../apis/orders-api.md) - Order REST resources
- [Payments API](../apis/payments-api.md) - Payment operations
