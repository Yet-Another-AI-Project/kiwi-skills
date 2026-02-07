# Domain Layer Patterns

## Aggregate Root

Aggregate roots are pure containers. They hold a KeyEntity, child entity lists, and business logic methods. They NEVER have direct data attributes.

```go
package aggregate

import (
    "myapp/domain/order/entity"
    "myapp/domain/order/vo"

    "github.com/futurxlab/golanggraph/xerror"
    "github.com/google/uuid"
)

type OrderAggregateRoot struct {
    Order    *entity.OrderEntity       // KeyEntity - holds all data
    Items    []*entity.OrderItemEntity // Child entities
}

// Business logic on aggregate root
func (r *OrderAggregateRoot) AddItem(item *entity.OrderItemEntity) error {
    if item.OrderID != r.Order.ID {
        return xerror.New("item does not belong to this order")
    }
    r.Items = append(r.Items, item)
    return nil
}

func (r *OrderAggregateRoot) CalculateTotal() float64 {
    var total float64
    for _, item := range r.Items {
        total += item.Price * float64(item.Quantity)
    }
    return total
}
```

## Entity

Entities hold data attributes and logic relevant to their own data.

```go
package entity

import (
    "myapp/domain/order/vo"
    "time"

    "github.com/google/uuid"
)

type OrderEntity struct {
    ID          uuid.UUID
    UserID      string
    Title       string
    Status      vo.OrderStatus
    TotalAmount float64
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

// Entity-level logic
func (e *OrderEntity) IsEditable() bool {
    return e.Status == vo.OrderStatusDraft
}

func (e *OrderEntity) SetStatus(status vo.OrderStatus) {
    e.Status = status
}
```

## Enum Pattern

Every enum MUST have a `GetAll*()` function. This is consumed by ent schemas via `utils.EnumToStrings()`.

```go
package vo

// OrderStatus represents order lifecycle states
type OrderStatus string

const (
    OrderStatusDraft     OrderStatus = "draft"
    OrderStatusPending   OrderStatus = "pending"
    OrderStatusCompleted OrderStatus = "completed"
    OrderStatusCancelled OrderStatus = "cancelled"
)

// GetAllOrderStatuses returns all possible order statuses
func GetAllOrderStatuses() []OrderStatus {
    return []OrderStatus{
        OrderStatusDraft,
        OrderStatusPending,
        OrderStatusCompleted,
        OrderStatusCancelled,
    }
}

// ParseOrderStatus validates and converts a string to OrderStatus
func ParseOrderStatus(s string) (OrderStatus, bool) {
    for _, status := range GetAllOrderStatuses() {
        if string(status) == s {
            return status, true
        }
    }
    return "", false
}
```

## Value Object Rules

- Do NOT create a VO with fewer than 3 fields
- 1-2 field concepts remain as primitives on the Entity
- 3+ field concepts are candidates for extraction into a VO
- VOs are stored as JSON in ent schemas, NOT as separate tables

```go
// CORRECT: 3+ fields -> VO
type ShippingAddress struct {
    Street  string `json:"street"`
    City    string `json:"city"`
    ZipCode string `json:"zip_code"`
    Country string `json:"country"`
}

// INCORRECT: 1-2 fields -> keep as primitives on Entity
// type Currency struct { Code string } -- DON'T do this
```

## Repository Interfaces (Triple Pattern)

Defined in `domain/{domain}/repository/interface.go`. Every domain aggregate gets three interfaces.

```go
package repository

import (
    "myapp/domain/common/contract"
    "myapp/domain/order/aggregate"
    "context"

    "github.com/google/uuid"
)

// IOrderRepositoryRead - read-only operations
type IOrderRepositoryRead interface {
    FindByID(ctx context.Context, id uuid.UUID) (*aggregate.OrderAggregateRoot, error)
    FindByUserID(ctx context.Context, userID string, limit, offset int) ([]*aggregate.OrderAggregateRoot, int, error)
}

// IOrderRepositoryWrite - write operations (MUST operate on AggregateRoot)
type IOrderRepositoryWrite interface {
    Save(ctx context.Context, root *aggregate.OrderAggregateRoot) (*aggregate.OrderAggregateRoot, error)
    Delete(ctx context.Context, id uuid.UUID, userID string) error
}

// IOrderRepository - full interface with transaction support
type IOrderRepository interface {
    IOrderRepositoryRead
    IOrderRepositoryWrite
    contract.ITransactionDecorator
}
```

Key rules:
- `Read` interface can return `*Entity` or `*AggregateRoot`
- `Write` interface MUST only operate on `*AggregateRoot`
- `IRepository` embeds Read + Write + `ITransactionDecorator`

## Transaction Contract

Defined in `domain/common/contract/transaction.go`:

```go
package contract

import "context"

type ITransactionDecorator interface {
    WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error
}
```

## Domain Service

Domain services encapsulate complex business logic spanning aggregates. Every domain service MUST have an interface.

```go
package service

import (
    "myapp/domain/order/aggregate"
    "myapp/domain/order/repository"
    "context"

    "github.com/futurxlab/golanggraph/logger"
    "github.com/futurxlab/golanggraph/xerror"
)

type IOrderService interface {
    CreateOrder(ctx context.Context, root *aggregate.OrderAggregateRoot) error
    CompleteOrder(ctx context.Context, root *aggregate.OrderAggregateRoot) error
}

type orderService struct {
    repo   repository.IOrderRepository
    logger logger.ILogger
}

func NewOrderService(repo repository.IOrderRepository, logger logger.ILogger) IOrderService {
    return &orderService{repo: repo, logger: logger}
}

func (s *orderService) CreateOrder(ctx context.Context, root *aggregate.OrderAggregateRoot) error {
    // Business logic
    root.Order.SetStatus(vo.OrderStatusPending)

    // Domain service CAN call repository write
    _, err := s.repo.Save(ctx, root)
    if err != nil {
        return xerror.Wrap(err)
    }
    return nil
}
```

## Domain Errors

Sentinel errors for domain-level conditions, in `domain/common/errors.go`:

```go
package common

import "github.com/futurxlab/golanggraph/xerror"

var (
    ErrNotFound     = xerror.New("not found")
    ErrUnauthorized = xerror.New("unauthorized")
)
```
