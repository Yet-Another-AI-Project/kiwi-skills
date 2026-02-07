# Backend Auth Integration

Complete patterns for verifying kiwi-user tokens and extracting user identity in a Go/Gin backend.

## Kiwi-User Client

The client communicates with kiwi-user service for token verification and user info lookups.

### Client Structure

```go
package kiwiuser

import (
    "bytes"
    "context"
    "encoding/json"
    "net/http"

    "github.com/futurxlab/golanggraph/logger"
    "github.com/futurxlab/golanggraph/xerror"
)

type Client struct {
    baseURL    string
    httpClient *http.Client
    logger     logger.ILogger
}

func NewClient(cfg *config.Config, httpClient *http.Client, logger logger.ILogger) *Client {
    return &Client{
        baseURL:    cfg.User.BaseURL, // e.g., "https://user.example.com"
        httpClient: httpClient,
        logger:     logger,
    }
}
```

### Token Verification (API Method)

This is the primary verification method. Calls kiwi-user's `/v1/token/verify` endpoint:

```go
type VerifyTokenRequest struct {
    AccessToken string `json:"access_token"`
}

type VerifyTokenResponse struct {
    Success  bool      `json:"success"`
    UserInfo *UserInfo `json:"user_info"`
}

type UserInfo struct {
    UserID      string `json:"id"`
    Application string `json:"application"`
    Name        string `json:"name"`
    DisplayName string `json:"display_name"`
    Avatar      string `json:"avatar"`
    Username    string `json:"username"`
    Department  string `json:"department"`
}

func (c *Client) VerifyToken(ctx context.Context, token string) (string, error) {
    reqBody := VerifyTokenRequest{AccessToken: token}
    body, err := json.Marshal(reqBody)
    if err != nil {
        return "", xerror.Wrap(err)
    }

    req, err := http.NewRequestWithContext(ctx, "POST",
        c.baseURL+"/v1/token/verify", bytes.NewReader(body))
    if err != nil {
        return "", xerror.Wrap(err)
    }
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return "", xerror.Wrap(err)
    }
    defer resp.Body.Close()

    // Response format: {status: "success", data: {success: true, user_info: {id: "..."}}}
    var result struct {
        Status string              `json:"status"`
        Data   VerifyTokenResponse `json:"data"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return "", xerror.Wrap(err)
    }

    if !result.Data.Success || result.Data.UserInfo == nil {
        return "", xerror.New("token verification failed")
    }

    return result.Data.UserInfo.UserID, nil
}
```

### Batch User Info Lookup (Internal API)

For fetching user info for multiple users (e.g., displaying names in a list):

```go
func (c *Client) InternalGetUserInfos(ctx context.Context, userIDs []string) ([]*UserInfo, error) {
    reqBody := map[string][]string{"user_ids": userIDs}
    body, err := json.Marshal(reqBody)
    if err != nil {
        return nil, xerror.Wrap(err)
    }

    req, err := http.NewRequestWithContext(ctx, "POST",
        c.baseURL+"/internal/user/infos", bytes.NewReader(body))
    if err != nil {
        return nil, xerror.Wrap(err)
    }
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, xerror.Wrap(err)
    }
    defer resp.Body.Close()

    var result struct {
        Status string      `json:"status"`
        Data   []*UserInfo `json:"data"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, xerror.Wrap(err)
    }

    return result.Data, nil
}
```

## Public Key Verification (Self-Contained Method)

Instead of calling the verify API per request, you can fetch the public key once and verify JWTs locally. This is how kiwi-user itself verifies tokens internally.

### JWT Structure

Kiwi-user uses RS256 (RSA + SHA256) for signing JWTs. The token format is `{base64url_header}.{base64url_payload}.{base64url_signature}`.

### Access Token Payload

```go
type AccessPayload struct {
    Type           string   `json:"type"`       // "access"
    Create         int64    `json:"iat"`        // issued at (unix seconds)
    Expire         int64    `json:"exp"`        // expiration (unix seconds)
    UserID         string   `json:"sub"`        // user ID
    Application    string   `json:"iss"`        // application name (issuer)
    PersonalRole   string   `json:"roles"`      // user's role
    Scopes         []string `json:"scopes"`     // permission scopes
    DeviceType     string   `json:"device_type"`
    DeviceID       string   `json:"device_id"`
    OrganizationID string   `json:"organization_id"`
}
```

### Fetching the Public Key

```go
func (c *Client) GetPublicKey(ctx context.Context) (string, error) {
    req, err := http.NewRequestWithContext(ctx, "GET",
        c.baseURL+"/v1/token/publickey", nil)
    if err != nil {
        return "", xerror.Wrap(err)
    }

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return "", xerror.Wrap(err)
    }
    defer resp.Body.Close()

    // Response: {status: "success", data: {public_key: "-----BEGIN PUBLIC KEY-----\n..."}}
    var result struct {
        Status string `json:"status"`
        Data   struct {
            PublicKey string `json:"public_key"`
        } `json:"data"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return "", xerror.Wrap(err)
    }

    return result.Data.PublicKey, nil
}
```

### Local JWT Verification

The verification flow kiwi-user uses internally:

```go
import (
    "crypto"
    "crypto/rsa"
    "crypto/sha256"
    "crypto/x509"
    b64 "encoding/base64"
    "encoding/json"
    "encoding/pem"
    "strings"
    "time"
)

func VerifyJWTLocally(token string, publicKeyPEM string) (*AccessPayload, error) {
    // 1. Split token into parts
    parts := strings.Split(token, ".")
    if len(parts) != 3 {
        return nil, fmt.Errorf("invalid JWT format")
    }

    // 2. Parse public key
    block, _ := pem.Decode([]byte(publicKeyPEM))
    pubKey, err := x509.ParsePKIXPublicKey(block.Bytes)
    if err != nil {
        return nil, fmt.Errorf("invalid public key: %w", err)
    }
    rsaPubKey := pubKey.(*rsa.PublicKey)

    // 3. Verify signature
    signedContent := []byte(parts[0] + "." + parts[1])
    signature, err := b64.RawURLEncoding.DecodeString(parts[2])
    if err != nil {
        return nil, fmt.Errorf("invalid signature encoding: %w", err)
    }

    hash := sha256.Sum256(signedContent)
    if err := rsa.VerifyPKCS1v15(rsaPubKey, crypto.SHA256, hash[:], signature); err != nil {
        return nil, fmt.Errorf("invalid signature: %w", err)
    }

    // 4. Decode payload
    payloadBytes, err := b64.RawURLEncoding.DecodeString(parts[1])
    if err != nil {
        return nil, fmt.Errorf("invalid payload encoding: %w", err)
    }

    var payload AccessPayload
    if err := json.Unmarshal(payloadBytes, &payload); err != nil {
        return nil, fmt.Errorf("invalid payload: %w", err)
    }

    // 5. Check expiration
    if time.Unix(payload.Expire, 0).Before(time.Now()) {
        return nil, fmt.Errorf("token expired")
    }

    // 6. Optionally check issuer/role
    // if payload.Application != expectedApp { return nil, fmt.Errorf("invalid issuer") }

    return &payload, nil
}
```

### Caching the Public Key

The public key rarely changes. Cache it with a TTL:

```go
type PublicKeyCache struct {
    key       string
    fetchedAt time.Time
    ttl       time.Duration
    mu        sync.RWMutex
}

func (c *PublicKeyCache) Get(ctx context.Context, client *Client) (string, error) {
    c.mu.RLock()
    if c.key != "" && time.Since(c.fetchedAt) < c.ttl {
        defer c.mu.RUnlock()
        return c.key, nil
    }
    c.mu.RUnlock()

    c.mu.Lock()
    defer c.mu.Unlock()

    // Double-check after acquiring write lock
    if c.key != "" && time.Since(c.fetchedAt) < c.ttl {
        return c.key, nil
    }

    key, err := client.GetPublicKey(ctx)
    if err != nil {
        return "", err
    }
    c.key = key
    c.fetchedAt = time.Now()
    return key, nil
}
```

## Auth Middleware

### Standard Auth Middleware (API Verification)

```go
package middleware

import (
    "strings"

    libfacade "github.com/Yet-Another-AI-Project/kiwi-lib/server/facade"
    "github.com/gin-gonic/gin"
)

const UserIDKey = "user_id"

func AuthMiddleware(
    logger logger.ILogger,
    cfg *config.Config,
    kiwiuser *kiwiuser.Client,
) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Skip auth in local development
        if cfg.Server.Env == "local" {
            c.Set(UserIDKey, "test_user")
            c.Next()
            return
        }

        // Extract Bearer token
        auth := c.GetHeader("Authorization")
        parts := strings.SplitN(auth, " ", 2)
        if len(parts) != 2 || strings.ToLower(parts[0]) != "bearer" {
            c.AbortWithStatusJSON(401, &libfacade.BaseResponse{
                Status: "error",
                Error:  libfacade.ErrUnauthorized,
            })
            return
        }

        // Verify with kiwi-user service
        userID, err := kiwiuser.VerifyToken(c.Request.Context(), parts[1])
        if err != nil {
            logger.Errorf(c, "Token verification failed: %v", err)
            c.AbortWithStatusJSON(401, &libfacade.BaseResponse{
                Status: "error",
                Error:  err,
            })
            return
        }

        // Set user ID in context for downstream handlers
        c.Set(UserIDKey, userID)
        c.Next()
    }
}
```

### Kiwi-User's Own Auth Middleware (Public Key Method)

This is how kiwi-user itself protects its authenticated routes (e.g., `/v1/user/info`). It uses local JWT verification with the private/public key pair:

```go
func NewKiwiUserAuth(application, role string, jwtHelper *jwt.JWTHelper) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Extract token from cookie or Authorization header
        token, err := getAccessToken(c.Request)
        if err != nil {
            responseError(c, err)
            return
        }

        // Verify JWT signature with RS256
        jwtToken, err := jwtHelper.VerifyRS256JWT(token)
        if err != nil {
            responseError(c, facade.ErrUnauthorized)
            return
        }

        // Decode payload
        payload, err := jwtHelper.DecodeAccessPayload(jwtToken.Payload)
        if err != nil {
            responseError(c, facade.ErrUnauthorized)
            return
        }

        // Check expiration
        if time.Unix(payload.Expire, 0).Before(time.Now()) {
            responseError(c, facade.ErrUnauthorized)
            return
        }

        // Check issuer and role (optional)
        if application != "" {
            if payload.Application != application || payload.PersonalRole != role {
                responseError(c, facade.ErrUnauthorized)
                return
            }
        }

        // Set user context
        c.Set("user_id", payload.UserID)
        c.Set("org_id", payload.OrganizationID)
        c.Next()
    }
}

// Token extraction: supports both cookie and Authorization header
func getAccessToken(r *http.Request) (string, error) {
    // Try cookie first
    if cookie, err := r.Cookie("access_token"); err == nil {
        return cookie.Value, nil
    }
    // Fall back to Authorization header (Bearer or JWT prefix)
    auth := strings.Fields(r.Header.Get("Authorization"))
    if len(auth) != 2 {
        return "", facade.ErrUnauthorized
    }
    prefix := strings.ToLower(auth[0])
    if prefix != "jwt" && prefix != "bearer" {
        return "", facade.ErrUnauthorized
    }
    return auth[1], nil
}
```

## Handler Patterns

### RequireUserHandler

A generic handler wrapper that extracts the user ID from gin context and passes it to the handler function. Handles the type assertion and error cases:

```go
func RequireUserHandler[T any](f func(*gin.Context, string) (T, *libfacade.Error)) gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if v := recover(); v != nil {
                err := libfacade.ErrServerInternal.Wrap(fmt.Errorf("%v\n%s", v, debug.Stack()))
                responseError(c, err)
            }
        }()

        userID, exists := c.Get(middleware.UserIDKey)
        if !exists {
            err := libfacade.NewForbiddenError("User ID not found in context")
            c.AbortWithStatusJSON(err.StatusCode(), &libfacade.BaseResponse{
                Status: "error",
                Error:  err,
            })
            return
        }

        userIDStr, ok := userID.(string)
        if !ok {
            err := libfacade.ErrServerInternal.Wrap(fmt.Errorf("invalid user ID type"))
            responseError(c, err)
            return
        }

        data, err := f(c, userIDStr)
        if err != nil {
            responseError(c, err)
            return
        }

        c.JSON(http.StatusOK, &libfacade.BaseResponse{
            Status: "success",
            Data:   data,
        })
    }
}
```

### EventStreamRequireUserHandler

Same pattern for SSE streaming endpoints:

```go
func EventStreamRequireUserHandler(f func(*gin.Context, string) *libfacade.Error) gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if v := recover(); v != nil {
                err := libfacade.ErrServerInternal.Wrap(fmt.Errorf("%v\n%s", v, debug.Stack()))
                responseError(c, err)
            }
        }()

        userID, exists := c.Get(middleware.UserIDKey)
        if !exists {
            err := libfacade.NewForbiddenError("User ID not found in context")
            c.AbortWithStatusJSON(err.StatusCode(), &libfacade.BaseResponse{
                Status: "error",
                Error:  err,
            })
            return
        }

        userIDStr := userID.(string)
        if ferr := f(c, userIDStr); ferr != nil {
            responseError(c, ferr)
        }
    }
}
```

### NormalHandler (No Auth Required)

For public endpoints that do not need authentication:

```go
func NormalHandler[T any](f func(*gin.Context) (T, *libfacade.Error)) gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if v := recover(); v != nil {
                err := libfacade.ErrServerInternal.Wrap(fmt.Errorf("%v\n%s", v, debug.Stack()))
                responseError(c, err)
            }
        }()

        data, err := f(c)
        if err != nil {
            responseError(c, err)
            return
        }

        c.JSON(http.StatusOK, &libfacade.BaseResponse{
            Status: "success",
            Data:   data,
        })
    }
}
```

## Route Registration

### Route Groups with Middleware

```go
func RegisterAPIV1(
    router *gin.Engine,
    authMiddleware gin.HandlerFunc,
    handler *api.Handler,
) {
    v1 := router.Group("/v1")

    // Public routes - no auth middleware
    public := v1.Group("/public")
    {
        public.GET("/health", NormalHandler(handler.Health))
    }

    // Authenticated routes
    authed := v1.Group("")
    authed.Use(authMiddleware)
    {
        // Profile
        authed.GET("/profile", RequireUserHandler(handler.GetProfile))
        authed.PUT("/profile", RequireUserHandler(handler.UpdateProfile))

        // Resources
        authed.GET("/items", RequireUserHandler(handler.ListItems))
        authed.POST("/items", RequireUserHandler(handler.CreateItem))

        // Streaming
        authed.POST("/chat/stream", EventStreamRequireUserHandler(handler.ChatStream))
    }
}
```

## Response Format

All responses follow the kiwi-lib `BaseResponse` format:

```go
// Success
{
    "status": "success",
    "data": { ... }
}

// Error
{
    "status": "error",
    "error": {
        "message": "...",
        "code": 401
    }
}
```

```go
func responseError(c *gin.Context, err *libfacade.Error) {
    response := &libfacade.BaseResponse{
        Status: "error",
        Error:  err,
    }
    c.AbortWithStatusJSON(err.StatusCode(), response)
}
```
