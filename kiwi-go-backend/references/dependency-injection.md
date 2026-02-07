# Dependency Injection with fx

## Overview

All dependency wiring uses `go.uber.org/fx`. Each architectural layer exports a `Module` variable that registers its providers.

## Main Bootstrap

```go
// cmd/server/main.go
package main

import (
    "myapp/api"
    "myapp/application"
    "myapp/config"
    "myapp/domain"
    "myapp/infrastructure"

    "go.uber.org/fx"
)

func main() {
    app := fx.New(
        fx.NopLogger,
        fx.Provide(config.LoadConfig),
        api.Module,
        application.Module,
        domain.Module,
        infrastructure.Module,
        fx.Invoke(func(*api.Server) {}), // Triggers server startup
    )
    app.Run()
}
```

## Layer Modules

### API Module

```go
// api/module.go
package api

import (
    controller "myapp/api/controller/api"
    "myapp/api/server"
    "myapp/api/server/route"

    "go.uber.org/fx"
)

var Module = fx.Provide(
    controller.NewController,
    route.NewRoute,
    server.NewServer,
)
```

### Application Module

```go
// application/module.go
package application

import (
    orderservice "myapp/application/service/order"

    "go.uber.org/fx"
)

var Module = fx.Provide(
    orderservice.NewOrderApplicationService,
)
```

### Domain Module

```go
// domain/module.go
package domain

import (
    "go.uber.org/fx"
)

var Module = fx.Provide(
    // Domain services are typically provided by infrastructure module
    // since they may need concrete implementations
)
```

### Infrastructure Module

The infrastructure module is the largest. It wires ALL concrete implementations to their domain interfaces.

```go
// infrastructure/module.go
package infrastructure

import (
    "myapp/config"
    "myapp/infrastructure/repository"
    "myapp/infrastructure/repository/ent"

    orderrepository "myapp/domain/order/repository"

    "github.com/futurxlab/golanggraph/logger"
    "github.com/futurxlab/golanggraph/xerror"
    "github.com/redis/go-redis/v9"
    "go.uber.org/fx"
)

func newLogger(config *config.Config) (logger.ILogger, error) {
    // Create custom logger
}

func newRedisClient(cfg *config.Config) *redis.Client {
    return redis.NewClient(&redis.Options{
        Addr:     cfg.Redis.Host,
        Password: cfg.Redis.Password,
        DB:       cfg.Redis.DbIndex,
        Protocol: 2,
    })
}

var Module = fx.Provide(
    // Utilities
    newLogger,

    // Clients
    newRedisClient,

    // Ent client with cleanup on stop
    fx.Annotate(
        repository.NewClient,
        fx.OnStop(func(client *ent.Client) {
            client.Close()
        }),
    ),

    // Repository binding: concrete -> all three interfaces
    fx.Annotate(
        repository.NewOrderRepository,
        fx.As(new(orderrepository.IOrderRepository)),
        fx.As(new(orderrepository.IOrderRepositoryRead)),
        fx.As(new(orderrepository.IOrderRepositoryWrite)),
    ),

    // Add more repositories following the same pattern...
)
```

## Key Patterns

### fx.Annotate + fx.As (Repository Binding)

Every repository concrete type maps to all three domain interfaces:

```go
fx.Annotate(
    repository.NewOrderRepository,
    fx.As(new(orderrepository.IOrderRepository)),
    fx.As(new(orderrepository.IOrderRepositoryRead)),
    fx.As(new(orderrepository.IOrderRepositoryWrite)),
),
```

This allows:
- Application layer to inject `IOrderRepositoryRead` (read-only access)
- Domain services to inject `IOrderRepository` (full access with transactions)
- Any component that only needs write can inject `IOrderRepositoryWrite`

### fx.OnStop (Cleanup)

Use `fx.OnStop` for cleanup hooks on clients that need graceful shutdown:

```go
fx.Annotate(
    repository.NewClient,
    fx.OnStop(func(client *ent.Client) {
        client.Close()
    }),
),
```

### fx.Lifecycle (Server Hooks)

Use `fx.Lifecycle` for start/stop hooks on servers:

```go
func NewServer(lc fx.Lifecycle, ...) {
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            go server.Start()
            return nil
        },
        OnStop: func(ctx context.Context) error {
            return server.Shutdown(ctx)
        },
    })
}
```

### fx.NopLogger

Always use `fx.NopLogger` in `fx.New()` to suppress fx's internal logging noise:

```go
app := fx.New(
    fx.NopLogger,
    // ...modules
)
```

## Import Alias Convention

When importing domain repository packages, use aliases to avoid collisions:

```go
import (
    orderrepository    "myapp/domain/order/repository"
    userrepository     "myapp/domain/user/repository"
    productrepository  "myapp/domain/product/repository"
)
```
