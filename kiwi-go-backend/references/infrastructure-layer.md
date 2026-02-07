# Infrastructure Layer Patterns

## Repository Implementation

Every repository struct embeds `transactionDecorator` and `clientGetter`. All database methods use `withDbClient`.

### Repository Struct

```go
package repository

import (
    "myapp/domain/order/aggregate"
    "myapp/domain/order/entity"
    "myapp/domain/order/repository"
    "myapp/infrastructure/repository/ent"
    "myapp/infrastructure/repository/ent/order"
    "context"

    "github.com/futurxlab/golanggraph/logger"
    "github.com/futurxlab/golanggraph/xerror"
    "github.com/google/uuid"
)

var (
    OrderNotFoundError = xerror.New("order not found")
)

type orderRepository struct {
    dbClient *ent.Client
    logger   logger.ILogger
    *transactionDecorator
    *clientGetter
}

func NewOrderRepository(
    dbClient *ent.Client,
    logger logger.ILogger,
) repository.IOrderRepository {
    return &orderRepository{
        dbClient:             dbClient,
        logger:               logger,
        transactionDecorator: &transactionDecorator{dbClient: dbClient, logger: logger},
        clientGetter:         &clientGetter{dbClient: dbClient},
    }
}

// WithTransaction delegates to transactionDecorator
func (r *orderRepository) WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error {
    return r.transactionDecorator.WithTransaction(ctx, fn)
}
```

### Read Methods (withDbClient pattern)

```go
func (r *orderRepository) FindByID(ctx context.Context, id uuid.UUID) (*aggregate.OrderAggregateRoot, error) {
    var root *aggregate.OrderAggregateRoot
    err := r.clientGetter.withDbClient(ctx, func(ctx context.Context, dbClient *ent.Client) error {
        entItem, err := dbClient.Order.Query().
            Where(order.ID(id)).
            Only(ctx)
        if err != nil {
            if ent.IsNotFound(err) {
                return OrderNotFoundError
            }
            return xerror.Wrap(err)
        }
        root = r.entToDomain(entItem)
        return nil
    })
    return root, err
}

func (r *orderRepository) FindByUserID(
    ctx context.Context,
    userID string,
    limit, offset int,
) ([]*aggregate.OrderAggregateRoot, int, error) {
    var roots []*aggregate.OrderAggregateRoot
    var total int
    err := r.clientGetter.withDbClient(ctx, func(ctx context.Context, dbClient *ent.Client) error {
        query := dbClient.Order.Query().
            Where(order.UserID(userID))

        count, err := query.Clone().Count(ctx)
        if err != nil {
            return xerror.Wrap(err)
        }
        total = count

        entItems, err := query.
            Order(order.ByCreatedAt(sql.OrderDesc())).
            Limit(limit).
            Offset(offset).
            All(ctx)
        if err != nil {
            return xerror.Wrap(err)
        }

        roots = make([]*aggregate.OrderAggregateRoot, 0, len(entItems))
        for _, entItem := range entItems {
            roots = append(roots, r.entToDomain(entItem))
        }
        return nil
    })
    return roots, total, err
}
```

### Write Methods

```go
func (r *orderRepository) Save(ctx context.Context, root *aggregate.OrderAggregateRoot) (*aggregate.OrderAggregateRoot, error) {
    var savedRoot *aggregate.OrderAggregateRoot
    err := r.clientGetter.withDbClient(ctx, func(ctx context.Context, dbClient *ent.Client) error {
        if root.Order.ID != uuid.Nil {
            // Update existing
            update := dbClient.Order.UpdateOneID(root.Order.ID).
                SetTitle(root.Order.Title).
                SetStatus(order.Status(root.Order.Status)).
                SetTotalAmount(root.Order.TotalAmount)

            entItem, err := update.Save(ctx)
            if err != nil {
                return xerror.Wrap(err)
            }
            savedRoot = r.entToDomain(entItem)
            return nil
        }

        // Create new
        create := dbClient.Order.Create().
            SetUserID(root.Order.UserID).
            SetTitle(root.Order.Title).
            SetStatus(order.Status(root.Order.Status)).
            SetTotalAmount(root.Order.TotalAmount)

        entItem, err := create.Save(ctx)
        if err != nil {
            return xerror.Wrap(err)
        }
        savedRoot = r.entToDomain(entItem)
        return nil
    })
    return savedRoot, err
}

func (r *orderRepository) Delete(ctx context.Context, id uuid.UUID, userID string) error {
    return r.clientGetter.withDbClient(ctx, func(ctx context.Context, dbClient *ent.Client) error {
        exists, err := dbClient.Order.Query().
            Where(order.ID(id), order.UserID(userID)).
            Exist(ctx)
        if err != nil {
            return xerror.Wrap(err)
        }
        if !exists {
            return OrderNotFoundError
        }
        err = dbClient.Order.DeleteOneID(id).Exec(ctx)
        if err != nil {
            return xerror.Wrap(err)
        }
        return nil
    })
}
```

### Entity Mapping (ent -> domain)

Every repository has a private method to convert ent entities to domain aggregates:

```go
func (r *orderRepository) entToDomain(e *ent.Order) *aggregate.OrderAggregateRoot {
    return &aggregate.OrderAggregateRoot{
        Order: &entity.OrderEntity{
            ID:          e.ID,
            UserID:      e.UserID,
            Title:       e.Title,
            Status:      vo.OrderStatus(e.Status),
            TotalAmount: e.TotalAmount,
            CreatedAt:   e.CreatedAt,
            UpdatedAt:   e.UpdatedAt,
        },
    }
}
```

## Transaction Decorator

Shared across all repositories. Defined in `infrastructure/repository/transaction.go`:

```go
package repository

import (
    "myapp/infrastructure/repository/ent"
    "context"

    "github.com/futurxlab/golanggraph/logger"
)

type transactionDecorator struct {
    logger   logger.ILogger
    dbClient *ent.Client
}

func (t *transactionDecorator) WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error {
    tx, err := t.dbClient.Tx(ctx)
    if err != nil {
        t.logger.Errorf(ctx, "Failed to begin transaction: %v", err)
        return err
    }

    ctx = ent.NewTxContext(ctx, tx)
    err = fn(ctx)
    if err != nil {
        t.logger.Errorf(ctx, "Transaction function failed: %v", err)
        if rollbackErr := tx.Rollback(); rollbackErr != nil {
            t.logger.Errorf(ctx, "Failed to rollback transaction: %v", rollbackErr)
            return rollbackErr
        }
        return err
    }

    if commitErr := tx.Commit(); commitErr != nil {
        t.logger.Errorf(ctx, "Failed to commit transaction: %v", commitErr)
        return commitErr
    }
    return nil
}
```

## Client Getter

Transparently handles both transactional and non-transactional contexts:

```go
type clientGetter struct {
    dbClient *ent.Client
}

func (c *clientGetter) getDBClient(ctx context.Context) *ent.Client {
    tx := ent.TxFromContext(ctx)
    if tx != nil {
        return tx.Client()
    }
    return c.dbClient
}

func (c *clientGetter) withDbClient(
    ctx context.Context,
    fn func(ctx context.Context, dbClient *ent.Client) error,
) error {
    dbClient := c.getDBClient(ctx)
    return fn(ctx, dbClient)
}
```

## Ent Schema Patterns

### Mandatory Fields — EVERY Table

Every ent schema MUST have these 3 fields. No exceptions.

| Field | How | Behavior |
|-------|-----|----------|
| `id` | `field.UUID("id", uuid.UUID{}).Default(func() uuid.UUID { id, _ := uuid.NewV7(); return id }).Immutable()` | UUIDv7, auto-generated, immutable after creation |
| `created_at` | Via `mixin.Time{}` | Auto-set to `time.Now()` on creation, immutable afterward |
| `updated_at` | Via `mixin.Time{}` | Auto-set to `time.Now()` on creation, auto-updated to `time.Now()` on every mutation via ent's `UpdateDefault` |

**How it works**: `mixin.Time{}` from `entgo.io/ent/schema/mixin` adds both `created_at` and `updated_at` fields with correct defaults and update behavior. You do NOT need to manually set these fields in your repository `Save`/`Update` methods — ent handles it automatically.

### Full Schema Example

```go
// infrastructure/repository/ent/schema/order.go
package schema

import (
    "myapp/domain/order/vo"
    "myapp/utils"

    "entgo.io/ent"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/index"
    "entgo.io/ent/schema/mixin"
    "github.com/google/uuid"
)

type Order struct {
    ent.Schema
}

func (Order) Fields() []ent.Field {
    return []ent.Field{
        // UUIDv7 primary key
        field.UUID("id", uuid.UUID{}).
            Default(func() uuid.UUID { id, _ := uuid.NewV7(); return id }).
            Immutable(),

        field.String("user_id").
            NotEmpty(),

        field.String("title").
            NotEmpty(),

        // Enum - NEVER hardcode values, use GetAll + EnumToStrings
        field.Enum("status").
            Values(utils.EnumToStrings(vo.GetAllOrderStatuses())...),

        field.Float("total_amount").
            Default(0),

        // VO stored as JSON - NOT a separate table
        field.JSON("shipping_address", &vo.ShippingAddress{}).
            Optional(),
    }
}

func (Order) Mixin() []ent.Mixin {
    return []ent.Mixin{
        mixin.Time{}, // Adds created_at and updated_at
    }
}

func (Order) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("user_id"),
        index.Fields("status"),
    }
}

func (Order) Edges() []ent.Edge {
    // Define edges to child entities if needed
    return nil
}
```

### Key Schema Rules

1. **MANDATORY**: Every table MUST have `id` (UUIDv7), `created_at`, and `updated_at`. No exceptions.
2. **UUIDv7**: All tables use `uuid.NewV7()` as default ID, marked `Immutable()`
3. **Timestamps**: Always include `mixin.Time{}` in `Mixin()` — this provides `created_at` (immutable, set on creation) and `updated_at` (auto-updated on every save)
4. **Enums**: Use `utils.EnumToStrings(vo.GetAll*())` -- NEVER hardcode strings
5. **VOs**: Use `field.JSON()` for value objects -- NEVER create separate tables
6. **Indexes**: Add indexes for common query fields

## Ent Client

```go
// infrastructure/repository/client.go
package repository

import (
    "myapp/infrastructure/repository/ent"
    "myapp/config"

    _ "github.com/lib/pq" // PostgreSQL driver
    "github.com/futurxlab/golanggraph/xerror"
)

func NewClient(cfg *config.Config) (*ent.Client, error) {
    client, err := ent.Open("postgres", cfg.Postgresql.ConnectionString)
    if err != nil {
        return nil, xerror.Wrap(err)
    }
    return client, nil
}
```

## Redis Usage

Redis client is created in infrastructure and injected via fx:

```go
func newRedisClient(cfg *config.Config) *redis.Client {
    return redis.NewClient(&redis.Options{
        Addr:     cfg.Redis.Host,
        Password: cfg.Redis.Password,
        DB:       cfg.Redis.DbIndex,
        Protocol: 2,
    })
}
```

Use Redis for caching, pub/sub, or idempotency stores. Inject `*redis.Client` into repositories or services that need it.

## Logger

Custom logger wrapping zap, implementing `logger.ILogger` from golanggraph. Context-aware logging with trace_id support.

```go
// infrastructure/logger/logger.go
// Implements github.com/futurxlab/golanggraph/logger.ILogger interface
// Methods: Debugf, Infof, Warnf, Errorf -- all take context as first arg

logger.Infof(ctx, "Processing order %s", orderID)
logger.Errorf(ctx, "Failed to save order: %v", err)
```
