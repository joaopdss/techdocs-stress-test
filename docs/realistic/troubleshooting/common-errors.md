# Common Errors and Solutions

## Overview

This document lists the most frequent errors encountered on the TechCorp platform and their solutions. Use this reference for quick problem diagnosis.

## Authentication Errors

### Error: "Token expired" (401)

**Message:**
```json
{
  "error": "token_expired",
  "message": "Access token has expired"
}
```

**Cause:** JWT access token expired (30-minute validity).

**Solution:**
1. Use refresh token to obtain new access token
2. If refresh token also expired, request new login

**Example code:**
```javascript
if (error.code === 'token_expired') {
  const newTokens = await authService.refresh(refreshToken);
  // Retry original request with new token
}
```

---

### Error: "Invalid credentials" (401)

**Message:**
```json
{
  "error": "invalid_credentials",
  "message": "Incorrect email or password"
}
```

**Cause:** Email not registered or incorrect password.

**Solution:**
1. Verify email is correct
2. Use password recovery flow
3. Verify account is not locked

**Investigation (admin):**
```sql
SELECT email, status, failed_login_attempts, locked_until
FROM auth.users_credentials
WHERE email = 'user@example.com';
```

---

### Error: "Account locked" (403)

**Message:**
```json
{
  "error": "account_locked",
  "message": "Account locked after multiple attempts"
}
```

**Cause:** 5 consecutive failed login attempts.

**Solution:**
1. Wait 30 minutes (automatic unlock)
2. Use password recovery
3. Contact support for manual unlock

**Manual unlock (admin):**
```sql
UPDATE auth.users_credentials
SET locked_until = NULL, failed_login_attempts = 0
WHERE email = 'user@example.com';
```

---

### Error: "MFA required" (401)

**Message:**
```json
{
  "error": "mfa_required",
  "mfa_token": "temporary_token",
  "message": "Two-factor authentication required"
}
```

**Cause:** User has MFA enabled, needs to send code.

**Solution:**
1. Request code from authenticator app or SMS
2. Send code to `/auth/mfa/verify` with `mfa_token`

---

## Order Errors

### Error: "Product out of stock" (409)

**Message:**
```json
{
  "error": "out_of_stock",
  "message": "Product unavailable",
  "product_id": "uuid"
}
```

**Cause:** Stock ran out between adding to cart and completing checkout.

**Solution for user:**
1. Remove item from cart
2. Check alternatives (other colors/sizes)

**Investigation (admin):**
```sql
SELECT product_id, available, reserved
FROM inventory.stock_levels
WHERE product_id = 'uuid';
```

---

### Error: "Order cannot be cancelled" (400)

**Message:**
```json
{
  "error": "invalid_operation",
  "message": "Order has been shipped and cannot be cancelled"
}
```

**Cause:** Order is in SHIPPED status or later.

**Solution:**
1. Wait for delivery and request return
2. Refuse delivery (if possible)
3. Contact support for special cases

---

### Error: "Invalid coupon" (422)

**Message:**
```json
{
  "error": "invalid_coupon",
  "message": "Coupon expired or not applicable"
}
```

**Possible causes:**
- Coupon expired
- Minimum value not reached
- Coupon already used (single use)
- Coupon not applicable to cart items

**Verification (admin):**
```sql
SELECT code, expires_at, min_order_value, usage_limit, used_count
FROM coupons.coupons
WHERE code = 'DISCOUNT10';
```

---

## Payment Errors

### Error: "Payment declined" (402)

**Message:**
```json
{
  "error": "payment_declined",
  "decline_code": "insufficient_funds",
  "message": "Payment declined by issuer"
}
```

**Common decline codes:**

| Code | Cause | Action |
|------|-------|--------|
| `insufficient_funds` | Insufficient balance | Use another card |
| `card_declined` | Card blocked | Contact bank |
| `expired_card` | Card expired | Use valid card |
| `invalid_cvv` | Incorrect CVV | Verify code |
| `fraud_suspected` | Fraud suspected | Contact bank |

**Investigation (admin):**
```sql
SELECT transaction_id, status, decline_code, provider_response
FROM payments.transactions
WHERE order_id = 'uuid';
```

---

### Error: "Card token expired" (422)

**Message:**
```json
{
  "error": "token_expired",
  "message": "Card token has expired"
}
```

**Cause:** Card token (generated in frontend) has 15-minute validity.

**Solution:**
1. Request user to enter card details again
2. Check for excessive delay in checkout

---

### Error: "PIX expired" (422)

**Message:**
```json
{
  "error": "pix_expired",
  "message": "PIX code has expired"
}
```

**Cause:** PIX has default 30-minute validity.

**Solution:**
1. Generate new PIX code
2. Restart checkout if order was cancelled

---

## API Errors

### Error: "Rate limit exceeded" (429)

**Message:**
```json
{
  "error": "rate_limit_exceeded",
  "message": "Too many requests",
  "retry_after": 60
}
```

**Cause:** Client exceeded request limit.

**Default limits:**

| Type | Limit |
|------|-------|
| Public API | 100 req/min |
| Authenticated API | 1000 req/min |
| Login | 5 req/min |

**Solution:**
1. Wait time indicated in `retry_after`
2. Implement exponential backoff
3. Optimize calls (caching, batching)

---

### Error: "Service unavailable" (503)

**Message:**
```json
{
  "error": "service_unavailable",
  "message": "Service temporarily unavailable"
}
```

**Cause:** Downstream service is not responding.

**Solution for user:**
1. Wait a few minutes and try again
2. Check status page: status.techcorp.com

**Investigation (admin):**
```bash
# Check pod status
kubectl get pods -n production

# Check recent logs
kubectl logs -n production -l app=<service> --tail=100
```

---

### Error: "Bad Gateway" (502)

**Message:**
```json
{
  "error": "bad_gateway",
  "message": "Service communication error"
}
```

**Cause:** API Gateway could not communicate with upstream service.

**Investigation:**
1. Check if service is running
2. Check network policies
3. Check if circuit breaker is open

---

### Error: "Gateway Timeout" (504)

**Message:**
```json
{
  "error": "gateway_timeout",
  "message": "Timeout exceeded"
}
```

**Cause:** Upstream service took more than 30 seconds to respond.

**Investigation:**
1. Check service latency in Grafana
2. Check slow queries in database
3. Check for traffic spike

---

## Search Errors

### Error: "Search returning zero results"

**Possible cause:** Query too restrictive or index outdated.

**Investigation:**
```bash
# Check if term exists in index
curl -X GET "elasticsearch:9200/products/_search?q=<term>"

# Check index status
curl -X GET "elasticsearch:9200/_cat/indices/products"
```

**Solution:**
1. Broaden search filters
2. Check configured synonyms
3. Force reindexing if necessary

---

## Upload Errors

### Error: "File too large" (413)

**Message:**
```json
{
  "error": "file_too_large",
  "message": "File exceeds 5MB limit"
}
```

**Limits:**

| Type | Limit |
|------|-------|
| Avatar | 5MB |
| Review image | 10MB |
| Import CSV | 50MB |

**Solution:**
1. Compress image before sending
2. Resize to smaller dimensions

---

### Error: "File type not allowed" (415)

**Message:**
```json
{
  "error": "unsupported_media_type",
  "message": "File type not supported"
}
```

**Allowed types:**

| Context | Types |
|---------|-------|
| Avatar | JPG, PNG |
| Review | JPG, PNG, GIF |
| Import | CSV |

## Related Links

- [Performance Issues](performance-issues.md) - Optimization
- [Integration FAQ](integration-faq.md) - Integrations
- [Auth API](../apis/auth-api.md) - Auth endpoints
- [Payments API](../apis/payments-api.md) - Payment operations
