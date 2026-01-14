# Payments API

## Overview

The Payments API provides operations for processing financial transactions, querying payments, and managing saved payment methods. This API integrates with multiple payment providers transparently.

## Base URL

```
https://api.techcorp.com/v1/payments
```

## Authentication

All operations require authentication via Bearer Token:

```
Authorization: Bearer <access_token>
```

## Available Operations

### POST /process

This operation processes a new payment.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request (Credit Card):**

```json
{
  "order_id": "uuid",
  "amount": 29990,
  "currency": "USD",
  "method": "credit_card",
  "card": {
    "token": "tok_visa_4242",
    "installments": 3,
    "save_card": true
  },
  "billing_address": {
    "postal_code": "10001",
    "number": "123"
  }
}
```

**Request (PIX):**

```json
{
  "order_id": "uuid",
  "amount": 29990,
  "currency": "BRL",
  "method": "pix",
  "pix": {
    "expiration_minutes": 30
  }
}
```

**Request (Boleto):**

```json
{
  "order_id": "uuid",
  "amount": 29990,
  "currency": "BRL",
  "method": "boleto",
  "boleto": {
    "due_days": 3,
    "instructions": "Do not accept after due date"
  }
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| order_id | string | Yes | Order ID |
| amount | integer | Yes | Amount in cents |
| currency | string | Yes | Currency (USD, BRL) |
| method | string | Yes | credit_card, debit_card, pix, boleto |
| card.token | string | Conditional | Card token (if card) |
| card.installments | integer | No | Installments (1-12) |
| card.save_card | boolean | No | Save card for future purchases |
| pix.expiration_minutes | integer | No | PIX expiration (default: 30) |
| boleto.due_days | integer | No | Days until due (default: 3) |

**Response 200 (Card Approved):**

```json
{
  "transaction_id": "uuid",
  "status": "APPROVED",
  "amount": 29990,
  "installments": 3,
  "installment_amount": 9997,
  "authorization_code": "123456",
  "card": {
    "brand": "Visa",
    "last_four": "4242",
    "saved_card_id": "uuid"
  },
  "processed_at": "2024-01-15T10:30:00Z"
}
```

**Response 200 (PIX Generated):**

```json
{
  "transaction_id": "uuid",
  "status": "PENDING",
  "amount": 29990,
  "pix": {
    "code": "00020126580014br.gov.bcb.pix0136...",
    "qr_code_base64": "data:image/png;base64,...",
    "expires_at": "2024-01-15T11:00:00Z"
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Response 200 (Boleto Generated):**

```json
{
  "transaction_id": "uuid",
  "status": "PENDING",
  "amount": 29990,
  "boleto": {
    "barcode": "23793.38128 60000.000003 00000.000400 1 84340000029990",
    "digitable_line": "23793381286000000000300000004001843400000299.90",
    "pdf_url": "https://cdn.techcorp.com/boletos/uuid.pdf",
    "due_date": "2024-01-18"
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid data |
| 401 | Not authenticated |
| 402 | Payment declined |
| 409 | Order already paid |
| 422 | Invalid or expired card |

---

### GET /{id}

This operation returns the details of a transaction.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| id | string | Transaction ID |

**Response 200 (Success):**

```json
{
  "transaction_id": "uuid",
  "order_id": "uuid",
  "status": "APPROVED",
  "amount": 29990,
  "currency": "USD",
  "method": "credit_card",
  "installments": 3,
  "installment_amount": 9997,
  "card": {
    "brand": "Visa",
    "last_four": "4242"
  },
  "events": [
    {
      "type": "AUTHORIZED",
      "timestamp": "2024-01-15T10:30:00Z"
    },
    {
      "type": "CAPTURED",
      "timestamp": "2024-01-15T10:30:05Z"
    }
  ],
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Not authenticated |
| 404 | Transaction not found |

---

### POST /{id}/refund

This operation processes full or partial refund.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| id | string | Transaction ID |

**Request:**

```json
{
  "amount": 10000,
  "reason": "customer_request",
  "description": "Customer requested partial cancellation"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| amount | integer | No | Amount in cents (if partial) |
| reason | string | Yes | customer_request, product_issue, fraud |
| description | string | No | Reason description |

**Response 200 (Success):**

```json
{
  "refund_id": "uuid",
  "transaction_id": "uuid",
  "status": "PROCESSED",
  "amount": 10000,
  "method": "original_payment",
  "estimated_arrival": "2024-01-25",
  "processed_at": "2024-01-15T14:00:00Z"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Amount greater than available |
| 401 | Not authenticated |
| 404 | Transaction not found |
| 422 | Transaction does not allow refund |

---

### GET /methods

This operation lists available payment methods.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Query Parameters:**

| Name | Type | Description |
|------|------|-------------|
| amount | integer | Amount to calculate installments |

**Response 200 (Success):**

```json
{
  "credit_card": {
    "enabled": true,
    "brands": ["visa", "mastercard", "amex", "discover"],
    "installments": [
      {"quantity": 1, "amount": 29990, "interest_free": true},
      {"quantity": 2, "amount": 14995, "interest_free": true},
      {"quantity": 3, "amount": 9997, "interest_free": true},
      {"quantity": 6, "amount": 5248, "interest_free": false, "total": 31488},
      {"quantity": 12, "amount": 2749, "interest_free": false, "total": 32988}
    ],
    "saved_cards": [
      {
        "id": "uuid",
        "brand": "Visa",
        "last_four": "4242",
        "holder_name": "JOHN SMITH",
        "expiry_month": 12,
        "expiry_year": 2025,
        "is_default": true
      }
    ]
  },
  "debit_card": {
    "enabled": true,
    "brands": ["visa", "mastercard"]
  },
  "pix": {
    "enabled": true,
    "discount_percentage": 5
  },
  "boleto": {
    "enabled": true,
    "due_days_options": [1, 3, 5]
  }
}
```

---

### POST /cards

This operation saves a new card for future purchases.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "token": "tok_visa_4242",
  "set_default": true
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| token | string | Yes | Card token (generated by frontend) |
| set_default | boolean | No | Set as default |

**Response 201 (Success):**

```json
{
  "card_id": "uuid",
  "brand": "Visa",
  "last_four": "4242",
  "holder_name": "JOHN SMITH",
  "expiry_month": 12,
  "expiry_year": 2025,
  "is_default": true
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid token |
| 401 | Not authenticated |
| 409 | Card already registered |
| 422 | Expired or invalid card |

---

### DELETE /cards/{id}

This operation removes a saved card.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| id | string | Card ID |

**Response 200 (Success):**

```json
{
  "message": "Card removed successfully"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Not authenticated |
| 404 | Card not found |

---

### PUT /cards/{id}/default

This operation sets a card as default.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Success):**

```json
{
  "message": "Card set as default"
}
```

---

### POST /validate-card

This operation validates a card without charging (verification).

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "token": "tok_visa_4242"
}
```

**Response 200 (Success):**

```json
{
  "valid": true,
  "brand": "Visa",
  "last_four": "4242",
  "country": "US",
  "type": "credit"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 422 | Invalid card |

## Transaction Status

| Status | Description |
|--------|-------------|
| `PENDING` | Awaiting payment (PIX/Boleto) |
| `AUTHORIZED` | Authorized, not captured |
| `APPROVED` | Approved and captured |
| `DECLINED` | Declined by issuer |
| `CANCELLED` | Cancelled before capture |
| `REFUNDED` | Fully refunded |
| `PARTIALLY_REFUNDED` | Partially refunded |
| `EXPIRED` | PIX/Boleto expired |
| `CHARGEBACK` | Cardholder dispute |

## Decline Codes

| Code | Description | Suggested Action |
|------|-------------|------------------|
| `insufficient_funds` | Insufficient funds | Try another card |
| `card_declined` | Card declined | Contact bank |
| `expired_card` | Expired card | Use valid card |
| `invalid_cvv` | Incorrect CVV | Verify code |
| `fraud_suspected` | Fraud suspected | Contact bank |
| `processing_error` | Processing error | Try again |

## Security

- All card data is tokenized on the frontend
- The API never receives or stores complete card numbers
- Transactions are processed in PCI-DSS compliant environment
- Card tokens expire in 15 minutes

## Related Links

- [Payment Service](../components/payment-service.md) - Service documentation
- [Orders API](orders-api.md) - Order REST resources
- [Security Model](../architecture/security-model.md) - Payment security
- [Common Errors](../troubleshooting/common-errors.md) - Payment issues
