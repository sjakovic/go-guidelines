# Go Engineering Guidelines

Version 1.0 - October 2026

These are the rules every Go change in this repository must follow. They exist so that a file never grows into a 2000-line handler, so that a new engineer can find any piece of logic by its name alone, and so that reviews argue about behaviour instead of style.

CI enforces the mechanical parts (format, vet, lint, size limits, tests). Reviewers enforce the rest. When a rule here conflicts with a reviewer's taste, the rule wins. When a rule is wrong, change the rule in a PR, do not ignore it.

Principles, in priority order:

1. **One responsibility per layer.** HTTP code knows nothing about SQL. SQL code knows nothing about HTTP. Business rules know neither.
2. **Small units.** A file is read in one sitting, a function fits on one screen. The limits below are hard, not aspirational.
3. **No duplication.** The second copy of anything becomes a shared helper in the same PR.
4. **Boring and explicit.** No reflection, no clever generics, no magic. A new engineer must be able to trace a request without a debugger.
5. **Standard library first.** A dependency must buy something the standard library cannot.

---

## 1. Project layout

### 1.1 Service / API

```
<project>/
├── cmd/
│   └── <binary>/
│       └── main.go          main calls run(); run loads config, wires dependencies, serves
├── internal/
│   ├── config/              env vars → Config struct; the only package that reads os.Getenv
│   ├── domain/<area>/       pure types, enums, rules; no I/O, no SQL, no HTTP
│   ├── service/<area>/      use cases: validation, authorization, transactions
│   ├── storage/<driver>/    all SQL; one file per aggregate; migrations/ inside
│   ├── api/
│   │   ├── server.go        router construction and the route table
│   │   ├── handlers/        decode → one service call → encode
│   │   ├── middleware/      auth, request id, logging, CORS, recovery
│   │   └── respond/         JSON/error writers, context accessors, pagination parsing
│   ├── apperr/              the application error type (not `errors`: never shadow the standard library)
│   ├── id/                  small helper packages directly under internal/, one capability each
│   ├── money/               no pkg/ bucket at any level
│   └── <adapter>/           one package per external system: email/, pdf/, oauth/, s3/
├── docs/                    ARCHITECTURE.md, API.md, this file
├── deploy/                  compose files, reverse proxy config, deploy scripts
├── Dockerfile
├── Makefile
├── .golangci.yml
├── .github/workflows/ci.yml
└── go.mod
```

Dependency direction is strictly downward:

```
api  →  service  →  storage  →  domain
```

- `domain` imports nothing from the project.
- `storage` never imports `api` or `service`.
- `service` never imports `net/http` or the database driver.
- `api` never imports the database driver.
- Helper packages (`apperr`, `id`, `money`) import nothing from the project.

### 1.2 CLI, worker, desktop app

No HTTP, no multi-tenancy, no authorization means no service layer. Use a flat `internal/` with capability-named packages, one orchestrator package and one boundary package:

```
<project>/
├── cmd/<binary>/main.go     (or main.go at the root if the framework requires it)
├── internal/
│   ├── app/                 the boundary: the only package that imports the UI/CLI framework
│   ├── library/             orchestrator: holds the packages below and sequences them
│   ├── scanner/             one capability per package
│   ├── store/               the only package with SQL
│   ├── thumbs/
│   ├── settings/
│   └── testutil/            helpers shared by tests in more than one package
└── go.mod
```

### 1.3 Library (importable module)

Root package is the public API. `internal/` for implementation details. No `cmd/` unless the library ships a tool. No `pkg/`.

### 1.4 Rules that apply to every layout

- `internal/` is a compiler-enforced visibility rule, not a folder for libraries: a package under `internal/` can be imported only by code rooted at the parent of that folder. The question for every package is "may anyone outside this module import this?". In an application the answer is always no, so all application code (HTTP, services, storage, domain, adapters) lives under `internal/` and can be refactored without breaking anyone. Only `cmd/` stays outside, because `main` packages cannot be imported anyway. In a library it is the other way around: the public API lives at the root and `internal/` holds what the author does not promise as stable.
- No `pkg/` at any level. No `utils/`, `helpers/`, `common/`, `base/`, `models/`, `services/`, `dto/` anywhere.
- One binary per `cmd/<name>/` folder. `main.go` contains `main` and `run` and nothing else of substance.
- Tests sit next to the code they test. A `testutil` package holds helpers used by more than one package.

---

## 2. Packages and naming

- **Name packages after what they do**, not after the layer they are in: `invoice`, `scanner`, `memstore`, `api`. The test is whether `pkg.Type` reads well: `note.Note`, `memstore.Store`, `http.Server`.
- **No stuttering.** `memstore.Store`, not `memstore.MemStore`. `note.Service`, not `note.NoteService`.
- **Never shadow the standard library.** No package named `http`, `errors`, `log`, `time`, `context` or `sql`. The transport layer is `api`, the error package is `apperr`. A shadowing name forces an alias in every file that imports both and confuses every reader.
- Package names are lowercase, one word, no underscores, no plural unless the package is a collection (`handlers` is fine).
- Exported identifiers have a doc comment that starts with the identifier name.
- Every package has a package comment in exactly one file (`doc.go` if the package has more than three files).
- Constructors are `New` (one per package) or `NewX`. They accept interfaces and return concrete types (`*Service`, never an interface).
- Acronyms keep their case: `userID`, `HTTPClient`, `parseURL`.
- Receivers are one or two letters, consistent within a type (`s *Service`, `c Client`). Value receiver when the method does not mutate and the struct is small; pointer receiver otherwise, and never mix on one type.
- Getter methods have no `Get` prefix: `c.Name()`, not `c.GetName()`.
- Enum-like constants are typed strings or ints with a prefix: `StatusDraft`, `StatusSent`.

---

## 3. Interfaces and dependencies

- **Declare interfaces where they are consumed**, in the package that calls them, not next to the implementation. The implementing package must not import the interface.
- **Interfaces are small**: one to three methods. If a function needs only `Get`, give it an interface with only `Get`. No `Repository` with twenty methods.
- **Accept interfaces, return structs.**
- The implementation does not know the interface exists. Verify satisfaction at compile time in the implementation's test file if useful: `var _ note.Store = (*Store)(nil)`.
- No dependency injection framework. `main` (or `run`) wires the graph by hand. If wiring exceeds about 50 lines, split it into a `newHandlers` or `newServices` function in the same place. Still no framework.
- Functional options (`WithClock`, `WithLogger`) for optional constructor parameters. An input struct for more than five required parameters.
- No package-level mutable state. No global `db`, `logger` or `config` variables. No `init()`.

---

## 4. `main`, `run` and configuration

```go
func main() {
    if err := run(context.Background(), os.Args[1:], os.Stderr); err != nil {
        slog.Error("<binary> failed", "err", err)
        os.Exit(1)
    }
}

func run(ctx context.Context, args []string, stderr io.Writer) error {
    cfg, err := config.Load()
    if err != nil {
        return err
    }
    // connect, wire, serve, shut down
}
```

- `main` does two things: call `run` and convert its error into an exit code. `log.Fatal` and `os.Exit` appear only in `main`.
- `run` takes `ctx`, `args` and `stderr` so it can be tested.
- Shutdown: `signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)`, then `srv.Shutdown` with a timeout derived from `context.WithoutCancel(ctx)`.
- Background goroutines (cron-like jobs, cleanup) start in `run` with the same `ctx` and stop when it is cancelled.
- Configuration comes from environment variables, read once in `config.Load()` into a `Config` struct. Nothing else calls `os.Getenv`. Required values with no default fail at boot with a clear message, not on first use.
- Secrets are validated at boot (decode the key, parse the URL) so a bad value fails on deploy, not on the first request that needs it.
- `.env.example` is committed, `.env` is not.

---

## 5. Errors

- Wrap every error with what you were doing: `fmt.Errorf("invoices: load %s: %w", id, err)`. Lowercase, no trailing punctuation, `%w` so `errors.Is` and `errors.As` work through layers.
- Never ignore an error silently. `_ = f.Close()` only with a comment saying why it is safe.
- Never compare error strings. Use sentinel errors (`var ErrNotFound = errors.New("not found")`) or typed errors and `errors.Is` / `errors.As`.
- Sentinel errors live in the package that produces them (`storage.ErrNotFound`, `note.ErrInvalid`). The service translates storage sentinels into application errors; the transport layer translates application errors into status codes. Each translation happens in exactly one place.
- Client-visible failures in an API are a typed application error with a stable machine-readable code:

```go
type AppError struct {
    Code    string            `json:"code"`     // "not_found", "validation_error", "conflict", "forbidden"
    Message string            `json:"message"`
    Fields  map[string]string `json:"fields,omitempty"`
    Status  int               `json:"-"`
    Err     error             `json:"-"`
}
```

- Anything that is not an `AppError` is a 500: logged in full with the request id, reported to the client as a generic message. Internals never leak.
- `errors.Join` for collecting multiple validation errors.
- No `panic` in library or application code. Recover only in the HTTP middleware, to turn a panic into a 500 and a log line.

---

## 6. Context

- `ctx context.Context` is always the first parameter and is always called `ctx`.
- Never store a context in a struct. Pass it.
- Every function that does I/O, waits, or calls something that does, takes a context. In-memory implementations still check `ctx.Err()` so the habit is uniform.
- Context values are for request-scoped identity only (user id, tenant id, request id). Use unexported key types and typed accessor functions (`respond.UserID(ctx)`). Never pass dependencies or business data through the context.
- Deadlines: HTTP server timeouts on the server, `context.WithTimeout` for outbound calls, cancellation propagated to background work.

---

## 7. Logging

- `log/slog`, structured, key/value pairs: `slog.Info("invoice sent", "invoice_id", id, "to", email)`.
- One logger, created in `run`, passed explicitly to the packages that need it (`*slog.Logger` as a constructor argument) or used via `slog.Default()` set once in `run`. Never a package-level logger variable of your own.
- Log at the edge: middleware and handlers log requests and failures. Services and storage return errors; they do not log them. A failure must appear in the log exactly once.
- Use `slog.ErrorContext` / `InfoContext` so request-scoped attributes (request id) are attached by the handler.
- Levels: `Debug` for developers, `Info` for lifecycle and business events, `Warn` for degraded but working, `Error` for failures that need a human. Nothing else.
- Never log secrets, tokens, passwords, full card numbers or personal data beyond an id.

---

## 8. Concurrency

- Every goroutine has a clear owner and a clear end. If you cannot say who stops it and when, do not start it.
- The sender closes the channel. Never the receiver.
- `sync.WaitGroup` or `golang.org/x/sync/errgroup` to wait for goroutines. `errgroup.WithContext` when the first failure should cancel the rest.
- Protect shared state with a mutex held for the shortest possible time, or avoid sharing. Prefer passing values through channels over sharing memory with locks, but do not build a channel pipeline where a mutex and a map would do.
- Tests run with `-race`. A race detector failure is a bug, never flaky.
- Background work started by a request must not use the request context (it is cancelled when the response is written). Use `context.WithoutCancel(r.Context())` with its own timeout.
- Worker pools have a bounded size. No unbounded `go func()` per item.

---

## 9. HTTP API conventions

### 9.1 Routing

- Go 1.22+ `http.ServeMux` with method and path patterns (`"GET /invoices/{id}"`) for simple services. chi (or equivalent) only when route groups with different middleware stacks are needed.
- One route table in `server.go`, grouped by area in `mount<Area>` functions. A reader finds every endpoint in one file.
- Middleware is a function `http.Handler → http.Handler`. Order: recover, request id, logging, CORS, auth, tenant guard.

### 9.2 Handlers

```go
func (h *Clients) Create(w http.ResponseWriter, r *http.Request) error {
    var in clients.CreateInput
    if err := respond.Decode(r, &in); err != nil {
        return err
    }
    c, err := h.svc.Create(r.Context(), respond.CompanyID(r.Context()), in)
    if err != nil {
        return err
    }
    respond.JSON(w, http.StatusCreated, c)
    return nil
}
```

- Signature `func (h *X) Action(w http.ResponseWriter, r *http.Request) error`, mounted through an adapter that maps the returned error to the JSON error body.
- A handler does exactly: read path and query params, decode the body, read identity from the context, call **one** service method, write the response.
- Zero SQL, zero database driver imports, zero business rules, zero error-to-status mapping.
- Decode with `http.MaxBytesReader` and `DisallowUnknownFields` where the API is strict.

### 9.3 DTOs

- Request and response types live next to the handler (or in the service input types) with `json` tags. Domain types are not serialized directly when the API shape differs from the domain shape.
- Partial updates (`PATCH`) use pointer fields. `nil` means unchanged. For slices, `nil` means unchanged and `[]` means clear.
- Money is a string in JSON. Times are RFC 3339 in UTC. IDs are strings.

### 9.4 Responses

- Success: `{"data": ...}`.
- List: `{"items": [...], "page": 1, "size": 25, "total": 140}`.
- Error: `{"error": {"code": "validation_error", "message": "Invalid input", "fields": {"email": "must be a valid email"}}}`.
- Status codes: 200 read/update, 201 create, 204 only when there is truly no body, 400 validation, 401 not signed in, 403 signed in but not allowed, 404 not found **and** cross-tenant access (never reveal existence), 409 conflict, 500 everything else.

### 9.5 Pagination, filtering, identity

- `?page=` and `?size=` with defaults (1, 25) and a hard cap (100), parsed in one helper.
- Filters are explicit query parameters parsed in the handler into a typed input struct. No generic query languages.
- Identity (user id, tenant id, roles) is placed in the context by the auth middleware and read through typed accessors. Handlers never read cookies or `Authorization` headers.

### 9.6 Server settings

Always set `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout` and `IdleTimeout`. A server without timeouts is a denial-of-service waiting to happen.

---

## 10. Database

### 10.1 Storage layer rules

- Every SQL statement in the application lives in `storage/<driver>`. One file per table or aggregate.
- Functions take `(ctx, db DBTX, ...)` where `DBTX` is the minimal interface shared by the pool and a transaction, so the same function works inside and outside a transaction:

```go
type DBTX interface {
    Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
    Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
    QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
}
```

- Transactions are a service concern, opened with a `WithTx(ctx, func(tx DBTX) error)` helper that commits on nil, rolls back on error or panic.
- Storage returns domain types and sentinel errors (`ErrNotFound`, `ErrConflict`). Never HTTP types, never driver errors unwrapped.
- Columns are listed explicitly. No `SELECT *`. Keep a `const xColumns` and a `scanX` function per aggregate.
- Parameters are numbered in order. One statement per function.
- **Multi-tenant: every tenant query filters by the tenant id. No exceptions.** This is reviewed on every PR.
- Raw SQL with the driver is the default. sqlc is acceptable for large schemas. ORMs are not used: they hide the query, and the tenant filter is the one thing that must never be hidden.

### 10.2 Migrations

- Plain `.sql` files, timestamp-prefixed, run by a migration tool (goose or equivalent) from `storage/<driver>/migrations/`.
- A migration that has been applied anywhere (local, staging, production) is never edited. Fix it with a new migration.
- Schema changes and data backfills are separate migrations.
- Migrations run before the application starts (deploy step or boot), never lazily on first request.

### 10.3 Data rules

- Money: `numeric(15,2)` (or the precision the domain needs) in the database, `decimal.Decimal` in Go, string in JSON. Never `float64`.
- Time: stored in UTC, `time.Now().UTC()` in code, `timestamptz` columns.
- IDs: strings, generated in Go (ULID/UUID/time-ordered), never database sequences exposed to clients.
- No JSON columns for business data that must be queried or reported on. Columns are searchable, JSON blobs are not.
- Soft delete only when the domain needs history. Otherwise delete.

---

## 11. Testing

### 11.1 What to test where

| Layer | Test type | Needs |
|---|---|---|
| domain | unit, table-driven, exhaustive | nothing |
| service | unit with fake storage | nothing |
| http | `httptest` against the real router with fakes or an in-memory store | nothing |
| storage | integration against a real database | Docker or `TEST_DATABASE_URL` |
| end to end | smoke scenarios against a running stack | full environment, optional |

### 11.2 Rules

- Tests live next to the code: `x_test.go`.
- External test package (`package foo_test`) when testing the public API. Internal package only to reach unexported helpers.
- Table-driven: the `name` field first, `t.Run(tt.name, func(t *testing.T) {...})`, `t.Parallel()` where tests are independent.
- Names: `TestType_Method_Case` or `TestFunc_Case`.
- Failure messages: `t.Errorf("Func(%v) = %v, want %v", in, got, want)`. Readable without opening the code.
- `t.Helper()` in every helper. `t.Cleanup` instead of manual defers. `t.TempDir()` for files.
- **Fakes, not mocks.** A fake is a small real implementation of the consumer interface (an in-memory store with a map). No gomock, no mockery, no testify/mock unless a reviewer agrees there is no alternative. `testify/assert` is acceptable if the project already uses it, not required.
- Determinism: inject the clock (`WithClock`) and the id generator (`WithIDGenerator`). No `time.Now()` or random ids inside a unit under test.
- Fixtures: builder functions in Go (`seed.Invoice().In(company).Build(t)`), not SQL fixture files that go stale silently.
- `go test -race ./...` in CI. Flaky tests are fixed or deleted, never retried.

### 11.3 The tenant isolation test

For every endpoint that takes a resource id: sign in as tenant A, create the resource, sign in as tenant B, request it, assert 404. This single pattern matters more than any coverage number.

### 11.4 Coverage

Domain: 85 percent or above. Services and handlers: 70 percent or above. Do not gate merges on the number. Gate on the critical paths being covered.

---

## 12. Size limits (hard, enforced by lint)

| Unit | Limit |
|---|---|
| File | 400 lines |
| Function | 60 lines, 40 statements |
| HTTP handler function | 40 lines |
| Cyclomatic complexity | 15 |
| Line length | 140 characters |
| Function parameters | 5 (use an input struct beyond that) |
| Interface methods | 3 (split beyond that) |

A function that hits the limit is split, not exempted. Test files are exempt from length limits only.

---

## 13. Tooling

### 13.1 Required on every change

```
gofmt -l .            # must print nothing
go vet ./...
golangci-lint run ./...
go build ./...
go test -race ./...
```

`make check` runs all of them. Run it before every push.

### 13.2 `.golangci.yml`

```yaml
version: "2"
run:
  timeout: 5m
linters:
  default: standard        # errcheck, govet, ineffassign, staticcheck, unused
  enable:
    - errorlint
    - gocyclo
    - funlen
    - lll
    - misspell
    - revive
    - unconvert
    - unparam
  settings:
    gocyclo:
      min-complexity: 15
    funlen:
      lines: 60
      statements: 40
    lll:
      line-length: 140
  exclusions:
    paths:
      - internal/storage/postgres/migrations
    rules:
      - path: _test\.go
        linters: [funlen, lll]
      # Legacy files pending refactor. Remove each path once it complies.
      # - path: internal/api/handlers/legacy\.go
      #   linters: [funlen, gocyclo, lll]
formatters:
  enable:
    - gofmt
    - goimports
```

Legacy exclusions are a to-do list, not a permanent state. Each PR that touches an excluded file brings the touched part into compliance and never makes the file worse.

### 13.3 Makefile

```make
.PHONY: dev test test-unit test-cover lint build check migrate-up migrate-down migrate-create tidy

-include .env
export

dev:
	docker compose up -d postgres
	$(MAKE) migrate-up
	go run github.com/air-verse/air@latest

test:
	go test -race -count=1 ./...

test-unit:
	go test -race -count=1 ./internal/domain/... ./internal/service/...

test-cover:
	go test -coverprofile=cover.out ./... && go tool cover -html=cover.out -o cover.html

lint:
	gofmt -l . && go vet ./... && golangci-lint run ./...

build:
	CGO_ENABLED=0 go build -ldflags "-X <module>/internal/version.Version=$(VERSION)" -o bin/<binary> ./cmd/<binary>

check: lint build test

migrate-up:
	go run github.com/pressly/goose/v3/cmd/goose@latest -dir internal/storage/postgres/migrations postgres "$(DATABASE_URL)" up

migrate-down:
	go run github.com/pressly/goose/v3/cmd/goose@latest -dir internal/storage/postgres/migrations postgres "$(DATABASE_URL)" down

migrate-create:
	go run github.com/pressly/goose/v3/cmd/goose@latest -dir internal/storage/postgres/migrations create $(NAME) sql

tidy:
	go mod tidy
```

### 13.4 CI

Runs on every pull request and on `main`: gofmt check, `go vet`, golangci-lint, `go build`, `go test -race`. Red CI blocks merge via branch protection. The whole run must stay under a few minutes; if it does not, fix the tests, do not skip them.

### 13.5 Build and deploy

- `CGO_ENABLED=0` static binaries. Version injected with `-ldflags "-X .../internal/version.Version=vX.Y.Z"` and exposed on a `/version` endpoint.
- Multi-stage Dockerfile: build stage on the official Go image, final stage on a minimal base with `ca-certificates` and `tzdata`.
- Only tagged commits are deployed. Tag format `vX.Y.Z`: minor for features, patch for fixes.

---

## 14. Dependencies

- Standard library first. Before adding a module, state what the standard library lacks.
- Every new dependency is named in the PR description with its purpose.
- Prefer small, focused, actively maintained modules. Check the last release date and open issues.
- Pin with `go.mod`; `go mod tidy` is part of `make check` and must produce no diff.
- Run `govulncheck ./...` in CI or at least before each release.

Accepted without discussion: `golang.org/x/*`, `github.com/jackc/pgx/v5`, `github.com/go-chi/chi/v5`, `github.com/shopspring/decimal`, `github.com/pressly/goose/v3`, `github.com/testcontainers/testcontainers-go`.

Not used: ORMs, dependency injection frameworks, mock generators, logging frameworks other than `slog`, web frameworks that replace `net/http`.

---

## 15. Git and pull requests

- Branch per task: `feature/<name>`, `fix/<name>`, `chore/<name>`. Nothing is committed directly to `main`.
- Small commits, one per finished step. Imperative subject under 72 characters, body explains why when it is not obvious.
- A pull request does one thing. Refactoring and behaviour changes are separate PRs.
- When touching a legacy file that violates these rules, bring the part you touch into compliance and leave the rest for its own PR. Never make a file worse.

### PR checklist

The author confirms before requesting review:

- [ ] `make check` passes locally (format, vet, lint, build, tests with `-race`).
- [ ] No SQL outside `storage/`. No business rules in handlers. No HTTP types in services.
- [ ] Every new tenant query filters by tenant id.
- [ ] Every error is wrapped with context or deliberately handled; no `_ =` without a comment.
- [ ] New interfaces are declared at the consumer and have at most three methods.
- [ ] No new package-level mutable state, no `init()`, no stored contexts.
- [ ] Domain and service changes have unit tests; new endpoints have `httptest` tests; new endpoints that take an id have the tenant isolation test.
- [ ] Migrations are new files; no applied migration was edited.
- [ ] New dependencies are listed in the PR description with their justification.
- [ ] Size limits respected; no lint exclusions added without a removal plan.
- [ ] Exported identifiers have doc comments; package comment exists.

The reviewer checks the same list and, in addition, that names read well (`pkg.Type`), that the dependency direction is respected, and that the change is as small as it can be.

---

## Appendix A: `respond.Handler` adapter

```go
type HandlerFunc func(w http.ResponseWriter, r *http.Request) error

func Handler(fn HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        err := fn(w, r)
        if err == nil {
            return
        }
        var ae *apperr.AppError
        if errors.As(err, &ae) && ae.Status < 500 {
            WriteError(w, ae)
            return
        }
        slog.ErrorContext(r.Context(), "request failed",
            "method", r.Method, "path", r.URL.Path, "error", err)
        WriteError(w, apperr.Internal(err))
    }
}
```

## Appendix B: `WithTx`

```go
func (d *DB) WithTx(ctx context.Context, fn func(tx DBTX) error) (err error) {
    tx, err := d.pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer func() {
        if p := recover(); p != nil {
            _ = tx.Rollback(ctx)
            panic(p)
        }
        if err != nil {
            _ = tx.Rollback(ctx)
        }
    }()
    if err = fn(tx); err != nil {
        return err
    }
    if err = tx.Commit(ctx); err != nil {
        return fmt.Errorf("commit tx: %w", err)
    }
    return nil
}
```

## Appendix C: table-driven test skeleton

```go
func TestService_Create(t *testing.T) {
    t.Parallel()
    fixed := time.Date(2026, 1, 1, 12, 0, 0, 0, time.UTC)

    tests := []struct {
        name    string
        title   string
        wantErr error
    }{
        {name: "valid", title: "hello"},
        {name: "empty title", title: "  ", wantErr: note.ErrInvalid},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            svc := note.NewService(newFakeStore(),
                note.WithClock(func() time.Time { return fixed }),
                note.WithIDGenerator(func() string { return "id-1" }),
            )

            got, err := svc.Create(context.Background(), tt.title, "")

            if !errors.Is(err, tt.wantErr) {
                t.Fatalf("Create(%q) error = %v, want %v", tt.title, err, tt.wantErr)
            }
            if tt.wantErr != nil {
                return
            }
            if got.ID != "id-1" {
                t.Errorf("Create(%q).ID = %q, want %q", tt.title, got.ID, "id-1")
            }
        })
    }
}
```

## Appendix D: quick reference

| Topic | Rule |
|---|---|
| Layout | `cmd/` + `internal/`; no `pkg/`, no `utils/`; transport is `api` |
| Packages | named by capability; `pkg.Type` reads well; no stutter |
| Interfaces | at the consumer; 1 to 3 methods; accept interfaces, return structs |
| Wiring | by hand in `run`; no DI framework |
| Globals | none; no `init()` |
| Errors | wrap with `%w`; sentinels + `errors.Is`; map to status once |
| Context | first param, named `ctx`, never stored |
| Logging | `slog`, structured, at the edge, once per failure |
| Concurrency | clear owner, sender closes, bounded pools, `-race` |
| Handlers | decode, one service call, encode; return `error`; 40 lines |
| Storage | all SQL here; `DBTX`; explicit columns; tenant filter always |
| Money / time / ids | `decimal` / UTC / string |
| Tests | next to code; table-driven; fakes not mocks; isolation test |
| Limits | file 400, func 60, handler 40, params 5, complexity 15 |
| CI | gofmt, vet, lint, build, test -race; red blocks merge |
