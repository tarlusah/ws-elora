# API Design Standards

## Naming Conventions

| Layer | Convention | Example |
|-------|-----------|---------|
| Go struct fields | `PascalCase` | `UserID`, `CreatedAt` |
| REST request/response JSON properties | `camelCase` | `userId`, `createdAt` |

**Rules:**
- DTO structs use `json:"camelCase"` tags — never `json:"snake_case"` on API-facing fields
- Never expose database column names directly in API responses

```go
// ✅ Correct
type CreateUserRequest struct {
    FullName string `json:"fullName"  validate:"required"`
    Email    string `json:"email"     validate:"required,email"`
}

// ❌ Wrong — snake_case leaking into JSON
type CreateUserRequest struct {
    FullName string `json:"full_name"`
}
```

---

## Event System

### EventBus Interface

```go
// pkg/eventbus/eventbus.go
type EventBus interface {
    Publish(ctx context.Context, topic string, payload any) error
    Subscribe(topic string, handler HandlerFunc)
}
```

- **Phase 1:** inject `inmemory.go` — events are processed in the same process
- **Phase 2:** inject `kafka.go` — zero changes in domain code

### Event Contract Rules

Event structs are a **public contract** between domains. Treat them like a versioned API:

```
✅ Allowed    — add new optional fields to existing events
❌ Forbidden  — rename, remove, or change the type of existing fields
❌ Forbidden  — change a topic name that other domains already subscribe to
```

If breaking changes are required, create a new versioned event:

```go
type UserCreatedV1 struct { UserID, Name, Email string }
type UserCreatedV2 struct { UserID, Name, Email, Phone string }
```

### Projection Rules

- A projection table only stores fields that the owning domain actually queries
- Never copy all fields from the source domain — only what is needed
- Projection consumers must use **UPSERT**, not INSERT — to handle duplicate events
- Every service must provide a backfill endpoint or script to populate projections
  on first deploy

---

## Correlation ID

Every request must carry a correlation ID from entry to exit, including all
events published during the request lifecycle.

```go
// pkg/middleware/correlation.go
// - Generate X-Correlation-ID on every incoming request if not present
// - Pass it through context
// - Include it in every log entry
// - Include it in every event payload
```

This enables full request tracing across services using a single ID.
