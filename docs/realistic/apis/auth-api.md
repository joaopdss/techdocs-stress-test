# Authentication API

## Overview

The Authentication API provides endpoints for identity and session management on the TechCorp platform. This API implements OAuth 2.0 and OpenID Connect standards for secure authentication.

## Base URL

```
https://api.techcorp.com/v1/auth
```

## Authentication

Most endpoints require authentication via Bearer Token in the header:

```
Authorization: Bearer <access_token>
```

Some endpoints are public (login, register) and do not require a token.

## Available Endpoints

### POST /login

This endpoint authenticates the user with credentials (email and password).

**Request:**

```json
{
  "email": "user@example.com",
  "password": "password123",
  "device_info": {
    "device_id": "uuid",
    "device_type": "web",
    "user_agent": "Mozilla/5.0..."
  }
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| email | string | Yes | User's email |
| password | string | Yes | User's password |
| device_info | object | No | Device information |

**Response 200 (Success):**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 1800,
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "name": "John Smith"
  }
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid request (missing fields) |
| 401 | Invalid credentials |
| 403 | Account blocked |
| 429 | Too many attempts, please wait |

---

### POST /logout

This endpoint ends the user's session, invalidating the access token.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "refresh_token": "eyJhbGciOiJSUzI1NiIs...",
  "logout_all_devices": false
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| refresh_token | string | Yes | Refresh token to invalidate |
| logout_all_devices | boolean | No | If true, invalidates all sessions |

**Response 200 (Success):**

```json
{
  "message": "Logout successful"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Invalid or expired token |

---

### POST /refresh

This endpoint renews the access token using a valid refresh token.

**Request:**

```json
{
  "refresh_token": "eyJhbGciOiJSUzI1NiIs..."
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| refresh_token | string | Yes | Valid refresh token |

**Response 200 (Success):**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 1800
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Invalid or expired refresh token |

---

### POST /register

This endpoint creates a new user account on the platform.

**Request:**

```json
{
  "email": "new@example.com",
  "password": "StrongPassword@123",
  "password_confirmation": "StrongPassword@123",
  "name": "Mary Santos",
  "phone": "+15551234567",
  "document_number": "12345678900",
  "birth_date": "1990-05-15",
  "accept_terms": true
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| email | string | Yes | Unique email |
| password | string | Yes | Minimum 8 characters, with letters and numbers |
| password_confirmation | string | Yes | Must match password |
| name | string | Yes | Full name |
| phone | string | Yes | Phone with country code |
| document_number | string | Yes | Tax ID (numbers only) |
| birth_date | string | Yes | Date in YYYY-MM-DD format |
| accept_terms | boolean | Yes | Must be true |

**Response 201 (Success):**

```json
{
  "message": "Account created successfully. Please check your email.",
  "user": {
    "id": "uuid",
    "email": "new@example.com",
    "status": "PENDING_VERIFICATION"
  }
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid data |
| 409 | Email or Tax ID already registered |

---

### POST /verify-email

This endpoint confirms the user's email through a code sent via email.

**Request:**

```json
{
  "email": "new@example.com",
  "code": "123456"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| email | string | Yes | Email to verify |
| code | string | Yes | 6-digit code |

**Response 200 (Success):**

```json
{
  "message": "Email verified successfully",
  "user": {
    "id": "uuid",
    "email": "new@example.com",
    "status": "ACTIVE"
  }
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid or expired code |
| 404 | Email not found |

---

### POST /forgot-password

This endpoint initiates the password recovery process.

**Request:**

```json
{
  "email": "user@example.com"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| email | string | Yes | Account email |

**Response 200 (Success):**

```json
{
  "message": "If the email exists, you will receive recovery instructions"
}
```

**Note:** This endpoint always returns 200, even if the email doesn't exist, to prevent account enumeration.

---

### POST /reset-password

This endpoint sets a new password using the recovery token.

**Request:**

```json
{
  "token": "abc123...",
  "password": "NewPassword@456",
  "password_confirmation": "NewPassword@456"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| token | string | Yes | Token received via email |
| password | string | Yes | New password |
| password_confirmation | string | Yes | New password confirmation |

**Response 200 (Success):**

```json
{
  "message": "Password changed successfully"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 400 | Invalid or expired token, or passwords don't match |

---

### POST /mfa/enable

This endpoint enables two-factor authentication for the account.

**Headers:**

```
Authorization: Bearer <access_token>
```

**Request:**

```json
{
  "type": "totp"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| type | string | Yes | MFA type: "totp" or "sms" |

**Response 200 (Success):**

```json
{
  "secret": "JBSWY3DPEHPK3PXP",
  "qr_code_url": "data:image/png;base64,...",
  "recovery_codes": [
    "abc123",
    "def456",
    "ghi789"
  ]
}
```

---

### POST /mfa/verify

This endpoint validates the MFA code during login.

**Request:**

```json
{
  "mfa_token": "eyJhbGciOiJSUzI1NiIs...",
  "code": "123456"
}
```

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| mfa_token | string | Yes | Temporary token received at login |
| code | string | Yes | Code from authenticator app or SMS |

**Response 200 (Success):**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 1800
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Invalid MFA code |
| 429 | Too many attempts |

---

### GET /me

This endpoint returns information about the authenticated user.

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
  "document_number": "***456***",
  "status": "ACTIVE",
  "mfa_enabled": true,
  "email_verified": true,
  "phone_verified": true,
  "created_at": "2024-01-01T10:00:00Z"
}
```

**Error Codes:**

| Code | Description |
|------|-------------|
| 401 | Invalid or expired token |

## Rate Limiting

Authentication endpoints have specific limits:

| Endpoint | Limit |
|----------|-------|
| /login | 5 attempts per minute per IP |
| /register | 3 attempts per minute per IP |
| /forgot-password | 3 attempts per hour per email |
| /mfa/verify | 3 attempts per minute |

## Related Links

- [Auth Service](../components/auth-service.md) - Service documentation
- [Security Model](../architecture/security-model.md) - Security architecture
- [Common Errors](../troubleshooting/common-errors.md) - Troubleshooting
