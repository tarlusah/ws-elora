# Database Conventions

## Naming Conventions

| Layer | Convention | Example |
|-------|-----------|---------|
| Database columns | `snake_case` | `user_id`, `created_at` |
| Database table names | `snake_case` | `order_items`, `user_profiles` |

**Rules:**
- Entity structs use `db:"snake_case"` tags to map to database columns
- Never expose database column names directly in API responses

```go
// ✅ Correct
type User struct {
    ID        string    `db:"id"`
    FullName  string    `db:"full_name"`
    CreatedAt time.Time `db:"created_at"`
}
```

---

## Transaction Management

**Transactions are owned by the usecase, not the repository.**

The usecase decides which operations must be atomic. The `TxManager` handles
the actual begin / commit / rollback mechanics. Repositories are transaction-agnostic
— they read the transaction from context if present.

```go
// pkg/database/transaction.go
type TxManager interface {
    RunInTx(ctx context.Context, fn func(ctx context.Context) error) error
}

// Usage in usecase
func (u *exampleUsecase) Create(ctx context.Context, req dto.CreateRequest) error {
    return u.txManager.RunInTx(ctx, func(ctx context.Context) error {
        if err := u.repoA.Save(ctx, entityA); err != nil {
            return err // triggers rollback
        }
        if err := u.repoB.Save(ctx, entityB); err != nil {
            return err // triggers rollback — repoA also rolled back
        }
        return nil // triggers commit
    })
}
```

**Rules:**
- `return nil` inside `RunInTx` → commit
- `return err` inside `RunInTx` → rollback all operations in that transaction
- Multiple entities within the same domain can share one transaction
- Cross-storage transactions (e.g. PostgreSQL + Redis) are not possible — PostgreSQL
  goes inside `RunInTx`, Redis operations happen after the transaction commits
- Cross-service transactions (after microservice split) require Saga or Outbox pattern —
  do not attempt this with SQL transactions

---

## Migrations Structure

Migration files live **inside the app module** that owns them, not at the repository root.
This allows `//go:embed` to bundle them into the binary. Migrations run automatically
on app startup via `golang-migrate/migrate`. Migration files are append-only —
never edit an existing migration file, only add new ones.

```
apps/
├── server/
│   └── migrations/
│       ├── 000001_create_{table}.up.sql
│       ├── 000001_create_{table}.down.sql
│       ├── 000002_add_{column}_to_{table}.up.sql
│       └── 000002_add_{column}_to_{table}.down.sql
└── {service}-svc/
    └── migrations/
        ├── 000001_create_{table}.up.sql
        └── 000001_create_{table}.down.sql
```

**Naming convention:** `{6-digit-version}_{description}.up.sql` / `.down.sql`

**Rules:**
- Every `.up.sql` must have a matching `.down.sql` for rollback support
- Version numbers are zero-padded to 6 digits: `000001`, `000002`, …
- The migration table `schema_migrations` is managed automatically by golang-migrate
- Migrations are append-only — never edit existing migration files; always add a new version
- Migrations live inside `apps/` — embedded via `//go:embed`; run automatically on startup via golang-migrate
