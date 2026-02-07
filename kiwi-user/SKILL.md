---
name: kiwi-user
description: Kiwi-user authentication integration for frontend (login flows, token storage, refresh) and backend (token verification via API or public key, auth middleware, user ID extraction).
---

# Kiwi-User Authentication Integration

This skill defines how to integrate with the kiwi-user authentication service from both frontend (Next.js/React) and backend (Go/Gin) applications.

## Architecture Overview

```
Frontend (Next.js)                     Kiwi-User Service                    Backend (Go/Gin)
     |                                       |                                      |
     |---(1) OAuth/login ------------------>|                                      |
     |<--(2) {access_token, refresh_token}--|                                      |
     |                                       |                                      |
     |    [Store tokens in localStorage]     |                                      |
     |                                       |                                      |
     |---(3) API request (Bearer token) -----|-------------------------------------->|
     |                                       |<---(4) POST /v1/token/verify --------|
     |                                       |----(5) {success, user_info} -------->|
     |                                       |                                      |
     |---(6) POST /v1/token/refresh ------->| [When access token near expiry]      |
     |<--(7) New tokens --------------------|                                      |
```

Kiwi-user is the central auth service. Frontends authenticate users and store tokens. Backends verify tokens on every request via kiwi-user's API (or optionally via public key).

## JWT Token Structure

Kiwi-user issues RS256-signed JWTs. The access token payload contains:

| Field | JWT Claim | Type | Description |
|-------|-----------|------|-------------|
| UserID | `sub` | string | User's unique ID |
| Application | `iss` | string | Application name (issuer) |
| PersonalRole | `roles` | string | User's role |
| Scopes | `scopes` | []string | Permission scopes |
| DeviceType | `device_type` | string | Device type |
| DeviceID | `device_id` | string | Device identifier |
| OrganizationID | `organization_id` | string | Current org ID |
| Create | `iat` | int64 | Issued at (unix seconds) |
| Expire | `exp` | int64 | Expiration (unix seconds) |

Default expiration: access token = 600s (10 min), refresh token = 86400s (24 hours).

## Environment Variables

### Frontend (Next.js)

| Variable | Purpose | Example |
|----------|---------|---------|
| `NEXT_PUBLIC_KIWI_USER_API_BASE_URL` | Kiwi-user API base URL | `https://user.example.com` |
| `NEXT_PUBLIC_APPLICATION_NAME` | Application name for login requests | `my-app` |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Google OAuth client ID | `xxx.apps.googleusercontent.com` |

### Backend (Go)

| Config Field | Purpose | Example |
|-------------|---------|---------|
| `cfg.User.BaseURL` | Kiwi-user API base URL | `https://user.example.com` |

## Frontend Integration

See [references/frontend-auth.md](references/frontend-auth.md) for complete code patterns.

### Login Flows

Kiwi-user supports multiple login methods. All return the same `LoginResponse`:

```typescript
interface LoginResponse {
  refresh_token: string;
  refresh_token_expires_at: number;  // unix seconds
  access_token: string;
  access_token_expires_at: number;   // unix seconds
  type: string;
  device_type: string;
  device_id: string;
  user_id: string;
}
```

| Method | Endpoint | Flow |
|--------|----------|------|
| Google OAuth (Web) | `POST /v1/login/google/web` | Redirect to Google -> callback with code -> exchange for tokens |
| Email | `POST /v1/login/email` | Send verification code -> `POST /v1/login/email/verify_code` |
| Phone | `POST /v1/login/phone` | Send verification code -> `POST /v1/login/phone/verify_code` |
| WeChat Mini Program | `POST /v1/login/wechat/miniprogram` | WeChat SDK login |
| WeChat Web | `POST /v1/login/wechat/web` | WeChat OAuth flow |
| Password | `POST /v1/login/password` | Direct username/password |

### Token Storage

Store all auth data in `localStorage` with these keys:

| Key | Value |
|-----|-------|
| `kiwi_access_token` | JWT access token string |
| `kiwi_refresh_token` | Refresh token string |
| `kiwi_access_token_expires_at` | Expiration (unix seconds, string) |
| `kiwi_refresh_token_expires_at` | Expiration (unix seconds, string) |
| `kiwi_user_id` | User ID string |
| `kiwi_device_type` | Device type string |
| `kiwi_device_id` | Device identifier string |

### Token Refresh Strategy

Access tokens expire quickly (default 10 min). Refresh BEFORE expiration using a buffer:

```typescript
function isAccessTokenExpired(): boolean {
  const expiresAt = localStorage.getItem("kiwi_access_token_expires_at");
  if (!expiresAt) return true;
  const bufferSeconds = 60; // 1-minute buffer
  return Date.now() / 1000 >= parseInt(expiresAt) - bufferSeconds;
}
```

The `getAccessToken()` function auto-refreshes transparently:

```typescript
async function getAccessToken(): Promise<string | null> {
  if (!isAccessTokenExpired()) {
    return localStorage.getItem("kiwi_access_token");
  }
  // Refresh the token
  const refreshed = await refreshAccessToken();
  if (!refreshed) {
    clearAuthData(); // Refresh token also expired
    return null;
  }
  return localStorage.getItem("kiwi_access_token");
}
```

Refresh calls `POST /v1/token/refresh` with `{user_id, refresh_token, device}`. Both tokens are replaced.

### API Client Pattern

Every authenticated API request MUST use `getAccessToken()` and set the `Authorization` header:

```typescript
async function apiRequest<T>(path: string, options: RequestInit = {}): Promise<T> {
  const token = await getAccessToken();
  if (!token) {
    window.location.href = "/"; // Redirect to login
    throw new Error("Not authenticated");
  }
  const response = await fetch(`${API_BASE_URL}${path}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
      ...options.headers,
    },
  });
  if (response.status === 401) {
    // Token rejected by backend - clear and redirect
    clearAuthData();
    window.location.href = "/";
    throw new Error("Unauthorized");
  }
  const data = await response.json();
  return data.data as T;
}
```

### React Auth Hook

Use a `useAuth` hook and `AuthProvider` context to manage auth state:

```typescript
function useAuth() {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [user, setUser] = useState<UserInfo | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    async function checkAuth() {
      const authed = isAuthenticated(); // checks refresh token validity
      setIsAuthenticated(authed);
      if (authed) {
        const userInfo = await getUserInfo();
        setUser(userInfo);
      }
      setIsLoading(false);
    }
    checkAuth();
  }, []);

  return { isAuthenticated, isLoading, user, logout };
}
```

### User Info Fetching

`GET /v1/user/info` with Bearer token. Implement request deduplication and caching:

```typescript
let userInfoPromise: Promise<UserInfo> | null = null;
let cachedUserInfo: { data: UserInfo; timestamp: number } | null = null;
const CACHE_DURATION = 5000; // 5 seconds

async function getUserInfo(): Promise<UserInfo> {
  // Return cached if fresh
  if (cachedUserInfo && Date.now() - cachedUserInfo.timestamp < CACHE_DURATION) {
    return cachedUserInfo.data;
  }
  // Deduplicate concurrent requests
  if (userInfoPromise) return userInfoPromise;
  userInfoPromise = fetchUserInfo();
  try {
    const info = await userInfoPromise;
    cachedUserInfo = { data: info, timestamp: Date.now() };
    return info;
  } finally {
    userInfoPromise = null;
  }
}
```

On 401 response: retry once after refreshing access token. If still 401, logout.

## Backend Integration

See [references/backend-auth.md](references/backend-auth.md) for complete code patterns.

### Method 1: Token Verification via API (Recommended)

Call kiwi-user's `POST /v1/token/verify` endpoint:

```go
type Client struct {
    baseURL    string
    httpClient *http.Client
}

func (c *Client) VerifyToken(ctx context.Context, token string) (string, error) {
    reqBody := map[string]string{"access_token": token}
    body, _ := json.Marshal(reqBody)

    req, _ := http.NewRequestWithContext(ctx, "POST",
        c.baseURL+"/v1/token/verify", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.httpClient.Do(req)
    // ... error handling, parse response ...

    // Response: {status: "success", data: {success: true, user_info: {id, name, ...}}}
    return response.Data.UserInfo.ID, nil
}
```

### Method 2: Public Key Verification (Self-Contained)

Fetch the public key from `GET /v1/token/publickey` and verify the JWT locally:

```go
// 1. Fetch public key (cache it - it rarely changes)
// GET /v1/token/publickey -> {public_key: "-----BEGIN PUBLIC KEY-----\n..."}

// 2. Parse and verify the JWT
// Split token into head.payload.signature
// Verify signature with RSA public key + SHA256
// Decode payload: base64url -> JSON -> AccessPayload
// Check exp > now

// 3. Extract user info from payload
// payload.sub = user ID
// payload.iss = application
// payload.roles = personal role
// payload.organization_id = org ID
```

This avoids a network call per request but requires managing key rotation.

### Auth Middleware Pattern (Gin)

```go
const UserIDKey = "user_id"

func AuthMiddleware(kiwiUserClient *kiwiuser.Client) gin.HandlerFunc {
    return func(c *gin.Context) {
        auth := c.GetHeader("Authorization")
        parts := strings.SplitN(auth, " ", 2)
        if len(parts) != 2 || strings.ToLower(parts[0]) != "bearer" {
            c.AbortWithStatusJSON(401, BaseResponse{Status: "error", Error: "missing token"})
            return
        }

        userID, err := kiwiUserClient.VerifyToken(c.Request.Context(), parts[1])
        if err != nil {
            c.AbortWithStatusJSON(401, BaseResponse{Status: "error", Error: err})
            return
        }

        c.Set(UserIDKey, userID)
        c.Next()
    }
}
```

### Extracting User ID in Handlers

Use the `RequireUserHandler` pattern to get a type-safe user ID:

```go
// Handler wrapper that extracts user ID
func RequireUserHandler[T any](f func(*gin.Context, string) (T, *facade.Error)) gin.HandlerFunc {
    return func(c *gin.Context) {
        userID, exists := c.Get(middleware.UserIDKey)
        if !exists {
            c.AbortWithStatusJSON(403, BaseResponse{Status: "error", Error: "user ID not found"})
            return
        }
        userIDStr := userID.(string)
        data, err := f(c, userIDStr)
        if err != nil {
            responseError(c, err)
            return
        }
        c.JSON(200, BaseResponse{Status: "success", Data: data})
    }
}

// Usage in route registration
router.GET("/v1/profile", RequireUserHandler(handler.GetProfile))

// Handler receives userID directly
func (h *Handler) GetProfile(c *gin.Context, userID string) (*Profile, *facade.Error) {
    return h.service.GetProfile(c, userID)
}
```

### Route Registration Pattern

Apply auth middleware to route groups:

```go
func RegisterRoutes(router *gin.Engine, authMW gin.HandlerFunc, handler *Handler) {
    v1 := router.Group("/v1")

    // Public routes (no auth)
    public := v1.Group("/public")
    public.GET("/health", handler.Health)

    // Authenticated routes
    authed := v1.Group("")
    authed.Use(authMW)
    authed.GET("/profile", RequireUserHandler(handler.GetProfile))
    authed.POST("/settings", RequireUserHandler(handler.UpdateSettings))
}
```

### Local Development

In local environment, skip token verification and use a test user:

```go
if cfg.Server.Env == "local" {
    c.Set(UserIDKey, "test_user")
    c.Next()
    return
}
```

## Logout Flow

### Frontend

```typescript
async function logout(): Promise<void> {
  const userId = localStorage.getItem("kiwi_user_id");
  const refreshToken = localStorage.getItem("kiwi_refresh_token");
  // Notify kiwi-user to invalidate tokens
  await fetch(`${KIWI_USER_API_BASE_URL}/v1/user/logout`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      user_id: userId,
      refresh_token: refreshToken,
      device: { device_type: getDeviceType(), device_id: getDeviceId() },
    }),
  });
  // Clear all local auth data
  clearAuthData();
}
```

## API Reference

See [references/kiwi-user-api.md](references/kiwi-user-api.md) for complete endpoint documentation with request/response types.

## Checklist

Before shipping auth integration, verify:

- [ ] `NEXT_PUBLIC_KIWI_USER_API_BASE_URL` and `NEXT_PUBLIC_APPLICATION_NAME` are configured
- [ ] Tokens stored in localStorage with correct keys (`kiwi_access_token`, etc.)
- [ ] Access token refresh uses 1-minute buffer before actual expiration
- [ ] `getAccessToken()` called before every API request (auto-refreshes)
- [ ] API client sets `Authorization: Bearer <token>` header
- [ ] 401 responses trigger token refresh retry, then logout on second failure
- [ ] `getUserInfo()` has request deduplication and short-lived cache
- [ ] Backend auth middleware extracts Bearer token from Authorization header
- [ ] Backend calls `POST /v1/token/verify` (or uses public key verification)
- [ ] User ID set in gin context with `c.Set("user_id", userID)`
- [ ] Handlers use `RequireUserHandler` pattern for type-safe user ID access
- [ ] Local dev mode skips verification with `test_user`
- [ ] Logout calls `/v1/user/logout` AND clears all localStorage keys
