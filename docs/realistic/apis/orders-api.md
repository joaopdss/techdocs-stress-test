# Orders API

## Overview

The Orders API provides REST resources for creating, querying, and managing orders on the TechCorp platform. This API is used by the web portal, mobile application, and partner system integrations.

## Base URL

```
https://api.techcorp.com/v1/orders
```

## Authentication

All REST resources require authentication via Bearer Token:

```
Authorization: Bearer <access_token>
```

## Available REST Resources

### GET /

This REST resource lists the authenticated user's orders.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Query Parameters:**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| page | integer | Page number | 1 |
| per_page | integer | Items per page | 20 |
| status | string | Filter by status | - |
| start_date | string | Start date (YYYY-MM-DD) | - |
| end_date | string | End date (YYYY-MM-DD) | - |
| sort | string | Sort by: created_at, total | created_at |
| order | string | Direction: asc, desc | desc |

**Response 200 (Success):**

```json
{
  "data": [
    {
      "id": "uuid",
      "number": "TC-2024-001234",
      "status": "DELIVERED",
      "total": 299.90,
      "items_count": 3,
      "created_at": "2024-01-15T10:30:00Z",
      "delivered_at": "2024-01-20T14:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 45,
    "total_pages": 3
  }
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Not authenticated |

---

### GET /{id}

This REST resource returns the complete details of an order.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| id | string | Order ID or number |

**Response 200 (Success):**

```json
{
  "id": "uuid",
  "number": "TC-2024-001234",
  "status": "DELIVERED",
  "payment_status": "PAID",
  "subtotal": 279.90,
  "shipping_cost": 20.00,
  "discount": 0.00,
  "total": 299.90,
  "items": [
    {
      "id": "uuid",
      "product_id": "uuid",
      "product_name": "Basic T-Shirt",
      "product_image": "https://cdn.techcorp.com/products/123.jpg",
      "variant": "Blue / M",
      "quantity": 2,
      "unit_price": 89.95,
      "total": 179.90
    },
    {
      "id": "uuid",
      "product_id": "uuid",
      "product_name": "Jeans",
      "product_image": "https://cdn.techcorp.com/products/456.jpg",
      "variant": "32",
      "quantity": 1,
      "unit_price": 100.00,
      "total": 100.00
    }
  ],
  "shipping_address": {
    "recipient_name": "John Smith",
    "street": "123 Main Street",
    "number": "123",
    "complement": "Apt 45",
    "neighborhood": "Downtown",
    "city": "New York",
    "state": "NY",
    "postal_code": "10001"
  },
  "payment": {
    "method": "credit_card",
    "brand": "Visa",
    "last_four_digits": "1234",
    "installments": 3,
    "installment_value": 99.97
  },
  "tracking": {
    "carrier": "UPS",
    "code": "1Z999AA10123456784",
    "url": "https://www.ups.com/track..."
  },
  "timeline": [
    {
      "status": "PENDING",
      "timestamp": "2024-01-15T10:30:00Z",
      "description": "Order received"
    },
    {
      "status": "PAYMENT_CONFIRMED",
      "timestamp": "2024-01-15T10:35:00Z",
      "description": "Payment confirmed"
    },
    {
      "status": "PROCESSING",
      "timestamp": "2024-01-16T08:00:00Z",
      "description": "Order being prepared"
    },
    {
      "status": "SHIPPED",
      "timestamp": "2024-01-17T16:00:00Z",
      "description": "Order shipped"
    },
    {
      "status": "DELIVERED",
      "timestamp": "2024-01-20T14:00:00Z",
      "description": "Order delivered"
    }
  ],
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-20T14:00:00Z"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Not authenticated |
| 404 | Order not found |

---

### POST /

This REST resource creates a new order from the cart.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "cart_id": "uuid",
  "shipping_address_id": "uuid",
  "billing_address_id": "uuid",
  "payment": {
    "method": "credit_card",
    "card_token": "tok_visa_1234",
    "installments": 3
  },
  "coupon_code": "DISCOUNT10",
  "notes": "Leave at the front desk"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| cart_id | string | Yes | Cart ID |
| shipping_address_id | string | Yes | Shipping address ID |
| billing_address_id | string | No | Billing address ID |
| payment | object | Yes | Payment data |
| payment.method | string | Yes | credit_card, debit_card, pix, boleto |
| payment.card_token | string | Conditional | Card token (if card) |
| payment.installments | integer | No | Installments (1-12) |
| coupon_code | string | No | Coupon code |
| notes | string | No | Notes |

**Response 201 (Success):**

```json
{
  "id": "uuid",
  "number": "TC-2024-001235",
  "status": "PENDING",
  "total": 299.90,
  "payment_url": "https://payment.techcorp.com/checkout/uuid",
  "pix_code": null,
  "boleto_url": null
}
```

**Response 201 (PIX):**

```json
{
  "id": "uuid",
  "number": "TC-2024-001235",
  "status": "PAYMENT_PENDING",
  "total": 299.90,
  "pix_code": "00020126580014br.gov.bcb.pix...",
  "pix_qr_code": "data:image/png;base64,...",
  "pix_expiration": "2024-01-15T11:30:00Z"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid data or empty cart |
| 401 | Not authenticated |
| 402 | Payment declined |
| 409 | Product out of stock |
| 422 | Invalid or expired coupon |

---

### POST /{id}/cancel

This REST resource cancels an order (before shipping).

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| id | string | Order ID |

**Request:**

```json
{
  "reason": "Changed my mind"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| reason | string | No | Cancellation reason |

**Response 200 (Success):**

```json
{
  "message": "Order cancelled successfully",
  "refund": {
    "amount": 299.90,
    "method": "credit_card",
    "estimated_date": "2024-01-25"
  }
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Order already shipped (cannot cancel) |
| 401 | Not authenticated |
| 404 | Order not found |

---

### POST /{id}/return

This REST resource requests a return for a delivered order.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| id | string | Order ID |

**Request:**

```json
{
  "items": [
    {
      "item_id": "uuid",
      "quantity": 1,
      "reason": "product_defect",
      "description": "Product arrived with a defect in the stitching"
    }
  ],
  "pickup_address_id": "uuid"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| items | array | Yes | Items to return |
| items[].item_id | string | Yes | Item ID |
| items[].quantity | integer | Yes | Quantity |
| items[].reason | string | Yes | Reason: product_defect, wrong_product, changed_mind |
| items[].description | string | No | Detailed description |
| pickup_address_id | string | Yes | Pickup address |

**Response 201 (Success):**

```json
{
  "return_id": "uuid",
  "status": "PENDING_APPROVAL",
  "message": "Return request registered. You will receive confirmation via email."
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Return period expired (30 days) |
| 401 | Not authenticated |
| 404 | Order or item not found |

---

### GET /{id}/invoice

This REST resource returns the order's invoice.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| id | string | Order ID |

**Query Parameters:**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| format | string | Format: json, pdf, xml | json |

**Response 200 (JSON):**

```json
{
  "invoice_number": "000123456",
  "invoice_key": "35240112345678000100550010001234561123456789",
  "issue_date": "2024-01-17T10:00:00Z",
  "pdf_url": "https://cdn.techcorp.com/invoices/uuid.pdf",
  "xml_url": "https://cdn.techcorp.com/invoices/uuid.xml"
}
```

**Response 200 (PDF):**

Returns the invoice PDF file.

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Not authenticated |
| 404 | Order not found or invoice not issued |

---

### GET /tracking/{code}

This REST resource returns the tracking status of an order.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| code | string | Tracking code |

**Response 200 (Success):**

```json
{
  "carrier": "UPS",
  "code": "1Z999AA10123456784",
  "status": "IN_TRANSIT",
  "estimated_delivery": "2024-01-20",
  "events": [
    {
      "timestamp": "2024-01-19T08:00:00Z",
      "status": "IN_TRANSIT",
      "location": "New York, NY",
      "description": "Out for delivery"
    },
    {
      "timestamp": "2024-01-18T16:00:00Z",
      "status": "IN_TRANSIT",
      "location": "Newark, NJ",
      "description": "In transit"
    },
    {
      "timestamp": "2024-01-17T16:00:00Z",
      "status": "POSTED",
      "location": "Los Angeles, CA",
      "description": "Shipment picked up"
    }
  ]
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Not authenticated |
| 404 | Tracking code not found |

## Order Status

| Status | Description |
|--------|-------------|
| `PENDING` | Order created, awaiting payment |
| `PAYMENT_PENDING` | Awaiting payment confirmation |
| `PAYMENT_CONFIRMED` | Payment confirmed |
| `PROCESSING` | Order being prepared |
| `SHIPPED` | Order shipped |
| `IN_TRANSIT` | In transit for delivery |
| `OUT_FOR_DELIVERY` | Out for delivery |
| `DELIVERED` | Delivered |
| `CANCELLED` | Cancelled |
| `RETURN_REQUESTED` | Return requested |
| `RETURNED` | Returned |
| `REFUNDED` | Refunded |

## Webhooks

For integrations, it's possible to receive status change notifications via webhook:

```json
{
  "event": "order.status_changed",
  "order_id": "uuid",
  "order_number": "TC-2024-001234",
  "previous_status": "SHIPPED",
  "new_status": "DELIVERED",
  "timestamp": "2024-01-20T14:00:00Z"
}
```

## Related Links

- [Order Service](../components/order-service.md) - Service documentation
- [Payments API](payments-api.md) - Payment operations
- [Products API](products-api.md) - Product API paths
- [Data Flow](../architecture/data-flow.md) - Order flow
