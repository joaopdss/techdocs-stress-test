# Users API

## Overview

The Users API provides routes for managing user profile data, preferences, and addresses on the TechCorp platform. This API is used by both the web portal and mobile applications.

## Base URL

```
https://api.techcorp.com/v1/users
```

## Authentication

All routes require authentication via Bearer Token:

```
Authorization: Bearer <access_token>
```

## Available Routes

### GET /profile

This route returns the complete profile of the authenticated user.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Success):**

```json
{
  "id": "uuid",
  "email": "user@example.com",
  "name": "John Smith",
  "phone": "+15551234567",
  "document_number": "12345678900",
  "birth_date": "1990-05-15",
  "avatar_url": "https://cdn.techcorp.com/avatars/uuid.jpg",
  "status": "ACTIVE",
  "organization": {
    "id": "uuid",
    "name": "ABC Company"
  },
  "created_at": "2024-01-01T10:00:00Z",
  "updated_at": "2024-01-15T14:30:00Z"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Not authenticated |

---

### PUT /profile

This route updates the user's profile data.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "name": "John Smith Santos",
  "phone": "+15559998888",
  "birth_date": "1990-05-15"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| name | string | No | Full name |
| phone | string | No | Phone with country code |
| birth_date | string | No | Date in YYYY-MM-DD format |

**Response 200 (Success):**

```json
{
  "message": "Profile updated successfully",
  "user": {
    "id": "uuid",
    "name": "John Smith Santos",
    "phone": "+15559998888"
  }
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid data |
| 401 | Not authenticated |

---

### PUT /email

This route initiates the email change process.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "new_email": "newemail@example.com",
  "password": "currentPassword123"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| new_email | string | Yes | New email |
| password | string | Yes | Current password for confirmation |

**Response 200 (Success):**

```json
{
  "message": "Confirmation email sent to the new address"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid email |
| 401 | Incorrect password |
| 409 | Email already in use |

---

### PUT /password

This route changes the user's password.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "current_password": "currentPassword123",
  "new_password": "NewPassword@456",
  "new_password_confirmation": "NewPassword@456"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| current_password | string | Yes | Current password |
| new_password | string | Yes | New password (minimum 8 characters) |
| new_password_confirmation | string | Yes | New password confirmation |

**Response 200 (Success):**

```json
{
  "message": "Password changed successfully"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Weak password or doesn't match |
| 401 | Incorrect current password |

---

### GET /preferences

This route returns the user's preferences.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Response 200 (Success):**

```json
{
  "language": "en-US",
  "timezone": "America/New_York",
  "currency": "USD",
  "theme": "light",
  "notifications": {
    "email": true,
    "sms": false,
    "push": true,
    "marketing": false
  }
}
```

---

### PUT /preferences

This route updates the user's preferences.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "language": "en-US",
  "theme": "dark",
  "notifications": {
    "email": true,
    "sms": true,
    "push": true,
    "marketing": false
  }
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| language | string | No | Language: pt-BR, en-US, es-ES |
| timezone | string | No | IANA timezone |
| currency | string | No | Currency: BRL, USD |
| theme | string | No | Theme: light, dark, system |
| notifications | object | No | Notification preferences |

**Response 200 (Success):**

```json
{
  "message": "Preferences updated successfully"
}
```

---

### GET /addresses

This route lists the user's registered addresses.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Query Parameters:**

| Name | Type | Description |
|------|------|-------------|
| type | string | Filter by type: shipping, billing |

**Response 200 (Success):**

```json
{
  "data": [
    {
      "id": "uuid",
      "label": "Home",
      "type": "shipping",
      "is_default": true,
      "recipient_name": "John Smith",
      "street": "123 Main Street",
      "number": "123",
      "complement": "Apt 45",
      "neighborhood": "Downtown",
      "city": "New York",
      "state": "NY",
      "postal_code": "10001",
      "country": "US",
      "phone": "+15551234567"
    }
  ],
  "total": 1
}
```

---

### POST /addresses

This route registers a new address.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "label": "Work",
  "type": "shipping",
  "is_default": false,
  "recipient_name": "John Smith",
  "street": "456 Business Ave",
  "number": "1000",
  "complement": "10th floor",
  "neighborhood": "Financial District",
  "city": "New York",
  "state": "NY",
  "postal_code": "10005",
  "country": "US",
  "phone": "+15558887777"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| label | string | Yes | Address identifier |
| type | string | Yes | Type: shipping or billing |
| is_default | boolean | No | If it's the default address |
| recipient_name | string | Yes | Recipient name |
| street | string | Yes | Street address |
| number | string | Yes | Number |
| complement | string | No | Complement |
| neighborhood | string | Yes | Neighborhood |
| city | string | Yes | City |
| state | string | Yes | State |
| postal_code | string | Yes | Postal code |
| country | string | No | Country (default: US) |
| phone | string | No | Contact phone |

**Response 201 (Success):**

```json
{
  "message": "Address registered successfully",
  "address": {
    "id": "uuid",
    "label": "Work"
  }
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid data |
| 401 | Not authenticated |

---

### PUT /addresses/{id}

This route updates an existing address.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| id | string | Address ID |

**Request:**

```json
{
  "label": "New Work",
  "complement": "12th floor"
}
```

**Response 200 (Success):**

```json
{
  "message": "Address updated successfully"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid data |
| 404 | Address not found |

---

### DELETE /addresses/{id}

This route removes an address.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Path Parameters:**

| Name | Type | Description |
|------|------|-------------|
| id | string | Address ID |

**Response 200 (Success):**

```json
{
  "message": "Address removed successfully"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Cannot remove default address |
| 404 | Address not found |

---

### POST /avatar

This route uploads the user's avatar.

**Headers:**

```
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

**Request:**

```
file: <image file>
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| file | file | Yes | JPG or PNG image, maximum 5MB |

**Response 200 (Success):**

```json
{
  "avatar_url": "https://cdn.techcorp.com/avatars/uuid.jpg"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid or too large file |

---

### DELETE /account

This route requests account deletion (GDPR).

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "password": "currentPassword123",
  "reason": "I no longer use the service"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| password | string | Yes | Password for confirmation |
| reason | string | No | Reason for deletion |

**Response 200 (Success):**

```json
{
  "message": "Deletion request registered. Your account will be removed in 30 days."
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | There are pending orders |
| 401 | Incorrect password |

## Pagination

List routes support pagination:

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| page | integer | Page number | 1 |
| per_page | integer | Items per page | 20 |

## Related Links

- [User Service](../components/user-service.md) - Service documentation
- [Auth API](auth-api.md) - Authentication endpoints
- [User Management](../components/user-management.md) - Admin screen
