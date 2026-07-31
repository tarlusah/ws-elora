# Coding Style & Conventions

## Go Module Structure

This repository uses **Go Workspaces** (`go.work`) — there is no `go.mod` at the repository root.

Each layer has its own module:

| Directory      | Module                                          |
|----------------|-------------------------------------------------|
| `pkg/`         | `github.com/elora/elora-be-go/pkg`             |
| `internal/`    | `github.com/elora/elora-be-go/internal`        |
| `apps/server/` | `github.com/elora/elora-be-go/apps/server`     |

**Rules:**
- `go.work` lives at the repository root and references all three modules
- `apps/server/` imports `pkg/` and `internal/` via `go.work` — **not** via `replace` directives
- When a new service is added under `apps/`, add it to the existing `go.work` — do not create a new `go.work`
- During CI/CD, run `go work sync` before building
- `go mod tidy` cannot resolve workspace-local modules via proxy (they are not published). Run it only for external deps: `go mod tidy -e` is acceptable. Use `go build ./...` from the repo root to verify the full workspace compiles.

---

## Domain Structure

Every domain inside `internal/` must follow this structure:

```
internal/{domain}/
├── handler/
│   ├── {domain}_handler.go
│   └── {domain}_router.go
├── usecase/
│   ├── {domain}_usecase.go
│   └── {domain}_usecase_test.go
├── repository/
│   ├── {domain}_repo.go         # PostgreSQL implementation
│   └── {domain}_cache_repo.go   # Redis implementation (if needed)
├── entity/
│   └── {domain}.go              # Core domain structs
├── dto/
│   ├── request.go
│   └── response.go
├── event/                       # Only if this domain publishes events
│   ├── {domain}_publisher.go
│   └── {domain}_event.go        # Event struct definitions
└── projection/                  # Only if this domain consumes events from other domains
    └── {source_domain}_projection.go
```

---

## File Naming Convention

All file names follow the pattern: **noun first, then qualifier**.

```
✅ Correct              ❌ Wrong
{domain}_handler.go     handler_{domain}.go
{domain}_usecase.go     usecase_{domain}.go
{domain}_repo.go        repo_{domain}.go
{domain}_event.go       event_{domain}.go
```

Struct names follow the same pattern:

```go
type OrderRepo struct {}       // ✅
type OrderUsecase struct {}    // ✅
type RepoOrder struct {}       // ❌
type UsecaseOrder struct {}    // ❌
```

---

## Layer Responsibilities

### handler/
- Receives HTTP / gRPC requests
- Validates request format (binding, required fields)
- Calls usecase
- Does NOT contain business logic
- Does NOT access the database directly

### usecase/
- Contains all business logic and decision making
- Defines interfaces for everything it depends on (repository, other domains, eventbus)
- Manages transactions — decides which operations must be atomic
- Does NOT know about HTTP, gRPC, or SQL
- Does NOT import other domain packages directly

### repository/
- The only layer that communicates with storage (PostgreSQL, Redis, etc.)
- Does NOT know about business rules
- Checks context for an active transaction — uses it if present, otherwise uses the default DB connection

### entity/
- Defines the core data structures of the domain
- No methods with business logic — plain structs only

### dto/
- Defines request and response shapes for the API layer
- Kept separate from entity to avoid coupling API contracts to internal models

### event/
- Publishes domain events after successful operations
- Event structs are considered a public contract — see `api-design.md` for Event Contract rules

### projection/
- Consumes events from other domains via the eventbus
- Writes a partial copy of external data into a local table
- Only stores fields that this domain actually needs — never a full copy

---

## Cross-Domain Communication Rules

**Domains must never import each other directly.**

```go
// ❌ Forbidden — direct import between domains
import "internal/user/usecase"   // inside order domain

// ✅ Correct — define an interface in the consuming domain
// internal/order/usecase/order_usecase.go
type UserProvider interface {
    GetByID(ctx context.Context, userID string) (*UserData, error)
}
```

The consuming domain defines the interface. The providing domain implements it
without knowing the interface exists (Go uses implicit interface satisfaction).

**The only place that knows all implementations is `apps/`.**

```go
// apps/server/main.go
userUsecase  := user.NewUsecase(userRepo)
orderUsecase := order.NewUsecase(orderRepo, userUsecase) // userUsecase satisfies UserProvider
```

---

## Dependency Injection — apps/ is the Orchestrator

`apps/` is the only place responsible for:
- Wiring all implementations to their interfaces
- Choosing the tech stack (Gin, gRPC, Kafka consumer, etc.)
- Connecting infrastructure (DB, Redis, Kafka) to domain usecases

```go
// apps/server/main.go
func main() {
    // 1. infrastructure
    db        := database.NewPostgres(cfg)
    redis     := database.NewRedis(cfg)
    eventBus  := eventbus.NewInMemory()   // phase 1: in-memory
    // eventBus := eventbus.NewKafka(cfg) // phase 2: swap to Kafka

    // 2. repositories
    domainRepo := domainrepo.New(db)

    // 3. usecases
    domainUsecase := domainusecase.New(domainRepo, eventBus)

    // 4. handlers
    domainHandler := domainhandler.New(domainUsecase)

    // 5. server
    r := gin.New()
    r.POST("/resource", domainHandler.Create)
    r.Run()
}
```

`internal/` must never know which tech stack `apps/` is using.

---

## pkg/ Rules

Packages inside `pkg/` are shared across all domains and services.

- `pkg/` must never import from `internal/`
- `pkg/` must never contain business logic
- Every package in `pkg/` must be independently usable by any domain

---

## configs/ Placement

Configuration files belong to the binary that uses them, not to the repository root.

```
apps/
├── server/
│   └── configs/
│       ├── app.dev.yaml
│       └── app.prod.yaml
└── {service}-svc/
    └── configs/
        ├── app.dev.yaml
        └── app.prod.yaml
```

Values that change between environments (DB URL, ports, secrets) must use
environment variables in production — never hardcode them.

---

## Hard Rules Summary

| Rule | Detail |
|------|--------|
| No direct domain imports | Domains communicate only via interfaces |
| Interfaces defined by consumer | The domain that needs data defines the interface |
| `apps/` is the only orchestrator | Only `apps/` knows all implementations |
| Transactions owned by usecase | Never manage transactions in repository or handler |
| Events are versioned contracts | Never rename or remove fields from existing events |
| Projections are partial copies | Only store fields the domain actually uses |
| Configs live inside `apps/` | Never put service configs at the repository root |
| File naming: noun first | `order_repo.go` not `repo_order.go` |
| Go module per layer | `pkg/`, `internal/`, and each `apps/*` have their own `go.mod` — connected via `go.work` |
