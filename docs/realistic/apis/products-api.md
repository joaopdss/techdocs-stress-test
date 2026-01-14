# Products API

## Overview

The Products API provides API paths for querying the product catalog, categories, search, and availability information. This API is public for listing and searching, but requires authentication for features like favorites and reviews.

## Base URL

```
https://api.techcorp.com/v1/products
```

## Authentication

Public API paths do not require authentication. API paths that modify data require:

```
Authorization: Bearer <access_token>
```

## Available API Paths

### GET /

This API path lists catalog products with filters and pagination.

**Query Parameters:**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| page | integer | Page number | 1 |
| per_page | integer | Items per page (max 100) | 20 |
| category | string | Category slug | - |
| brand | string | Brand slug | - |
| min_price | number | Minimum price | - |
| max_price | number | Maximum price | - |
| in_stock | boolean | Only with stock | false |
| sort | string | Sort by: price, name, popularity, newest | popularity |
| order | string | Direction: asc, desc | desc |
| attributes | string | Attribute filters (see below) | - |

**Attribute Filters:**

```
attributes=color:blue,red;size:m,l
```

**Response 200 (Success):**

```json
{
  "data": [
    {
      "id": "uuid",
      "slug": "basic-cotton-tshirt",
      "name": "Basic Cotton T-Shirt",
      "short_description": "100% cotton t-shirt, comfortable for everyday use",
      "price": 89.90,
      "original_price": 119.90,
      "discount_percentage": 25,
      "image": "https://cdn.techcorp.com/products/123/main.jpg",
      "brand": {
        "slug": "techcorp-basics",
        "name": "TechCorp Basics"
      },
      "category": {
        "slug": "tshirts",
        "name": "T-Shirts"
      },
      "rating": 4.5,
      "reviews_count": 127,
      "in_stock": true,
      "variants_count": 12
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 1523,
    "total_pages": 77
  },
  "facets": {
    "categories": [
      {"slug": "tshirts", "name": "T-Shirts", "count": 450},
      {"slug": "pants", "name": "Pants", "count": 320}
    ],
    "brands": [
      {"slug": "techcorp-basics", "name": "TechCorp Basics", "count": 200}
    ],
    "price_ranges": [
      {"min": 0, "max": 50, "count": 150},
      {"min": 50, "max": 100, "count": 400},
      {"min": 100, "max": 200, "count": 300}
    ],
    "attributes": {
      "color": [
        {"value": "blue", "count": 120},
        {"value": "red", "count": 80}
      ],
      "size": [
        {"value": "s", "count": 200},
        {"value": "m", "count": 250}
      ]
    }
  }
}
```

---

### GET /{slug}

This API path returns the complete details of a product.

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| slug | string | Product slug |

**Response 200 (Success):**

```json
{
  "id": "uuid",
  "slug": "basic-cotton-tshirt",
  "sku": "TSH-BAS-001",
  "name": "Basic Cotton T-Shirt",
  "description": "T-shirt made from 100% premium cotton...",
  "short_description": "100% cotton t-shirt, comfortable for everyday use",
  "price": 89.90,
  "original_price": 119.90,
  "discount_percentage": 25,
  "brand": {
    "slug": "techcorp-basics",
    "name": "TechCorp Basics",
    "logo": "https://cdn.techcorp.com/brands/techcorp-basics.png"
  },
  "category": {
    "slug": "tshirts",
    "name": "T-Shirts",
    "breadcrumb": [
      {"slug": "apparel", "name": "Apparel"},
      {"slug": "mens", "name": "Men's"},
      {"slug": "tshirts", "name": "T-Shirts"}
    ]
  },
  "images": [
    {
      "url": "https://cdn.techcorp.com/products/123/main.jpg",
      "alt": "Basic Cotton T-Shirt - Front",
      "is_main": true
    },
    {
      "url": "https://cdn.techcorp.com/products/123/back.jpg",
      "alt": "Basic Cotton T-Shirt - Back",
      "is_main": false
    }
  ],
  "variants": [
    {
      "id": "uuid",
      "sku": "TSH-BAS-001-BLU-M",
      "attributes": {
        "color": "Blue",
        "size": "M"
      },
      "price": 89.90,
      "in_stock": true,
      "stock_quantity": 15
    },
    {
      "id": "uuid",
      "sku": "TSH-BAS-001-BLU-L",
      "attributes": {
        "color": "Blue",
        "size": "L"
      },
      "price": 89.90,
      "in_stock": false,
      "stock_quantity": 0
    }
  ],
  "attributes": {
    "material": "100% Cotton",
    "origin": "USA",
    "care": "Machine wash cold"
  },
  "rating": 4.5,
  "reviews_count": 127,
  "questions_count": 23,
  "related_products": ["uuid1", "uuid2", "uuid3"],
  "created_at": "2024-01-01T10:00:00Z",
  "updated_at": "2024-01-15T14:30:00Z"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 404 | Product not found |

---

### GET /search

This API path performs full-text search on the catalog.

**Query Parameters:**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| q | string | Search term (required) | - |
| page | integer | Page number | 1 |
| per_page | integer | Items per page | 20 |
| filters | string | Additional filters | - |

**Response 200 (Success):**

```json
{
  "query": "blue tshirt",
  "data": [...],
  "pagination": {...},
  "facets": {...},
  "suggestions": {
    "did_you_mean": null,
    "related_searches": ["blue navy tshirt", "light blue tshirt"]
  }
}
```

---

### GET /search/autocomplete

This API path returns real-time search suggestions.

**Query Parameters:**

| Name | Type | Description |
|------|------|-------------|
| q | string | Partial term (minimum 2 characters) |

**Response 200 (Success):**

```json
{
  "suggestions": [
    {
      "type": "product",
      "text": "Basic Cotton T-Shirt",
      "slug": "basic-cotton-tshirt",
      "image": "https://cdn.techcorp.com/products/123/thumb.jpg"
    },
    {
      "type": "category",
      "text": "T-Shirts",
      "slug": "tshirts"
    },
    {
      "type": "brand",
      "text": "Shirt & Co",
      "slug": "shirt-co"
    }
  ]
}
```

---

### GET /categories

This API path lists all categories.

**Query Parameters:**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| parent | string | Parent category slug | - |
| depth | integer | Tree depth | 3 |

**Response 200 (Success):**

```json
{
  "data": [
    {
      "slug": "apparel",
      "name": "Apparel",
      "image": "https://cdn.techcorp.com/categories/apparel.jpg",
      "products_count": 2500,
      "children": [
        {
          "slug": "mens",
          "name": "Men's",
          "products_count": 1200,
          "children": [
            {"slug": "tshirts", "name": "T-Shirts", "products_count": 450},
            {"slug": "pants", "name": "Pants", "products_count": 320}
          ]
        }
      ]
    }
  ]
}
```

---

### GET /{slug}/reviews

This API path lists reviews for a product.

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| slug | string | Product slug |

**Query Parameters:**

| Name | Type | Description | Default |
|------|------|-------------|---------|
| page | integer | Page number | 1 |
| per_page | integer | Items per page | 10 |
| rating | integer | Filter by rating (1-5) | - |
| sort | string | Sort by: newest, helpful | newest |

**Response 200 (Success):**

```json
{
  "summary": {
    "average_rating": 4.5,
    "total_reviews": 127,
    "distribution": {
      "5": 80,
      "4": 30,
      "3": 10,
      "2": 5,
      "1": 2
    }
  },
  "data": [
    {
      "id": "uuid",
      "rating": 5,
      "title": "Excellent quality!",
      "content": "I bought it for everyday use and it exceeded expectations...",
      "author": {
        "name": "John S.",
        "verified_purchase": true
      },
      "variant": "Blue / M",
      "helpful_count": 15,
      "images": [
        "https://cdn.techcorp.com/reviews/uuid/1.jpg"
      ],
      "created_at": "2024-01-10T10:00:00Z"
    }
  ],
  "pagination": {...}
}
```

---

### POST /{slug}/reviews

This API path creates a review for a purchased product.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| slug | string | Product slug |

**Request:**

```json
{
  "order_item_id": "uuid",
  "rating": 5,
  "title": "Excellent quality!",
  "content": "I bought it for everyday use and it exceeded expectations...",
  "images": ["base64..."]
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| order_item_id | string | Yes | Order item ID |
| rating | integer | Yes | Rating from 1 to 5 |
| title | string | Yes | Review title |
| content | string | Yes | Review text |
| images | array | No | Images in base64 (max 5) |

**Response 201 (Success):**

```json
{
  "message": "Review submitted for moderation",
  "review_id": "uuid"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid data |
| 401 | Not authenticated |
| 403 | Did not purchase this product |
| 409 | Already reviewed this product |

---

### GET /{slug}/availability

This API path checks availability and delivery time.

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| slug | string | Product slug |

**Query Parameters:**

| Name | Type | Description |
|------|------|-------------|
| variant_id | string | Variant ID |
| postal_code | string | Postal code for shipping calculation |
| quantity | integer | Desired quantity |

**Response 200 (Success):**

```json
{
  "available": true,
  "stock_quantity": 15,
  "shipping_options": [
    {
      "carrier": "UPS Ground",
      "price": 15.90,
      "estimated_days": 7,
      "delivery_date": "2024-01-25"
    },
    {
      "carrier": "UPS Express",
      "price": 29.90,
      "estimated_days": 3,
      "delivery_date": "2024-01-21"
    }
  ]
}
```

---

### POST /favorites/{slug}

This API path adds a product to favorites.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 201 (Success):**

```json
{
  "message": "Product added to favorites"
}
```

---

### DELETE /favorites/{slug}

This API path removes a product from favorites.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Success):**

```json
{
  "message": "Product removed from favorites"
}
```

---

### GET /favorites

This API path lists the user's favorite products.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Success):**

```json
{
  "data": [
    {
      "product": {...},
      "added_at": "2024-01-10T10:00:00Z"
    }
  ],
  "pagination": {...}
}
```

## Related Links

- [Search Service](../components/search-service.md) - Search engine
- [Inventory Service](../components/inventory-service.md) - Inventory control
- [Orders API](orders-api.md) - Order REST resources
