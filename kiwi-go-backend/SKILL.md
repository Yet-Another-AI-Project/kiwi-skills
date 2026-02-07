---
name: kiwi-go-backend
description: Go backend development standards using DDD architecture, ent ORM, fx DI, and kiwi-lib utilities. Covers project structure, domain design, API/application/infrastructure layers, database workflow, and coding requirements.
---

# Kiwi Go Backend Development

This skill defines the standards for building Go backends in the kiwi ecosystem. All code generation, refactoring, and new feature work MUST follow these rules.

## Architecture Overview

Four-layer DDD architecture with strict dependency direction: `api -> application -> domain <- infrastructure`.

| Layer | Responsibility | Key Rule |
|-------|---------------|----------|
| API | HTTP handling, param parsing, response serialization | Thin layer. No business logic. |
| Application | Orchestrates domain services for use cases | Can READ from repo. CANNOT WRITE directly. |
| Domain | Business logic, entities, aggregate roots, VOs | Pure Go. No framework dependencies. |
| Infrastructure | Repository implementations, external clients, DI | Implements domain interfaces. |

See [references/project-structure.md](references/project-structure.md) for full directory layout.

## Domain Layer Rules

### Naming Conventions
- Entities: `*Entity` suffix (e.g., `UserEntity`)
- Aggregate Roots: `*AggregateRoot` suffix (e.g., `OrderAggregateRoot`)
- Interfaces: `I` prefix (e.g., `IOrderRepository`)

### Aggregate Root = Pure Container
- NO direct data attributes on aggregate root
- Structure: `KeyEntity` + `[]ChildEntity` + business logic methods
- All data lives in the KeyEntity

### Value Objects
- Do NOT create a VO with fewer than 3 fields
- 1-2 field concepts stay as primitives on the Entity
- Store VOs as `field.JSON` in ent schema, NOT separate tables

### Enums
Every enum type MUST have a `GetAll*()` function returning all values. This is used by ent schemas.

```go
type OrderStatus string
const (
    OrderStatusPending   OrderStatus = "pending"
    OrderStatusCompleted OrderStatus = "completed"
)
func GetAllOrderStatuses() []OrderStatus {
    return []OrderStatus{OrderStatusPending, OrderStatusCompleted}
}
```

### Repository Interfaces (Triple Pattern)
Defined in `domain/{domain}/repository/interface.go`:

```go
type IOrderRepositoryRead interface {
    FindByID(ctx context.Context, id uuid.UUID) (*aggregate.OrderAggregateRoot, error)
}
type IOrderRepositoryWrite interface {
    Save(ctx context.Context, root *aggregate.OrderAggregateRoot) (*aggregate.OrderAggregateRoot, error)
}
type IOrderRepository interface {
    IOrderRepositoryRead
    IOrderRepositoryWrite
    contract.ITransactionDecorator
}
```

- ReadOnly: can return `*Entity` or `*AggregateRoot`
- Write: MUST operate on `*AggregateRoot` only

### Transaction Contract
Defined in `domain/common/contract/transaction.go`:
```go
type ITransactionDecorator interface {
    WithTransaction(ctx context.Context, fn func(ctx context.Context) error) error
}
```

See [references/domain-layer.md](references/domain-layer.md) for detailed patterns.

## Application Layer Rules

- Orchestrates domain services to implement use cases
- Can inject and use `RepositoryRead` interfaces for queries
- CANNOT call `RepositoryWrite` directly -- MUST go through Domain Service
- Input: can reuse API layer DTOs or define own
- Output: application-defined response models
- Translates domain errors to `libfacade.Error` types

```go
// application/service/error/error.go
var (
    ErrOrderNotFound = libfacade.NewNotFoundError("order not found")
    ErrForbidden     = libfacade.NewForbiddenError("access denied")
)
```

See [references/api-application-layer.md](references/api-application-layer.md) for full patterns.

## API Layer Rules

- Thin layer: parse params, call application service, return response
- No business logic, no direct domain service calls, no repository usage
- DTOs in `api/dto/` with `json` and `binding` tags
- Use generic handler wrappers for consistent error/response formatting:

```go
// Normal endpoint (no auth)
NormalHandler[T any](f func(*gin.Context) (T, *libfacade.Error))

// Auth-required endpoint (extracts userID from middleware)
RequireUserHandler[T any](f func(*gin.Context, string) (T, *libfacade.Error))

// SSE streaming
EventStreamHandler(f func(*gin.Context) *libfacade.Error)
EventStreamRequireUserHandler(f func(*gin.Context, string) *libfacade.Error)
```

Handler wrappers auto-wrap responses in `libfacade.BaseResponse{Status, Data}` and handle panics.

## Infrastructure Layer Rules

### Repository Implementation
Every repository embeds `transactionDecorator` + `clientGetter`. ALL methods use `withDbClient` pattern:

```go
func (r *repo) FindByID(ctx context.Context, id uuid.UUID) (*aggregate.Root, error) {
    var root *aggregate.Root
    err := r.clientGetter.withDbClient(ctx, func(ctx context.Context, dbClient *ent.Client) error {
        entItem, err := dbClient.Order.Query().Where(order.ID(id)).Only(ctx)
        if err != nil {
            if ent.IsNotFound(err) { return OrderNotFoundError }
            return xerror.Wrap(err)
        }
        root = r.entToDomain(entItem)
        return nil
    })
    return root, err
}
```

### Ent Schema Rules
- UUIDv7 primary key: `field.UUID("id", uuid.UUID{}).Default(func() uuid.UUID { id, _ := uuid.NewV7(); return id }).Immutable()`
- Timestamps via `mixin.Time{}`
- Enums: `field.Enum("status").Values(utils.EnumToStrings(vo.GetAllOrderStatuses())...)` -- NEVER hardcode
- VOs as JSON: `field.JSON("metadata", &vo.OrderMetadata{}).Optional()`

See [references/infrastructure-layer.md](references/infrastructure-layer.md) for full patterns.

## Database Workflow

1. Define/modify ent schema in `infrastructure/repository/ent/schema/`
2. Generate ent code: `make generate-repository-code`
3. Generate migration: `make generate-migration-repository-db`
4. Apply migration: `make apply-migration-repository-db`

See [references/database-workflow.md](references/database-workflow.md) for Makefile and details.

## Dependency Injection (fx)

All wiring uses `go.uber.org/fx`. Each layer exports a `Module` variable.

```go
// main.go
app := fx.New(
    fx.NopLogger,
    api.Module,
    application.Module,
    domain.Module,
    infrastructure.Module,
)
app.Run()
```

Repository binding uses `fx.Annotate` + `fx.As` to map concrete types to all three interfaces:
```go
fx.Annotate(
    repository.NewOrderRepository,
    fx.As(new(orderrepo.IOrderRepository)),
    fx.As(new(orderrepo.IOrderRepositoryRead)),
    fx.As(new(orderrepo.IOrderRepositoryWrite)),
),
```

See [references/dependency-injection.md](references/dependency-injection.md) for module patterns.

## Error Handling

| Layer | Method | Example |
|-------|--------|---------|
| Domain/Infrastructure | `xerror.Wrap(err)`, `xerror.New("msg")` | `return xerror.Wrap(err)` |
| Application | `libfacade.NewNotFoundError()`, `NewForbiddenError()` | `return nil, ErrOrderNotFound` |
| API | Handler wrappers auto-format `*libfacade.Error` | Automatic |

NEVER use `fmt.Errorf()`, `errors.New()`, or return raw unwrapped errors.

## Key Import Paths

```go
import (
    libgin    "github.com/Yet-Another-AI-Project/kiwi-lib/server/gin"
    libfacade "github.com/Yet-Another-AI-Project/kiwi-lib/server/facade"
    "github.com/futurxlab/golanggraph/logger"   // logger.ILogger
    "github.com/futurxlab/golanggraph/xerror"    // xerror.Wrap, xerror.New
    "entgo.io/ent"                               // ORM
    "go.uber.org/fx"                             // DI
    "github.com/google/uuid"                     // uuid.NewV7()
    "github.com/gookit/config/v2"                // Config loading
    "github.com/redis/go-redis/v9"               // Redis client
)
```

## Config Pattern

Use `gookit/config/v2` with YAML driver. Config struct uses `config:"field_name"` tags. Load via flag-based config file path.

```go
type Config struct {
    Server     ServerConfig     `config:"server"`
    Postgresql PostgresqlConfig `config:"postgres"`
    Redis      RedisConfig      `config:"redis"`
    Log        LogConfig        `config:"log"`
}
```

## Coding Checklist

Before submitting any code, verify:

- [ ] All enum types have `GetAll*()` function
- [ ] All `return err` use `xerror.Wrap(err)`
- [ ] All new errors use `xerror.New("message")`
- [ ] No `fmt.Errorf()` or `errors.New()` usage
- [ ] Aggregate roots have no direct attributes (use KeyEntity)
- [ ] VOs have 3+ fields (otherwise use primitives)
- [ ] Ent schemas use `utils.EnumToStrings()` for enums, not hardcoded strings
- [ ] Ent schemas use UUIDv7 and `mixin.Time{}`
- [ ] Repository methods use `withDbClient` pattern
- [ ] Application layer reads via `RepositoryRead`, writes via Domain Service
- [ ] API layer is thin -- no business logic
- [ ] fx modules use `fx.Annotate` + `fx.As` for repository binding
- [ ] Run `make generate-repository-code` after schema changes

## Utility: EnumToStrings

```go
// utils/enum_converter.go
type EnumType interface{ ~string }
func EnumToStrings[T EnumType](enums []T) []string {
    result := make([]string, len(enums))
    for i, e := range enums { result[i] = string(e) }
    return result
}
```
