# Project Structure

```
backend/
├── cmd/
│   └── server/
│       └── main.go                        # Entry point, fx.New bootstrap
├── config/
│   ├── config.go                          # Main config struct, loader
│   ├── config_server.go                   # ServerConfig (Port, Env)
│   ├── config_postgresql.go               # PostgresqlConfig (ConnectionString)
│   ├── config_redis.go                    # RedisConfig (Host, Password, DbIndex)
│   └── config_log.go                      # LogConfig (Level, Format, File)
├── api/
│   ├── controller/
│   │   └── api/
│   │       ├── controller.go              # Single Controller struct holding all app service interfaces
│   │       └── {domain}_controller.go     # Per-domain handler methods on Controller
│   ├── dto/
│   │   ├── base_response.go              # OperationResponse and shared DTOs
│   │   └── {domain}.go                   # Request/Response DTOs per domain
│   ├── server/
│   │   ├── api_server.go                 # Creates gin server via libgin.NewGin(), fx lifecycle
│   │   ├── route/
│   │   │   ├── route.go                  # Route struct with config, controller, logger
│   │   │   ├── api.go                    # RegisterRoutes: groups, middleware, handler registration
│   │   │   └── handler.go               # NormalHandler, RequireUserHandler, EventStream* wrappers
│   │   └── middleware/
│   │       └── auth.go                   # Bearer token auth via kiwiuser client
│   └── module.go                         # fx.Provide for controller, route, server
├── application/
│   ├── service/
│   │   ├── {domain}/
│   │   │   └── service.go               # Application service: interface + implementation
│   │   └── error/
│   │       └── error.go                 # Application-level errors (libfacade.Error)
│   └── module.go                         # fx.Provide for all application services
├── domain/
│   ├── {domain}/
│   │   ├── aggregate/
│   │   │   └── {domain}_aggregate_root.go  # AggregateRoot struct + business methods
│   │   ├── entity/
│   │   │   └── {domain}_entity.go          # Entity structs with data attributes
│   │   ├── vo/
│   │   │   └── enums.go                    # Enum types with GetAll*() functions
│   │   ├── repository/
│   │   │   └── interface.go                # IRead + IWrite + IRepository interfaces
│   │   └── service/
│   │       └── service.go                  # Domain service interface + implementation
│   ├── common/
│   │   ├── contract/
│   │   │   └── transaction.go              # ITransactionDecorator interface
│   │   └── errors.go                       # Domain-level sentinel errors
│   └── module.go                           # fx.Provide for domain services
├── infrastructure/
│   ├── repository/
│   │   ├── client.go                       # ent.Open("postgres", connectionString)
│   │   ├── transaction.go                  # transactionDecorator + clientGetter
│   │   ├── {domain}_repository.go          # Concrete repository implementations
│   │   └── ent/
│   │       ├── schema/
│   │       │   └── {domain}.go             # Ent schema definitions
│   │       └── migrate/
│   │           └── migrations/             # Atlas migration files
│   ├── logger/
│   │   └── logger.go                       # Custom logger wrapping zap, implements ILogger
│   └── module.go                           # fx.Provide for ALL infra: logger, redis, ent, repos, clients
├── utils/
│   └── enum_converter.go                   # EnumToStrings generic converter
├── Makefile                                # generate, migrate, build commands
└── go.mod
```

## Key Points

- One `controller.go` holds a single `Controller` struct with all injected app service interfaces
- Each domain gets its own `{domain}_controller.go` with handler methods on that Controller
- `module.go` exists at each layer root, exporting `var Module = fx.Provide(...)` or `fx.Module(...)`
- Domain layer has zero external dependencies (no frameworks, no infra imports)
- Infrastructure `module.go` is the largest -- it wires ALL concrete implementations to domain interfaces
