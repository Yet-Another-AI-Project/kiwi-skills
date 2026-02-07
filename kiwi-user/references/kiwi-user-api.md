# Kiwi-User API Reference

Complete endpoint documentation for the kiwi-user authentication service.

All responses follow the format:
```json
{
  "status": "success" | "error",
  "data": { ... },
  "error": { ... }  // only on error
}
```

## Login Endpoints

All login endpoints are public (no auth required). All return `LoginResponse`.

### LoginResponse (Common Response)

```json
{
  "refresh_token": "string",
  "refresh_token_expires_at": 1700000000,
  "access_token": "string (JWT)",
  "access_token_expires_at": 1700000600,
  "type": "string",
  "device_type": "string",
  "device_id": "string",
  "user_id": "string"
}
```

### Device Object (Common Request Field)

```json
{
  "device_type": "web",
  "device_id": "uuid-string"
}
```

Both fields are required in all login and token requests.

---

### POST /v1/login/google/web

Google OAuth login for web applications.

**Request:**
```json
{
  "application_name": "my-app",
  "code": "google-authorization-code",
  "redirect_uri": "https://example.com/auth/google/callback",
  "referral_channel": "optional-referral",
  "device": { "device_type": "web", "device_id": "uuid" }
}
```

Alternative (with ID token instead of code):
```json
{
  "application_name": "my-app",
  "id_token": "google-id-token",
  "device": { "device_type": "web", "device_id": "uuid" }
}
```

**Response:** `LoginResponse`

---

### POST /v1/login/email

Request email verification code.

**Request:**
```json
{
  "application_name": "my-app",
  "email": "user@example.com"
}
```

### POST /v1/login/email/verify_code

Verify email code and get tokens.

**Request:**
```json
{
  "application_name": "my-app",
  "email": "user@example.com",
  "verify_code": "123456",
  "device": { "device_type": "web", "device_id": "uuid" }
}
```

**Response:** `LoginResponse`

---

### POST /v1/login/phone

Request phone verification code.

**Request:**
```json
{
  "application_name": "my-app",
  "phone": "+1234567890"
}
```

### POST /v1/login/phone/verify_code

Verify phone code and get tokens.

**Request:**
```json
{
  "application_name": "my-app",
  "phone": "+1234567890",
  "verify_code": "123456",
  "device": { "device_type": "web", "device_id": "uuid" }
}
```

**Response:** `LoginResponse`

---

### POST /v1/login/password

Direct password login.

**Request:**
```json
{
  "application_name": "my-app",
  "username": "user@example.com",
  "password": "password123",
  "device": { "device_type": "web", "device_id": "uuid" }
}
```

**Response:** `LoginResponse`

---

### POST /v1/login/wechat/miniprogram

WeChat Mini Program login.

**Request:**
```json
{
  "application_name": "my-app",
  "code": "wechat-login-code",
  "device": { "device_type": "miniprogram", "device_id": "uuid" }
}
```

**Response:** `LoginResponse`

---

### POST /v1/login/wechat/web

WeChat Web OAuth login.

**Request:**
```json
{
  "application_name": "my-app",
  "code": "wechat-oauth-code",
  "device": { "device_type": "web", "device_id": "uuid" }
}
```

**Response:** `LoginResponse`

---

### POST /v1/login/organization

Organization-specific login.

**Request:**
```json
{
  "application_name": "my-app",
  "organization_id": "org-id",
  "code": "invite-code",
  "device": { "device_type": "web", "device_id": "uuid" }
}
```

**Response:** `LoginResponse`

---

## Token Endpoints

### POST /v1/token/verify

Verify an access token and get user info. Public endpoint (no auth required).

**Request:**
```json
{
  "access_token": "jwt-token-string"
}
```

**Response (valid token):**
```json
{
  "success": true,
  "user_info": {
    "id": "user-uuid",
    "application": "my-app",
    "name": "John Doe",
    "avatar": "https://...",
    "phone": "+1234567890",
    "email": "user@example.com",
    "personal_role": "user",
    "personal_scopes": ["read", "write"],
    "current_org_id": "org-uuid",
    "department": "Engineering"
  }
}
```

**Response (invalid/expired token):**
```json
{
  "success": false
}
```

Note: Invalid tokens return `success: false` without an error status code. The response is still HTTP 200.

---

### GET /v1/token/publickey

Get the RSA public key for local JWT verification. Public endpoint.

**Response:**
```json
{
  "public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkq...\n-----END PUBLIC KEY-----"
}
```

The public key is used for RS256 signature verification of access tokens. Cache this key (it rarely changes).

---

### POST /v1/token/refresh

Refresh both access and refresh tokens. Public endpoint.

**Request:**
```json
{
  "user_id": "user-uuid",
  "refresh_token": "current-refresh-token",
  "device": { "device_type": "web", "device_id": "uuid" }
}
```

**Response:** `LoginResponse` (contains new access AND refresh tokens)

Both tokens are replaced. The old refresh token is invalidated.

---

## User Endpoints

All user endpoints require authentication (Bearer token in Authorization header).

### GET /v1/user/info

Get the authenticated user's profile.

**Headers:** `Authorization: Bearer <access_token>`

**Response:**
```json
{
  "id": "user-uuid",
  "application": "my-app",
  "name": "John Doe",
  "avatar": "https://...",
  "phone": "+1234567890",
  "email": "user@example.com",
  "personal_role": "user",
  "personal_scopes": ["read", "write"],
  "current_org_id": "org-uuid",
  "orgs": [
    {
      "id": "org-uuid",
      "name": "My Org",
      "role": "member"
    }
  ],
  "department": "Engineering"
}
```

---

### PUT /v1/user/info

Update the authenticated user's profile.

**Headers:** `Authorization: Bearer <access_token>`

**Request:**
```json
{
  "name": "New Name",
  "avatar": "https://new-avatar-url"
}
```

---

### POST /v1/user/logout

Logout and invalidate tokens.

**Headers:** `Authorization: Bearer <access_token>`

**Request:**
```json
{
  "user_id": "user-uuid",
  "refresh_token": "current-refresh-token",
  "device": { "device_type": "web", "device_id": "uuid" }
}
```

---

## Internal Endpoints

These endpoints are for service-to-service communication. Not exposed to frontend.

### POST /internal/user/infos

Get user info for multiple users by ID.

**Request:**
```json
{
  "user_ids": ["user-uuid-1", "user-uuid-2"]
}
```

**Response:**
```json
[
  {
    "id": "user-uuid-1",
    "application": "my-app",
    "name": "User One",
    "display_name": "User One",
    "avatar": "https://...",
    "username": "userone",
    "department": "Engineering"
  },
  {
    "id": "user-uuid-2",
    "application": "my-app",
    "name": "User Two",
    "display_name": "User Two",
    "avatar": "https://...",
    "username": "usertwo",
    "department": "Design"
  }
]
```

---

### POST /internal/organization/infos

Get organization info for multiple orgs by ID.

**Request:**
```json
{
  "organization_ids": ["org-uuid-1", "org-uuid-2"]
}
```

---

## JWT Access Token Claims

The access token is an RS256-signed JWT with these claims:

| Claim | Key | Type | Description |
|-------|-----|------|-------------|
| Subject | `sub` | string | User ID |
| Issuer | `iss` | string | Application name |
| Issued At | `iat` | int64 | Creation timestamp (unix seconds) |
| Expiration | `exp` | int64 | Expiration timestamp (unix seconds) |
| Type | `type` | string | Always `"access"` |
| Roles | `roles` | string | Personal role |
| Scopes | `scopes` | []string | Permission scopes |
| Device Type | `device_type` | string | Device type |
| Device ID | `device_id` | string | Device identifier |
| Organization | `organization_id` | string | Current organization ID |

### Default Token Lifetimes

| Token | Default TTL | Config Key |
|-------|-------------|------------|
| Access Token | 600 seconds (10 min) | `jwt.access_token_expire` |
| Refresh Token | 86400 seconds (24 hours) | `jwt.refresh_token_expire` |
