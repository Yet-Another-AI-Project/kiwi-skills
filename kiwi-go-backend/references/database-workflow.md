# Database Workflow

## Overview

The database workflow follows a code-first approach using ent ORM for schema definition and Atlas for migrations.

```
Define Schema -> Generate Code -> Generate Migration -> Apply Migration
```

## Step 1: Define Ent Schema

Create or modify schemas in `infrastructure/repository/ent/schema/`.

**Every schema MUST include**: UUIDv7 `id` field + `mixin.Time{}` for `created_at`/`updated_at`. See Infrastructure Layer docs for details.

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
        // MANDATORY: UUIDv7 primary key (immutable)
        field.UUID("id", uuid.UUID{}).
            Default(func() uuid.UUID { id, _ := uuid.NewV7(); return id }).
            Immutable(),
        field.String("user_id").NotEmpty(),
        field.String("title").NotEmpty(),
        field.Enum("status").
            Values(utils.EnumToStrings(vo.GetAllOrderStatuses())...),
        field.Float("total_amount").Default(0),
        field.JSON("metadata", &vo.OrderMetadata{}).Optional(),
    }
}

func (Order) Mixin() []ent.Mixin {
    // MANDATORY: provides created_at (immutable, set on creation) and
    // updated_at (auto-updated on every mutation)
    return []ent.Mixin{mixin.Time{}}
}

func (Order) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("user_id"),
    }
}
```

## Step 2: Generate Ent Code

After modifying schemas, run code generation to update the generated ent package:

```bash
make generate-repository-code
```

This generates query builders, mutations, and client code in `infrastructure/repository/ent/`.

## Step 3: Generate Migration

Generate an Atlas migration diff based on schema changes:

```bash
make generate-migration-repository-db
```

This compares the current ent schema against the migration directory and creates a new migration file in `infrastructure/repository/ent/migrate/migrations/`.

Requires a dev database for diffing (typically local PostgreSQL).

## Step 4: Apply Migration

Apply pending migrations to the target database:

```bash
make apply-migration-repository-db
```

Reads the connection string from the config file.

## Makefile Template

```makefile
config=`pwd`/conf/config.json

# Run unit tests
test-unit:
	go test -v -count 1 -run Unit ./...

# Generate ent ORM code from schemas
generate-repository-code:
	go run -mod=mod entgo.io/ent/cmd/ent generate --feature sql/execquery ./infrastructure/repository/ent/schema

# Generate Atlas migration diff
generate-migration-repository-db:
	atlas migrate diff --dir "file://infrastructure/repository/ent/migrate/migrations" --to "ent://infrastructure/repository/ent/schema" --dev-url "postgres://root:pgpass@localhost:5432/atlas_dev?search_path=public&sslmode=disable"

# Apply migrations to target database
apply-migration-repository-db:
	atlas migrate apply --dir "file://infrastructure/repository/ent/migrate/migrations" --url $(shell cat $(config) | jq .postgres.connection_string)

# Build Docker images
build-staging-image:
	docker buildx build --platform linux/amd64 -f build/Dockerfile --build-arg github_user=${GITHUB_USER} --build-arg github_access_token=${GITHUB_ACCESS_TOKEN} -t myapp:staging .

build-production-image:
	docker buildx build --platform linux/amd64 -f build/Dockerfile --build-arg github_user=${GITHUB_USER} --build-arg github_access_token=${GITHUB_ACCESS_TOKEN} -t myapp:$(VERSION) .
```

## Important Notes

1. ALWAYS run `make generate-repository-code` after ANY schema change
2. Review generated migration SQL before applying to production
3. The dev-url database for Atlas should be a clean database used only for diffing
4. Migration files are versioned and should be committed to git
5. Never manually edit generated ent code -- it will be overwritten
