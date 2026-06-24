---
name: golang-service-patterns
description: Best practices for the Go backend services in shopoo (auth-service, media-service, noti-service). Use when writing, reviewing, or refactoring any Golang code in this project — covers package layout, error handling, context, pgx/Postgres, Redis, bounded concurrency, config, structured logging, HTTP routing with chi, graceful shutdown, and testing. Pair with marketplace-conventions.
---

# Go Service Patterns

> Reference implementation: `backend/services/auth-service`. New Go services mirror its layout.

## Layout (standard for every Go service)
```
cmd/server/main.go        wiring + graceful shutdown only
internal/config           typed env config, validated at startup
internal/db               pgx pool, redis client, embedded migrations
internal/token            JWT verify (public key) / sign (auth only)
internal/model            domain structs (no behaviour)
internal/repo             SQL via pgx; returns domain models + sentinel errors
internal/service          business logic; orchestrates repo + redis + mq
internal/handler          HTTP handlers: decode → call service → map errors → JSON
internal/middleware       auth, request-id, recoverer
internal/router           chi routes wiring handlers + middleware
internal/event            RabbitMQ publish/consume (when needed)
```
`internal/` keeps the package private. `main` does no business logic.

## Core rules
- **`context.Context` is the first arg** of every repo/service/IO call; propagate request context, honour cancellation. Never `context.Background()` below `main`/startup.
- **Errors:** wrap with `fmt.Errorf("doing x: %w", err)`. Define sentinel errors in the service/repo layer (`var ErrNotFound = errors.New(...)`); map them to HTTP **only in the handler** with `errors.Is`. Never `panic` in library code; never ignore an error (`_ =` only for genuinely-ignorable like buffered writes).
- **Accept interfaces, return structs.** Constructors `New...` return concrete types; depend on small interfaces where you need to mock.
- **No global mutable state.** Pass dependencies in explicitly (the `Handler`/`Service` structs hold them).

## Postgres (pgx/pgxpool)
- One `*pgxpool.Pool` per service, created in `db.NewPool`, `Ping` on startup, `Close` on shutdown.
- Parameterized queries only (`$1,$2`). Centralise column lists + a `scanX(row)` helper. Map `pgx.ErrNoRows` → your `ErrNotFound`.
- Run idempotent embedded migrations (`//go:embed migrations/*.sql`) at startup for now; graduate to golang-migrate + a `schema_migrations` table when schemas churn.
- Use `search_path` (set on the role + connection) so unqualified names resolve to the service schema.

## Redis & concurrency
- `redis/go-redis/v9`; `Ping` on startup. Use it for refresh tokens, cache, presence, rate-limit — never as the source of truth.
- CPU/IO fan-out (e.g. image resize): bounded worker pool or `golang.org/x/sync/errgroup` with a semaphore — never spawn unbounded goroutines per request. Always have a way for every goroutine to exit (context).

## HTTP
- Router: `chi`. Wrap with `RequestID`, `RealIP`, `Logger`, `Recoverer`. Static routes win over `{param}` routes — exploit that, don't fight it.
- `http.Server` with `ReadTimeout`/`WriteTimeout`; **graceful shutdown** on SIGINT/SIGTERM via `srv.Shutdown(ctx)`.
- Handlers: decode JSON (reject unknown bodies), validate required fields explicitly, call the service, write `writeJSON`/`writeError` helpers that emit the project error shape `{"error":"..."}`.

## Config & logging
- Typed `Config` struct loaded in `config.Load()`; defaults for dev, but **fail fast** if a required prod var is missing/invalid.
- Logging: standard library **`log/slog`** (JSON handler) with `requestId`/`service` attrs. No `fmt.Println` debugging left in.

## Testing
- Table-driven tests; `httptest` for handlers; mock the repo via a small interface. Keep service logic pure enough to unit-test without a DB. Name `TestXxx_Scenario`.

## Don't
Don't put SQL in handlers · don't return raw pgx/redis errors to clients · don't log-and-return the same error at every layer · don't block on RabbitMQ in the request path.
