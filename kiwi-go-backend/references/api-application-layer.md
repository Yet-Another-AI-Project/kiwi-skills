# API and Application Layer Patterns

## API Layer

### Controller

A single `Controller` struct holds all injected application service interfaces. Each domain gets a separate file with handler methods.

```go
// api/controller/api/controller.go
package api

import (
    orderservice "myapp/application/service/order"
)

type Controller struct {
    orderService orderservice.IOrderApplicationService
}

func NewController(
    orderService orderservice.IOrderApplicationService,
) *Controller {
    return &Controller{
        orderService: orderService,
    }
}
```

```go
// api/controller/api/order_controller.go
package api

import (
    "myapp/api/dto"

    libfacade "github.com/Yet-Another-AI-Project/kiwi-lib/server/facade"
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

func (c *Controller) GetOrder(ctx *gin.Context, userID string) (*dto.OrderResponse, *libfacade.Error) {
    idStr := ctx.Param("id")
    id, err := uuid.Parse(idStr)
    if err != nil {
        return nil, libfacade.ErrBadRequest.Wrap(err)
    }

    result, appErr := c.orderService.GetOrder(ctx, userID, id)
    if appErr != nil {
        return nil, appErr
    }
    return result, nil
}

func (c *Controller) CreateOrder(ctx *gin.Context, userID string) (*dto.OrderResponse, *libfacade.Error) {
    var req dto.CreateOrderRequest
    if err := ctx.ShouldBindJSON(&req); err != nil {
        return nil, libfacade.ErrBadRequest.Wrap(err)
    }

    result, appErr := c.orderService.CreateOrder(ctx, userID, &req)
    if appErr != nil {
        return nil, appErr
    }
    return result, nil
}
```

### DTOs

Request and response models in `api/dto/`. Use `json` tags for serialization and `binding` tags for validation.

```go
// api/dto/order.go
package dto

type CreateOrderRequest struct {
    Title string `json:"title" binding:"required"`
    Items []OrderItemRequest `json:"items" binding:"required,min=1"`
}

type OrderItemRequest struct {
    ProductID string  `json:"product_id" binding:"required"`
    Quantity  int     `json:"quantity" binding:"required,min=1"`
    Price     float64 `json:"price" binding:"required"`
}

type OrderResponse struct {
    ID          string  `json:"id"`
    Title       string  `json:"title"`
    Status      string  `json:"status"`
    TotalAmount float64 `json:"total_amount"`
    CreatedAt   string  `json:"created_at"`
}
```

```go
// api/dto/base_response.go
package dto

type OperationResponse struct {
    Success bool `json:"success"`
}
```

### Handler Wrappers

Generic handler wrappers in `api/server/route/handler.go` provide consistent response formatting, panic recovery, and auth extraction.

```go
package route

import (
    "fmt"
    "net/http"
    "runtime/debug"

    "myapp/api/server/middleware"

    libfacade "github.com/Yet-Another-AI-Project/kiwi-lib/server/facade"
    "github.com/futurxlab/golanggraph/logger"
    "github.com/gin-gonic/gin"
)

// NormalHandler - no auth required, returns data + error
func NormalHandler[T any](f func(*gin.Context) (T, *libfacade.Error)) func(*gin.Context) {
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

// RequireUserHandler - auth required, extracts userID from context
func RequireUserHandler[T any](f func(*gin.Context, string) (T, *libfacade.Error)) func(*gin.Context) {
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

// EventStreamHandler - SSE without auth
func EventStreamHandler(f func(*gin.Context) *libfacade.Error) func(*gin.Context) { /* similar pattern */ }

// EventStreamRequireUserHandler - SSE with auth
func EventStreamRequireUserHandler(f func(*gin.Context, string) *libfacade.Error) func(*gin.Context) { /* similar pattern */ }

func responseError(c *gin.Context, err *libfacade.Error) {
    response := &libfacade.BaseResponse{}
    path := c.Request.URL.Path
    futurxLogger, ok := c.Get("logger")
    if ok {
        futurxLogger.(logger.ILogger).Errorf(c, "Gin Request Failed, path: %s, facade_message: %s, internal_error: %v", path, err.FacadeMessage, err.InternalError)
    }
    response.Status = "error"
    response.Error = err
    c.AbortWithStatusJSON(err.StatusCode(), response)
}
```

### Route Registration

```go
// api/server/route/api.go
package route

func (r *Route) RegisterRoutes(router *gin.Engine) {
    apiGroup := router.Group("/api/v1")

    // Public routes
    publicGroup := apiGroup.Group("")
    {
        publicGroup.GET("/health", NormalHandler(r.controller.HealthCheck))
    }

    // Auth-required routes
    authGroup := apiGroup.Group("")
    authGroup.Use(r.authMiddleware.Handle())
    {
        authGroup.GET("/orders/:id", RequireUserHandler(r.controller.GetOrder))
        authGroup.POST("/orders", RequireUserHandler(r.controller.CreateOrder))
    }
}
```

### Server Setup

```go
// api/server/api_server.go
package server

import (
    "myapp/config"

    libgin "github.com/Yet-Another-AI-Project/kiwi-lib/server/gin"
    "github.com/futurxlab/golanggraph/logger"
    "go.uber.org/fx"
)

func NewServer(lc fx.Lifecycle, config *config.Config, route *route.Route, logger logger.ILogger) {
    ginServer := libgin.NewGin(
        libgin.WithPort(config.Server.Port),
        libgin.WithLogger(logger),
    )
    route.RegisterRoutes(ginServer.Engine())

    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            go ginServer.Start()
            return nil
        },
        OnStop: func(ctx context.Context) error {
            return ginServer.Shutdown(ctx)
        },
    })
}
```

---

## Application Layer

### Application Service

Application services orchestrate domain services. They define their own interface.

```go
// application/service/order/service.go
package order

import (
    "myapp/api/dto"
    apperror "myapp/application/service/error"
    "myapp/domain/order/aggregate"
    "myapp/domain/order/entity"
    "myapp/domain/order/repository"
    domainservice "myapp/domain/order/service"
    "context"

    libfacade "github.com/Yet-Another-AI-Project/kiwi-lib/server/facade"
    "github.com/futurxlab/golanggraph/logger"
    "github.com/google/uuid"
)

type IOrderApplicationService interface {
    GetOrder(ctx context.Context, userID string, id uuid.UUID) (*dto.OrderResponse, *libfacade.Error)
    CreateOrder(ctx context.Context, userID string, req *dto.CreateOrderRequest) (*dto.OrderResponse, *libfacade.Error)
}

type orderApplicationService struct {
    orderRepoRead repository.IOrderRepositoryRead   // Read-only repo
    orderService  domainservice.IOrderService        // Domain service for writes
    logger        logger.ILogger
}

func NewOrderApplicationService(
    orderRepoRead repository.IOrderRepositoryRead,
    orderService domainservice.IOrderService,
    logger logger.ILogger,
) IOrderApplicationService {
    return &orderApplicationService{
        orderRepoRead: orderRepoRead,
        orderService:  orderService,
        logger:        logger,
    }
}

func (s *orderApplicationService) GetOrder(ctx context.Context, userID string, id uuid.UUID) (*dto.OrderResponse, *libfacade.Error) {
    // Application CAN use RepositoryRead for queries
    root, err := s.orderRepoRead.FindByID(ctx, id)
    if err != nil {
        return nil, apperror.ErrOrderNotFound
    }
    if root.Order.UserID != userID {
        return nil, apperror.ErrForbidden
    }
    return s.toResponse(root), nil
}

func (s *orderApplicationService) CreateOrder(ctx context.Context, userID string, req *dto.CreateOrderRequest) (*dto.OrderResponse, *libfacade.Error) {
    // Build aggregate root
    root := &aggregate.OrderAggregateRoot{
        Order: &entity.OrderEntity{
            UserID: userID,
            Title:  req.Title,
        },
    }

    // Application MUST use Domain Service for writes
    if err := s.orderService.CreateOrder(ctx, root); err != nil {
        s.logger.Errorf(ctx, "Failed to create order: %v", err)
        return nil, libfacade.ErrServerInternal.Wrap(err)
    }

    return s.toResponse(root), nil
}

func (s *orderApplicationService) toResponse(root *aggregate.OrderAggregateRoot) *dto.OrderResponse {
    return &dto.OrderResponse{
        ID:          root.Order.ID.String(),
        Title:       root.Order.Title,
        Status:      string(root.Order.Status),
        TotalAmount: root.Order.TotalAmount,
        CreatedAt:   root.Order.CreatedAt.Format("2006-01-02T15:04:05Z"),
    }
}
```

### Application Errors

Translate domain concerns to HTTP-friendly errors using `libfacade`:

```go
// application/service/error/error.go
package error

import (
    libfacade "github.com/Yet-Another-AI-Project/kiwi-lib/server/facade"
)

var (
    ErrOrderNotFound = libfacade.NewNotFoundError("order not found")
    ErrForbidden     = libfacade.NewForbiddenError("access denied")
    ErrUnauthorized  = libfacade.NewUnauthorizedError("unauthorized")
)
```

### Key Rules

1. Application layer can inject `IOrderRepositoryRead` for queries
2. Application layer CANNOT inject `IOrderRepositoryWrite` -- writes go through Domain Service
3. Application layer catches domain errors and translates to `*libfacade.Error`
4. Application layer defines its own response format (or reuses API DTOs for simple cases)
5. API layer calls application service and returns the result -- no business logic
