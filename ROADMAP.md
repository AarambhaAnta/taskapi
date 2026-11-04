# Build a REST API in Go: Step-by-Step Learning Roadmap

You will build **TaskAPI**, a multi-user to-do backend, from an empty folder to a tagged release. Every feature you asked for has its own phase, and each phase teaches the Go concepts it needs.

| You asked for | What you will use | Phase |
|---|---|---|
| Version control | Git (branches, commits, tags) | 1, and every phase after |
| Air (live reload) | `air-verse/air` | 2 |
| REST API | Standard library `net/http` | 3, 7 |
| SQLite | `database/sql` + `modernc.org/sqlite` | 4, 5 |
| Migrations | `pressly/goose` | 4, 8 |
| Auth | bcrypt + JWT (`golang-jwt/jwt`) | 6 |
| API versioning | `/api/v1` and `/api/v2` URL prefixes | 8 |
| Swagger | `swaggo/swag` + `http-swagger` | 9 |

Tests come in at Phase 5 and are expanded in Phase 10. Phase 11 adds a Makefile and your first release.

## How to use this roadmap

- **Type the code by hand.** Copy-pasting teaches you very little. Typing makes you notice the syntax and the compiler errors teach you the rest.
- **Finish each phase's checkpoint before moving on.** Every phase ends with the project compiling and something you can run.
- **Read the "Go concepts" box first.** It explains the new syntax you are about to type.
- **Blocks labelled `File:` show the complete file.** If the file already exists, replace all of its contents. Unlabelled blocks are commands or illustrations.
- **Do the exercises.** They are where you stop following along and start writing Go yourself.
- **Replace `yourname`** in `github.com/yourname/taskapi` with your GitHub username throughout.

Why the standard library rather than Gin or Echo? Since Go 1.22, `net/http` routes by method and path parameter, so you can build a real API without a framework. Learning it first also makes every Go framework easier to pick up later, because they are all built on the same interfaces.

---

## Table of contents

- [Phase 0: Tools and orientation](#phase-0-tools-and-orientation)
- [Phase 1: Git, the Go module and your first server](#phase-1-git-the-go-module-and-your-first-server)
- [Phase 2: Live reload with Air](#phase-2-live-reload-with-air)
- [Phase 3: Project structure, config, middleware and graceful shutdown](#phase-3-project-structure-config-middleware-and-graceful-shutdown)
- [Phase 4: SQLite and migrations](#phase-4-sqlite-and-migrations)
- [Phase 5: Domain models and the store layer](#phase-5-domain-models-and-the-store-layer)
- [Phase 6: Authentication](#phase-6-authentication)
- [Phase 7: Task CRUD REST API](#phase-7-task-crud-rest-api)
- [Phase 8: API versioning](#phase-8-api-versioning)
- [Phase 9: Swagger documentation](#phase-9-swagger-documentation)
- [Phase 10: Testing](#phase-10-testing)
- [Phase 11: Makefile and your first release](#phase-11-makefile-and-your-first-release)
- [Phase 12: Where to go next](#phase-12-where-to-go-next)
- [Appendix A: Go syntax cheat sheet](#appendix-a-go-syntax-cheat-sheet)
- [Appendix B: Troubleshooting](#appendix-b-troubleshooting)

---

## Phase 0: Tools and orientation

**Goal:** have the tools installed and understand where you are heading.

### Step 0.1: Install the tools

You need Go 1.24 or newer. Check what you have:

```bash
go version
git --version
sqlite3 --version
```

Install the three command-line tools the project uses:

```bash
go install github.com/air-verse/air@latest
go install github.com/swaggo/swag/cmd/swag@latest
go install github.com/pressly/goose/v3/cmd/goose@latest
```

`go install` puts binaries in `$(go env GOPATH)/bin`. If your shell can't find `air` afterwards, add this line to `~/.zshrc` and restart your terminal:

```bash
export PATH="$PATH:$(go env GOPATH)/bin"
```

### Step 0.2: The API you will build

| Method | Path | Auth? | Purpose |
|---|---|---|---|
| GET | `/healthz` | No | Health check |
| GET | `/swagger/index.html` | No | Interactive API docs |
| POST | `/api/v1/auth/register` | No | Create an account |
| POST | `/api/v1/auth/login` | No | Get a JWT |
| GET | `/api/v1/auth/me` | Yes | Current user |
| POST | `/api/v1/tasks` | Yes | Create a task |
| GET | `/api/v1/tasks` | Yes | List all your tasks (plain array) |
| GET | `/api/v1/tasks/{id}` | Yes | Get one task |
| PUT | `/api/v1/tasks/{id}` | Yes | Replace a task |
| DELETE | `/api/v1/tasks/{id}` | Yes | Delete a task |
| GET | `/api/v2/tasks` | Yes | List with pagination and filtering |

### Step 0.3: The final file structure

```text
project1/
├── cmd/
│   └── api/
│       └── main.go              # Entry point: wires everything together
├── internal/                    # Private code; other modules can't import it
│   ├── auth/
│   │   ├── auth_test.go
│   │   ├── context.go           # Store/read the user ID in a request context
│   │   ├── password.go          # bcrypt hashing
│   │   └── token.go             # JWT issue and verify
│   ├── config/
│   │   └── config.go            # Reads environment variables
│   ├── database/
│   │   └── database.go          # Opens SQLite, runs migrations
│   ├── domain/
│   │   ├── errors.go            # Shared error values
│   │   ├── task.go
│   │   └── user.go
│   ├── handler/
│   │   ├── v1/
│   │   │   ├── auth.go          # Register, login, me
│   │   │   ├── routes.go
│   │   │   ├── task.go          # Task CRUD
│   │   │   └── task_test.go
│   │   └── v2/
│   │       ├── routes.go
│   │       └── task.go          # Paginated list
│   ├── httpx/
│   │   └── httpx.go             # JSON request/response helpers
│   ├── middleware/
│   │   ├── auth.go              # Rejects requests without a valid JWT
│   │   └── middleware.go        # Logging, panic recovery, chaining
│   ├── server/
│   │   ├── router.go            # Mounts versions, swagger and middleware
│   │   └── router_test.go       # End-to-end tests
│   └── store/
│       ├── task_page.go         # Pagination query (added for v2)
│       ├── task_store.go
│       ├── task_store_test.go
│       └── user_store.go
├── migrations/
│   ├── 00001_create_users.sql
│   ├── 00002_create_tasks.sql
│   ├── 00003_add_tasks_user_created_index.sql
│   └── embed.go                 # Bakes the SQL files into the binary
├── docs/                        # Generated by swag; don't edit by hand
├── .air.toml
├── .env                         # Your secrets; never committed
├── .env.example
├── .gitignore
├── go.mod
├── go.sum
└── Makefile
```

### Step 0.4: How a request flows through the layers

```text
HTTP request
    │
    ▼
middleware   Logger → Recover → (RequireAuth on protected routes)
    │
    ▼
handler      Parse and validate the request, write the response. Knows HTTP, not SQL.
    │
    ▼
store        Run SQL and map rows to structs. Knows SQL, not HTTP.
    │
    ▼
SQLite
```

Each layer only talks to the one below it. `domain` holds the plain structs and errors that all the layers share. This separation is what lets you test a handler with a fake store in Phase 10, or swap SQLite for Postgres later without touching a handler.

---

## Phase 1: Git, the Go module and your first server

**Goal:** a version-controlled Go project with a running HTTP server.

> **Go concepts**
> - **Module:** a project with a `go.mod` file. It declares the project's import path and its dependencies.
> - **Package:** a folder of `.go` files that all start with the same `package` line. `package main` is special: it builds into a program, and `func main()` is where that program starts.
> - **Imports and visibility:** names starting with a capital letter (`http.NewServeMux`) are exported. Lowercase names are private to their package.
> - **`:=`** declares a variable and infers its type. **`if err := f(); err != nil`** declares `err` for just that `if` block.
> - **Errors are values.** Go has no exceptions. Functions return an `error`, and you check it.

### Step 1.1: Initialise Git

Run everything in this roadmap from the project root (`project1/`).

```bash
git init -b main
```

Create the ignore file **before your first commit**, so secrets and build output can never get in.

**File: `.gitignore`**
```gitignore
# Build output
/tmp/
/bin/

# Local SQLite databases
*.db
*.db-shm
*.db-wal

# Secrets
.env

# Air
build-errors.log

# OS / editor
.DS_Store
.idea/
.vscode/
```

### Step 1.2: Create the Go module

```bash
go mod init github.com/yourname/taskapi
```

This creates `go.mod`. Every import of your own code will now start with that path.

### Step 1.3: Write the server

Go convention puts each program's entry point under `cmd/<program-name>/`.

**File: `cmd/api/main.go`**
```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

func main() {
	// A ServeMux matches incoming requests to handlers.
	mux := http.NewServeMux()

	// Since Go 1.22, a pattern can include the HTTP method.
	mux.HandleFunc("GET /healthz", healthz)

	log.Println("listening on :8080")
	if err := http.ListenAndServe(":8080", mux); err != nil {
		log.Fatal(err)
	}
}

// Every handler has this signature: w is where you write the response,
// r holds everything about the request.
func healthz(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}
```

### Checkpoint

```bash
go run ./cmd/api
```

In a second terminal:

```bash
curl -i localhost:8080/healthz      # 200 with {"status":"ok"}
curl -i -X POST localhost:8080/healthz   # 405 Method Not Allowed: the mux did that for you
```

Press `Ctrl+C` to stop the server.

### Step 1.4: Your first commit

```bash
git add .
git status        # Always read this before committing
git commit -m "feat: add health check server"
```

### The Git workflow for the rest of this project

From here on, do each phase on its own branch, as you would on a team:

```bash
git switch -c phase-2-air        # Start a branch
# ...do the work, committing in small steps...
git add .
git commit -m "chore: add air live reload"
git switch main                  # Go back to main
git merge --no-ff phase-2-air    # Merge, keeping a record of the branch
git branch -d phase-2-air        # Delete the finished branch
```

Commit message prefixes used here follow the [Conventional Commits](https://www.conventionalcommits.org) style: `feat:` new behaviour, `fix:` bug fix, `chore:` tooling, `docs:` documentation, `test:` tests, `refactor:` restructuring with no behaviour change.

To put your work on GitHub, create an empty repository there, then:

```bash
git remote add origin git@github.com:yourname/taskapi.git
git push -u origin main
```

### Exercises

1. Add a `GET /version` route that returns `{"version":"0.0.1"}`.
2. Remove the `"GET "` from the pattern and see how the `POST` request behaves. Put it back afterwards.
3. Run `git log --oneline` and `git diff` after making a change, and read their output.

---

## Phase 2: Live reload with Air

**Goal:** the server rebuilds and restarts whenever you save a file.

### Step 2.1: Create the config

`air init` generates a config with every option listed. This smaller version contains only what the project needs.

**File: `.air.toml`**
```toml
root = "."
tmp_dir = "tmp"

[build]
  # How to compile. Our main package isn't in the root, so name its path.
  cmd = "go build -o ./tmp/main ./cmd/api"
  # What to run after a successful build.
  entrypoint = ["./tmp/main"]
  # Which files trigger a rebuild.
  include_ext = ["go", "sql"]
  # Generated code and build output must be excluded, or Air can loop forever.
  exclude_dir = ["tmp", "vendor", "docs"]
  exclude_regex = ["_test\\.go$"]
  # Wait this many milliseconds after a change, so one save means one rebuild.
  delay = 500
  # Stop the old server if the new code doesn't compile, so you never test stale code.
  stop_on_error = true
  # Send Ctrl+C rather than killing the process; Phase 3's shutdown relies on it.
  send_interrupt = true
  kill_delay = "2s"

[misc]
  clean_on_exit = true
```

> Air's options change between releases. Older versions use `bin = "./tmp/main"` instead of `entrypoint`. If Air complains, run `air init` and compare with the file it generates.

### Checkpoint

```bash
air
```

With Air running, change `"ok"` to `"healthy"` in `main.go` and save. Air rebuilds, and `curl localhost:8080/healthz` shows the new value. Change it back.

Now break the code on purpose by deleting a closing brace and saving. Air prints the compiler error and waits. Fix the brace and it recovers.

### Commit

```bash
git add .
git commit -m "chore: add air live reload"
```

---

## Phase 3: Project structure, config, middleware and graceful shutdown

**Goal:** turn the single file into a real project layout, with the plumbing every later handler will use.

> **Go concepts**
> - **`internal/`:** the compiler stops other modules from importing anything under this folder.
> - **Structs and tags:** `json:"error"` tells `encoding/json` what key to use.
> - **Generics:** `func DecodeJSON[T any](...) (T, error)` works for any type `T`.
> - **Closures and higher-order functions:** middleware is a function that takes a handler and returns a new one wrapping it.
> - **Struct embedding:** putting `http.ResponseWriter` in a struct with no field name gives that struct all its methods, and you can override just the ones you want.
> - **`defer`:** runs a call when the surrounding function returns. `recover()` only works inside a deferred function.
> - **Goroutines and channels:** `go f()` runs `f` concurrently. A channel (`chan error`) passes a value between goroutines safely.
> - **`select`:** waits on several channels and proceeds with whichever is ready first.
> - **`context.Context`:** carries cancellation signals and deadlines through your program.

### Step 3.1: Install the dependency

```bash
go get github.com/joho/godotenv
```

`godotenv` loads a `.env` file into environment variables during development. In production, you would set real environment variables instead.

### Step 3.2: Configuration

**File: `.env.example`**
```bash
PORT=8080
DATABASE_PATH=app.db
# Generate your own with: openssl rand -hex 32
JWT_SECRET=replace-me-with-the-output-of-openssl-rand-hex-32
JWT_TTL=24h
```

Create your real, git-ignored copy with a random secret:

```bash
cp .env.example .env
sed -i '' "s/^JWT_SECRET=.*/JWT_SECRET=$(openssl rand -hex 32)/" .env
```

(That `sed -i ''` form is for macOS. On Linux, use `sed -i` with no quotes after it.)

`Load` reads every setting once at startup and fails immediately if something is wrong, rather than failing on some later request.

**File: `internal/config/config.go`**
```go
package config

import (
	"errors"
	"fmt"
	"os"
	"time"

	"github.com/joho/godotenv"
)

// Config holds every setting the application needs.
type Config struct {
	Port         string
	DatabasePath string
	JWTSecret    string
	JWTTTL       time.Duration
}

// Load reads configuration from the environment, using a .env file if present.
func Load() (Config, error) {
	// The .env file is optional, so the error is deliberately ignored.
	_ = godotenv.Load()

	ttl, err := time.ParseDuration(getEnv("JWT_TTL", "24h"))
	if err != nil {
		return Config{}, fmt.Errorf("parse JWT_TTL: %w", err)
	}

	cfg := Config{
		Port:         getEnv("PORT", "8080"),
		DatabasePath: getEnv("DATABASE_PATH", "app.db"),
		JWTSecret:    os.Getenv("JWT_SECRET"),
		JWTTTL:       ttl,
	}

	if len(cfg.JWTSecret) < 32 {
		return Config{}, errors.New("JWT_SECRET must be at least 32 characters")
	}
	return cfg, nil
}

func getEnv(key, fallback string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return fallback
}
```

The `%w` in `fmt.Errorf` **wraps** the original error: it adds context while keeping the original available to `errors.Is` and `errors.As`. You will use it everywhere.

### Step 3.3: JSON helpers

Every handler reads and writes JSON, so that code lives in one place.

**File: `internal/httpx/httpx.go`**
```go
// Package httpx holds helpers shared by all HTTP handlers.
package httpx

import (
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
)

// maxBodyBytes limits request bodies so a client can't exhaust our memory.
const maxBodyBytes = 1 << 20 // 1 MiB

// ErrorResponse is the body of every error reply.
type ErrorResponse struct {
	Error string `json:"error" example:"something went wrong"`
}

// JSON writes data as a JSON response with the given status code.
func JSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(data); err != nil {
		slog.Error("encode response", "err", err)
	}
}

// Error writes an ErrorResponse with the given status code.
func Error(w http.ResponseWriter, status int, message string) {
	JSON(w, status, ErrorResponse{Error: message})
}

// InternalError logs the real error and sends the client a generic message,
// so that internal details never leak out.
func InternalError(w http.ResponseWriter, r *http.Request, err error) {
	slog.Error("internal error", "method", r.Method, "path", r.URL.Path, "err", err)
	Error(w, http.StatusInternalServerError, "internal server error")
}

// DecodeJSON reads the request body into a new value of type T.
func DecodeJSON[T any](w http.ResponseWriter, r *http.Request) (T, error) {
	var v T
	r.Body = http.MaxBytesReader(w, r.Body, maxBodyBytes)

	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields() // Reject typos like {"titel": "..."}
	if err := dec.Decode(&v); err != nil {
		return v, fmt.Errorf("invalid JSON body: %w", err)
	}
	return v, nil
}
```

You call the generic function by naming the type, and Go gives you a value of that type back:

```go
req, err := httpx.DecodeJSON[CreateTaskRequest](w, r)
```

### Step 3.4: Middleware

Middleware wraps a handler to add behaviour before and after it runs. Its signature is always the same:

```go
func(next http.Handler) http.Handler
```

**File: `internal/middleware/middleware.go`**
```go
// Package middleware holds wrappers that run around HTTP handlers.
package middleware

import (
	"log/slog"
	"net/http"
	"runtime/debug"
	"time"

	"github.com/yourname/taskapi/internal/httpx"
)

// Middleware wraps a handler with extra behaviour.
type Middleware func(http.Handler) http.Handler

// Chain applies middleware so that the first one listed runs first.
func Chain(h http.Handler, mws ...Middleware) http.Handler {
	for i := len(mws) - 1; i >= 0; i-- {
		h = mws[i](h)
	}
	return h
}

// statusRecorder remembers the status code written by a handler.
// Embedding http.ResponseWriter gives it all of that interface's methods;
// we then override only WriteHeader.
type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}

// Logger logs one line for every request.
func Logger(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}

		next.ServeHTTP(rec, r)

		slog.Info("request",
			"method", r.Method,
			"path", r.URL.Path,
			"status", rec.status,
			"duration", time.Since(start),
		)
	})
}

// Recover turns a panic in a handler into a 500 response.
func Recover(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				slog.Error("panic", "err", err, "stack", string(debug.Stack()))
				httpx.Error(w, http.StatusInternalServerError, "internal server error")
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

`http.HandlerFunc(...)` is a type conversion, not a function call. It turns an ordinary function into something that satisfies the `http.Handler` interface.

### Step 3.5: The router

**File: `internal/server/router.go`**
```go
// Package server assembles the application's routes and middleware.
package server

import (
	"net/http"

	"github.com/yourname/taskapi/internal/httpx"
	"github.com/yourname/taskapi/internal/middleware"
)

// NewRouter builds the root HTTP handler for the application.
func NewRouter() http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", healthz)

	return middleware.Chain(mux, middleware.Logger, middleware.Recover)
}

func healthz(w http.ResponseWriter, r *http.Request) {
	httpx.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}
```

### Step 3.6: A new `main` with graceful shutdown

Graceful shutdown means that when the server is told to stop, it finishes the requests in progress rather than cutting them off.

A common pattern is to keep `main` tiny and put the real work in `run() error`. That gives you a single place to handle errors, and it lets `defer` statements run, because `os.Exit` skips them.

**File: `cmd/api/main.go`**
```go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/yourname/taskapi/internal/config"
	"github.com/yourname/taskapi/internal/server"
)

// version is replaced at build time (see Phase 11).
var version = "dev"

func main() {
	if err := run(); err != nil {
		slog.Error("fatal", "err", err)
		os.Exit(1)
	}
}

func run() error {
	cfg, err := config.Load()
	if err != nil {
		return err
	}

	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           server.NewRouter(),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	// ctx is cancelled when the process receives Ctrl+C or SIGTERM.
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	// ListenAndServe blocks, so it runs in its own goroutine. The channel is
	// buffered so that the goroutine can always send and then exit.
	serverErr := make(chan error, 1)
	go func() {
		slog.Info("server starting", "addr", srv.Addr, "version", version)
		if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
			serverErr <- err
		}
	}()

	// Wait for whichever happens first: a server failure or a stop signal.
	select {
	case err := <-serverErr:
		return err
	case <-ctx.Done():
		slog.Info("shutting down")
	}

	// Give in-flight requests up to 10 seconds to finish.
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	return srv.Shutdown(shutdownCtx)
}
```

### Checkpoint

```bash
go mod tidy    # Sync go.mod with your imports
air
```

```bash
curl -i localhost:8080/healthz
```

Air's terminal now shows a structured log line for the request. Press `Ctrl+C` and you should see `shutting down`.

To see `Recover` work, temporarily add `panic("boom")` as the first line of `healthz`. The client gets a clean 500, the stack trace and a request line with `status=500` are logged, and the server keeps running. Remove it afterwards.

### Commit

```bash
git add .
git commit -m "feat: add project structure, config, middleware and graceful shutdown"
```

Note what `git status` did **not** list: `.env` is ignored, and `.env.example` is committed as documentation of what's needed.

### Exercises

1. Add a `RequestID` middleware that generates a random ID and sets it as an `X-Request-ID` response header. Hint: `crypto/rand` and `encoding/hex`.
2. Swap the order in `Chain` to `Recover, Logger` and trigger a panic. The client still gets a 500, but the request's log line disappears. Work out why. (Hint: a panic skips everything after `next.ServeHTTP` in `Logger`, unless something inside it has already recovered.)
3. Set `JWT_SECRET=short` in `.env` and confirm the program refuses to start.

---

## Phase 4: SQLite and migrations

**Goal:** a database whose schema is versioned alongside your code.

> **Go concepts**
> - **`database/sql`:** the standard database interface. A separate driver package supplies the SQLite support.
> - **Blank imports:** `import _ "modernc.org/sqlite"` runs that package's `init()` function, which registers the driver, without using any of its names.
> - **`//go:embed`:** compiles files into your binary, so that you deploy one file with the migrations inside it.
> - **Multiple return values** and the `(value, error)` convention.

**Why `modernc.org/sqlite`?** The other popular driver, `mattn/go-sqlite3`, needs a C compiler (CGO). The `modernc` driver is pure Go, so builds stay simple and cross-compiling just works.

**What are migrations?** Numbered SQL scripts that describe every change to your schema. Goose records which ones have run in a `goose_db_version` table, so every database (yours, a teammate's, production) can be brought to the same state. Never edit a migration that has already been committed or run. Add a new one instead.

### Step 4.1: Install the dependencies

```bash
go get modernc.org/sqlite
go get github.com/pressly/goose/v3
```

### Step 4.2: Write the migrations

Goose can create a numbered template file for you, but the folder must already exist:

```bash
mkdir -p migrations
goose -dir migrations -s create create_users sql
goose -dir migrations -s create create_tasks sql
```

(`-s` means sequential numbering, such as `00001`, rather than a timestamp.) Each file has an `Up` section to apply the change, and a `Down` section to undo it. Replace the generated contents with the following:

**File: `migrations/00001_create_users.sql`**
```sql
-- +goose Up
CREATE TABLE users (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    email         TEXT     NOT NULL UNIQUE,
    password_hash TEXT     NOT NULL,
    created_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- +goose Down
DROP TABLE users;
```

**File: `migrations/00002_create_tasks.sql`**
```sql
-- +goose Up
CREATE TABLE tasks (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id     INTEGER  NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    title       TEXT     NOT NULL,
    description TEXT     NOT NULL DEFAULT '',
    done        INTEGER  NOT NULL DEFAULT 0 CHECK (done IN (0, 1)),
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- +goose Down
DROP TABLE tasks;
```

SQLite has no boolean type, so `done` is an integer limited to 0 or 1. `ON DELETE CASCADE` means deleting a user also deletes their tasks, but only when foreign keys are switched on, which SQLite doesn't do by default. The next step handles that.

### Step 4.3: Embed the migrations

**File: `migrations/embed.go`**
```go
// Package migrations embeds the SQL migration files into the binary.
package migrations

import "embed"

//go:embed *.sql
var FS embed.FS
```

There must be no space between `//` and `go:embed`, and the comment must sit directly above the variable.

### Step 4.4: The database package

**File: `internal/database/database.go`**
```go
// Package database opens SQLite connections and runs migrations.
package database

import (
	"context"
	"database/sql"
	"fmt"
	"time"

	"github.com/pressly/goose/v3"
	_ "modernc.org/sqlite" // Registers the "sqlite" driver.

	"github.com/yourname/taskapi/migrations"
)

// Open connects to the SQLite database at path, creating the file if needed.
func Open(path string) (*sql.DB, error) {
	// foreign_keys:  enforce REFERENCES constraints (off by default in SQLite).
	// journal_mode:  WAL lets reads continue while a write is in progress.
	// busy_timeout:  wait up to 5s for a lock rather than failing at once.
	dsn := fmt.Sprintf(
		"file:%s?_pragma=foreign_keys(1)&_pragma=journal_mode(WAL)&_pragma=busy_timeout(5000)",
		path,
	)

	// sql.Open only validates its arguments; it doesn't actually connect.
	db, err := sql.Open("sqlite", dsn)
	if err != nil {
		return nil, fmt.Errorf("open database: %w", err)
	}

	// SQLite allows one writer at a time. A single connection is the simplest
	// way to avoid "database is locked" errors, and is plenty for this project.
	db.SetMaxOpenConns(1)

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := db.PingContext(ctx); err != nil {
		db.Close()
		return nil, fmt.Errorf("ping database: %w", err)
	}
	return db, nil
}

// Migrate applies all pending migrations.
func Migrate(db *sql.DB) error {
	goose.SetBaseFS(migrations.FS)
	if err := goose.SetDialect("sqlite3"); err != nil {
		return fmt.Errorf("set goose dialect: %w", err)
	}
	if err := goose.Up(db, "."); err != nil {
		return fmt.Errorf("run migrations: %w", err)
	}
	return nil
}
```

The trade-off with `SetMaxOpenConns(1)` is that if you hold a transaction or an open `rows` and then run another query, the second query waits forever for the only connection. Always finish with one result before starting the next query.

### Step 4.5: Connect it in `main`

**File: `cmd/api/main.go`**
```go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/yourname/taskapi/internal/config"
	"github.com/yourname/taskapi/internal/database"
	"github.com/yourname/taskapi/internal/server"
)

// version is replaced at build time (see Phase 11).
var version = "dev"

func main() {
	if err := run(); err != nil {
		slog.Error("fatal", "err", err)
		os.Exit(1)
	}
}

func run() error {
	cfg, err := config.Load()
	if err != nil {
		return err
	}

	db, err := database.Open(cfg.DatabasePath)
	if err != nil {
		return err
	}
	defer db.Close()

	if err := database.Migrate(db); err != nil {
		return err
	}

	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           server.NewRouter(),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	// ctx is cancelled when the process receives Ctrl+C or SIGTERM.
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	// ListenAndServe blocks, so it runs in its own goroutine. The channel is
	// buffered so that the goroutine can always send and then exit.
	serverErr := make(chan error, 1)
	go func() {
		slog.Info("server starting", "addr", srv.Addr, "version", version)
		if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
			serverErr <- err
		}
	}()

	// Wait for whichever happens first: a server failure or a stop signal.
	select {
	case err := <-serverErr:
		return err
	case <-ctx.Done():
		slog.Info("shutting down")
	}

	// Give in-flight requests up to 10 seconds to finish.
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	return srv.Shutdown(shutdownCtx)
}
```

### Checkpoint

```bash
go mod tidy
air
```

Goose logs the migrations it applied. Stop Air, then look at the database yourself:

```bash
sqlite3 app.db ".tables"
sqlite3 app.db ".schema tasks"
```

Now try the goose CLI, which is how you inspect and roll back migrations by hand:

```bash
goose -dir migrations sqlite3 ./app.db status     # Which migrations have run?
goose -dir migrations sqlite3 ./app.db down       # Undo the latest one
sqlite3 app.db ".tables"                          # tasks is gone
goose -dir migrations sqlite3 ./app.db up         # Reapply it
```

### Commit

```bash
git add .
git commit -m "feat: add sqlite database and goose migrations"
```

### Exercises

1. Run `goose -dir migrations -s create add_task_priority sql`, write the `Up` and `Down` for a `priority` column, then run `up`, `down` and `up` again. Delete the migration file when you are done (after running `down`), since the rest of the roadmap doesn't expect it.
2. Remove `_pragma=foreign_keys(1)` from the connection string, and explain what you expect to happen to a user's tasks when that user is deleted. The store test in Phase 5 can confirm it.

---

## Phase 5: Domain models and the store layer

**Goal:** Go functions for all the SQL, and your first automated tests.

> **Go concepts**
> - **Methods:** `func (s *TaskStore) Get(...)` attaches a function to a type. `s` is the receiver, Go's version of `this`.
> - **Pointer receivers:** `*TaskStore` means the method works on the original value rather than a copy.
> - **Constructors:** Go has none built in, so the convention is a `NewX` function.
> - **Interfaces:** any type with the right methods satisfies an interface automatically. There's no `implements` keyword.
> - **Sentinel errors:** package-level `var ErrNotFound = errors.New(...)`, compared with `errors.Is`.
> - **`errors.As`:** checks whether an error is, or wraps, a specific type, and extracts it.
> - **Slices:** `[]T`, grown with `append`.
> - **Testing:** a `_test.go` file, functions named `TestXxx(t *testing.T)`, and table-driven tests with `t.Run`.

### Step 5.1: Domain types

The `domain` package imports nothing from the rest of your project, so that any package can use it without creating an import cycle. Go refuses to compile if two packages import each other.

**File: `internal/domain/errors.go`**
```go
// Package domain holds the core types shared across the application.
package domain

import "errors"

var (
	// ErrNotFound means the requested record does not exist.
	ErrNotFound = errors.New("not found")
	// ErrEmailTaken means an account with that email already exists.
	ErrEmailTaken = errors.New("email already registered")
)
```

**File: `internal/domain/user.go`**
```go
package domain

import "time"

// User is a registered account.
type User struct {
	ID    int64  `json:"id" example:"1"`
	Email string `json:"email" example:"ada@example.com"`
	// The "-" tag keeps the hash out of all JSON responses.
	PasswordHash string    `json:"-"`
	CreatedAt    time.Time `json:"created_at"`
}
```

The `json:"-"` tag is a safety feature: even if you accidentally return a whole `User`, the password hash can't be included.

**File: `internal/domain/task.go`**
```go
package domain

import "time"

// Task is a to-do item owned by a user.
type Task struct {
	ID          int64     `json:"id" example:"1"`
	UserID      int64     `json:"-"`
	Title       string    `json:"title" example:"Learn Go"`
	Description string    `json:"description" example:"Finish the roadmap"`
	Done        bool      `json:"done" example:"false"`
	CreatedAt   time.Time `json:"created_at"`
	UpdatedAt   time.Time `json:"updated_at"`
}
```

### Step 5.2: The user store

Two rules apply to every query from here on:

- **Always use `?` placeholders for values.** Never build SQL by joining user input into a string. That is how SQL injection happens.
- **Always pass `ctx`.** If the client disconnects, the query gets cancelled.

**File: `internal/store/user_store.go`**
```go
// Package store contains all SQL queries.
package store

import (
	"context"
	"database/sql"
	"errors"
	"fmt"

	"modernc.org/sqlite"
	sqlite3 "modernc.org/sqlite/lib"

	"github.com/yourname/taskapi/internal/domain"
)

// UserStore reads and writes users.
type UserStore struct {
	db *sql.DB
}

// NewUserStore returns a UserStore that uses db.
func NewUserStore(db *sql.DB) *UserStore {
	return &UserStore{db: db}
}

const userColumns = "id, email, password_hash, created_at"

// Create inserts a new user. It returns domain.ErrEmailTaken if the email exists.
func (s *UserStore) Create(ctx context.Context, email, passwordHash string) (domain.User, error) {
	row := s.db.QueryRowContext(ctx,
		"INSERT INTO users (email, password_hash) VALUES (?, ?) RETURNING "+userColumns,
		email, passwordHash,
	)
	user, err := scanUser(row)
	if err != nil {
		// Translate the driver's error into one the rest of the app understands.
		var sqliteErr *sqlite.Error
		if errors.As(err, &sqliteErr) && sqliteErr.Code() == sqlite3.SQLITE_CONSTRAINT_UNIQUE {
			return domain.User{}, domain.ErrEmailTaken
		}
		return domain.User{}, fmt.Errorf("insert user: %w", err)
	}
	return user, nil
}

// GetByEmail finds a user by email. It returns domain.ErrNotFound if none exists.
func (s *UserStore) GetByEmail(ctx context.Context, email string) (domain.User, error) {
	row := s.db.QueryRowContext(ctx, "SELECT "+userColumns+" FROM users WHERE email = ?", email)
	return getUser(row)
}

// GetByID finds a user by ID. It returns domain.ErrNotFound if none exists.
func (s *UserStore) GetByID(ctx context.Context, id int64) (domain.User, error) {
	row := s.db.QueryRowContext(ctx, "SELECT "+userColumns+" FROM users WHERE id = ?", id)
	return getUser(row)
}

func getUser(row *sql.Row) (domain.User, error) {
	user, err := scanUser(row)
	if errors.Is(err, sql.ErrNoRows) {
		return domain.User{}, domain.ErrNotFound
	}
	if err != nil {
		return domain.User{}, fmt.Errorf("get user: %w", err)
	}
	return user, nil
}

func scanUser(row *sql.Row) (domain.User, error) {
	var u domain.User
	err := row.Scan(&u.ID, &u.Email, &u.PasswordHash, &u.CreatedAt)
	return u, err
}
```

`Scan(&u.ID, ...)` takes pointers so it can write into your struct's fields. The columns must be listed in the same order as the arguments.

### Step 5.3: The task store

Every query includes `user_id = ?`. That is the authorisation rule enforced at the data layer: one user can never read or change another user's task, even by guessing its ID.

**File: `internal/store/task_store.go`**
```go
package store

import (
	"context"
	"database/sql"
	"errors"
	"fmt"

	"github.com/yourname/taskapi/internal/domain"
)

// TaskStore reads and writes tasks. Every method is scoped to one user.
type TaskStore struct {
	db *sql.DB
}

// NewTaskStore returns a TaskStore that uses db.
func NewTaskStore(db *sql.DB) *TaskStore {
	return &TaskStore{db: db}
}

const taskColumns = "id, user_id, title, description, done, created_at, updated_at"

// scanner is satisfied by both *sql.Row and *sql.Rows, so scanTask works with
// either one. This is an interface earning its keep.
type scanner interface {
	Scan(dest ...any) error
}

func scanTask(s scanner) (domain.Task, error) {
	var t domain.Task
	err := s.Scan(&t.ID, &t.UserID, &t.Title, &t.Description, &t.Done, &t.CreatedAt, &t.UpdatedAt)
	return t, err
}

// Create inserts a new task for the user.
func (s *TaskStore) Create(ctx context.Context, userID int64, title, description string) (domain.Task, error) {
	row := s.db.QueryRowContext(ctx,
		"INSERT INTO tasks (user_id, title, description) VALUES (?, ?, ?) RETURNING "+taskColumns,
		userID, title, description,
	)
	task, err := scanTask(row)
	if err != nil {
		return domain.Task{}, fmt.Errorf("insert task: %w", err)
	}
	return task, nil
}

// List returns all of the user's tasks, newest first.
func (s *TaskStore) List(ctx context.Context, userID int64) ([]domain.Task, error) {
	rows, err := s.db.QueryContext(ctx,
		"SELECT "+taskColumns+" FROM tasks WHERE user_id = ? ORDER BY created_at DESC, id DESC",
		userID,
	)
	if err != nil {
		return nil, fmt.Errorf("list tasks: %w", err)
	}
	return collectTasks(rows)
}

// collectTasks reads every row and closes rows when finished.
func collectTasks(rows *sql.Rows) ([]domain.Task, error) {
	defer rows.Close()

	// Start with an empty slice, not nil, so that JSON renders [] and not null.
	tasks := []domain.Task{}
	for rows.Next() {
		task, err := scanTask(rows)
		if err != nil {
			return nil, fmt.Errorf("scan task: %w", err)
		}
		tasks = append(tasks, task)
	}
	// rows.Next returns false on errors as well as at the end, so always check.
	if err := rows.Err(); err != nil {
		return nil, fmt.Errorf("iterate tasks: %w", err)
	}
	return tasks, nil
}

// Get returns one of the user's tasks, or domain.ErrNotFound.
func (s *TaskStore) Get(ctx context.Context, userID, id int64) (domain.Task, error) {
	row := s.db.QueryRowContext(ctx,
		"SELECT "+taskColumns+" FROM tasks WHERE id = ? AND user_id = ?",
		id, userID,
	)
	task, err := scanTask(row)
	if errors.Is(err, sql.ErrNoRows) {
		return domain.Task{}, domain.ErrNotFound
	}
	if err != nil {
		return domain.Task{}, fmt.Errorf("get task: %w", err)
	}
	return task, nil
}

// Update replaces a task's fields, or returns domain.ErrNotFound.
func (s *TaskStore) Update(ctx context.Context, userID, id int64, title, description string, done bool) (domain.Task, error) {
	row := s.db.QueryRowContext(ctx,
		`UPDATE tasks
		 SET title = ?, description = ?, done = ?, updated_at = CURRENT_TIMESTAMP
		 WHERE id = ? AND user_id = ?
		 RETURNING `+taskColumns,
		title, description, done, id, userID,
	)
	task, err := scanTask(row)
	if errors.Is(err, sql.ErrNoRows) {
		return domain.Task{}, domain.ErrNotFound
	}
	if err != nil {
		return domain.Task{}, fmt.Errorf("update task: %w", err)
	}
	return task, nil
}

// Delete removes a task, or returns domain.ErrNotFound.
func (s *TaskStore) Delete(ctx context.Context, userID, id int64) error {
	result, err := s.db.ExecContext(ctx, "DELETE FROM tasks WHERE id = ? AND user_id = ?", id, userID)
	if err != nil {
		return fmt.Errorf("delete task: %w", err)
	}
	affected, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("delete task: %w", err)
	}
	if affected == 0 {
		return domain.ErrNotFound
	}
	return nil
}
```

A quick reference for the three `database/sql` query methods:

| Method | Use when | Returns |
|---|---|---|
| `QueryRowContext` | You expect one row, including `RETURNING` | `*sql.Row` |
| `QueryContext` | You expect many rows | `*sql.Rows` (you must close it) |
| `ExecContext` | No rows come back | `sql.Result` |

### Step 5.4: Your first tests

These tests use a real, temporary SQLite database, because the whole point of a store is its SQL. `t.TempDir()` gives each test a folder that gets deleted automatically.

**File: `internal/store/task_store_test.go`**
```go
package store

import (
	"database/sql"
	"errors"
	"path/filepath"
	"testing"

	"github.com/yourname/taskapi/internal/database"
	"github.com/yourname/taskapi/internal/domain"
)

// newTestDB returns a migrated database that lasts only as long as the test.
// t.Helper makes failures point at the caller's line rather than this one.
func newTestDB(t *testing.T) *sql.DB {
	t.Helper()
	db, err := database.Open(filepath.Join(t.TempDir(), "test.db"))
	if err != nil {
		t.Fatalf("open database: %v", err)
	}
	t.Cleanup(func() { db.Close() })

	if err := database.Migrate(db); err != nil {
		t.Fatalf("migrate: %v", err)
	}
	return db
}

func newTestUser(t *testing.T, users *UserStore, email string) domain.User {
	t.Helper()
	user, err := users.Create(t.Context(), email, "hash")
	if err != nil {
		t.Fatalf("create user: %v", err)
	}
	return user
}

func TestUserStore_DuplicateEmail(t *testing.T) {
	users := NewUserStore(newTestDB(t))
	newTestUser(t, users, "ada@example.com")

	_, err := users.Create(t.Context(), "ada@example.com", "hash")
	if !errors.Is(err, domain.ErrEmailTaken) {
		t.Fatalf("got error %v, want %v", err, domain.ErrEmailTaken)
	}
}

func TestTaskStore_CRUD(t *testing.T) {
	db := newTestDB(t)
	tasks := NewTaskStore(db)
	user := newTestUser(t, NewUserStore(db), "ada@example.com")
	ctx := t.Context()

	created, err := tasks.Create(ctx, user.ID, "Learn Go", "Finish the roadmap")
	if err != nil {
		t.Fatalf("Create: %v", err)
	}
	if created.ID == 0 || created.Done || created.CreatedAt.IsZero() {
		t.Fatalf("unexpected created task: %+v", created)
	}

	updated, err := tasks.Update(ctx, user.ID, created.ID, "Learn Go well", "", true)
	if err != nil {
		t.Fatalf("Update: %v", err)
	}
	if updated.Title != "Learn Go well" || !updated.Done {
		t.Fatalf("unexpected updated task: %+v", updated)
	}

	list, err := tasks.List(ctx, user.ID)
	if err != nil {
		t.Fatalf("List: %v", err)
	}
	if len(list) != 1 {
		t.Fatalf("got %d tasks, want 1", len(list))
	}

	if err := tasks.Delete(ctx, user.ID, created.ID); err != nil {
		t.Fatalf("Delete: %v", err)
	}
	if _, err := tasks.Get(ctx, user.ID, created.ID); !errors.Is(err, domain.ErrNotFound) {
		t.Fatalf("Get after delete: got %v, want %v", err, domain.ErrNotFound)
	}
}

// A table-driven test: one loop runs many named cases. This is the most common
// test style in Go. Here it proves that another user's task is out of reach.
func TestTaskStore_OtherUsersTaskIsHidden(t *testing.T) {
	db := newTestDB(t)
	users := NewUserStore(db)
	tasks := NewTaskStore(db)
	owner := newTestUser(t, users, "owner@example.com")
	intruder := newTestUser(t, users, "intruder@example.com")

	task, err := tasks.Create(t.Context(), owner.ID, "Private", "")
	if err != nil {
		t.Fatalf("Create: %v", err)
	}

	cases := []struct {
		name string
		call func() error
	}{
		{"get", func() error {
			_, err := tasks.Get(t.Context(), intruder.ID, task.ID)
			return err
		}},
		{"update", func() error {
			_, err := tasks.Update(t.Context(), intruder.ID, task.ID, "Hacked", "", true)
			return err
		}},
		{"delete", func() error {
			return tasks.Delete(t.Context(), intruder.ID, task.ID)
		}},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			if err := tc.call(); !errors.Is(err, domain.ErrNotFound) {
				t.Fatalf("got %v, want %v", err, domain.ErrNotFound)
			}
		})
	}
}

func TestDeletingUserDeletesTheirTasks(t *testing.T) {
	db := newTestDB(t)
	tasks := NewTaskStore(db)
	user := newTestUser(t, NewUserStore(db), "ada@example.com")

	if _, err := tasks.Create(t.Context(), user.ID, "Learn Go", ""); err != nil {
		t.Fatalf("Create: %v", err)
	}
	if _, err := db.ExecContext(t.Context(), "DELETE FROM users WHERE id = ?", user.ID); err != nil {
		t.Fatalf("delete user: %v", err)
	}

	list, err := tasks.List(t.Context(), user.ID)
	if err != nil {
		t.Fatalf("List: %v", err)
	}
	if len(list) != 0 {
		t.Fatalf("got %d tasks, want 0: ON DELETE CASCADE did not run", len(list))
	}
}
```

### Checkpoint

```bash
go mod tidy
go test ./...
go test -v ./internal/store     # See each test by name
```

Everything should report `ok`. Now do exercise 2 from Phase 4: remove `_pragma=foreign_keys(1)`, re-run the tests, and watch `TestDeletingUserDeletesTheirTasks` fail. Put it back afterwards. A test that you have seen fail is a test you can trust.

### Commit

```bash
git add .
git commit -m "feat: add domain models and sqlite stores with tests"
```

### Exercises

1. Add `func (s *TaskStore) CountDone(ctx context.Context, userID int64) (int, error)` and a test for it.
2. Add a case-insensitive `Search(ctx, userID, query)` method using `WHERE title LIKE ?`, passing `"%" + query + "%"` as the argument. Why is this still safe from SQL injection?

---

## Phase 6: Authentication

**Goal:** users can register and log in, and protected routes reject anyone without a valid token.

> **Go concepts**
> - **Defining interfaces where they are used:** the handler declares the methods it needs from a store, keeping it decoupled from the concrete type. The Go proverb: "accept interfaces, return structs".
> - **Context values:** passing request-scoped data, such as the logged-in user's ID, through `r.Context()`.
> - **Unexported key types:** using a private type as the context key, so that no other package can collide with it.
> - **Type assertions:** `value, ok := x.(int64)` safely gets a concrete type out of an `any`.
> - **Dependency injection:** passing dependencies into constructors, instead of using global variables.

### How the authentication works

1. **Register:** the server hashes the password with **bcrypt** and stores only the hash. bcrypt is deliberately slow and salts each hash automatically, so a stolen database is expensive to crack. Never store plain passwords, and never use a fast hash such as MD5 or SHA-256 for them.
2. **Login:** the server checks the password against the hash, then issues a **JWT**.
3. **Later requests:** the client sends `Authorization: Bearer <token>`. The server verifies the token's signature and expiry, without needing a database lookup.

A JWT has three dot-separated, base64-encoded parts: `header.payload.signature`. The payload is **readable by anyone** and only protected against tampering, so never put secrets in it. Paste a token into [jwt.io](https://jwt.io) later to see for yourself (fine for a local test token, but never paste a production one into a website).

### Step 6.1: Install the dependencies

```bash
go get golang.org/x/crypto/bcrypt
go get github.com/golang-jwt/jwt/v5
```

### Step 6.2: Password hashing

**File: `internal/auth/password.go`**
```go
// Package auth handles password hashing, tokens and the authenticated user.
package auth

import (
	"fmt"

	"golang.org/x/crypto/bcrypt"
)

// HashPassword returns a bcrypt hash of password. bcrypt rejects passwords
// longer than 72 bytes, so handlers must validate the length first.
func HashPassword(password string) (string, error) {
	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return "", fmt.Errorf("hash password: %w", err)
	}
	return string(hash), nil
}

// CheckPassword reports whether password matches hash.
func CheckPassword(hash, password string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

### Step 6.3: JWT tokens

**File: `internal/auth/token.go`**
```go
package auth

import (
	"errors"
	"fmt"
	"strconv"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

const tokenIssuer = "taskapi"

// ErrInvalidToken means a token is malformed, expired or wrongly signed.
var ErrInvalidToken = errors.New("invalid token")

// TokenManager issues and verifies signed JWTs.
type TokenManager struct {
	secret []byte
	ttl    time.Duration
}

// NewTokenManager returns a TokenManager whose tokens last for ttl.
func NewTokenManager(secret string, ttl time.Duration) *TokenManager {
	return &TokenManager{secret: []byte(secret), ttl: ttl}
}

// Issue creates a signed token for the given user.
func (m *TokenManager) Issue(userID int64) (string, error) {
	now := time.Now()
	claims := jwt.RegisteredClaims{
		Subject:   strconv.FormatInt(userID, 10),
		Issuer:    tokenIssuer,
		IssuedAt:  jwt.NewNumericDate(now),
		ExpiresAt: jwt.NewNumericDate(now.Add(m.ttl)),
	}

	signed, err := jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString(m.secret)
	if err != nil {
		return "", fmt.Errorf("sign token: %w", err)
	}
	return signed, nil
}

// Verify checks a token and returns the user ID inside it.
func (m *TokenManager) Verify(tokenString string) (int64, error) {
	var claims jwt.RegisteredClaims
	_, err := jwt.ParseWithClaims(tokenString, &claims,
		func(*jwt.Token) (any, error) { return m.secret, nil },
		// Pinning the algorithm blocks attacks that swap in a different one.
		jwt.WithValidMethods([]string{jwt.SigningMethodHS256.Alg()}),
		jwt.WithIssuer(tokenIssuer),
		jwt.WithExpirationRequired(),
	)
	if err != nil {
		return 0, fmt.Errorf("%w: %v", ErrInvalidToken, err)
	}

	userID, err := strconv.ParseInt(claims.Subject, 10, 64)
	if err != nil {
		return 0, fmt.Errorf("%w: bad subject", ErrInvalidToken)
	}
	return userID, nil
}
```

### Step 6.4: The user ID in the request context

Middleware runs before the handler, and it needs some way to tell the handler who the user is. The request's context carries that value.

**File: `internal/auth/context.go`**
```go
package auth

import "context"

// contextKey is private, so no other package can read or overwrite our value
// by accident, even if it also uses the string "userID".
type contextKey struct{}

// WithUserID returns a copy of ctx that carries the user ID.
func WithUserID(ctx context.Context, userID int64) context.Context {
	return context.WithValue(ctx, contextKey{}, userID)
}

// UserID returns the user ID stored in ctx, if there is one.
func UserID(ctx context.Context) (int64, bool) {
	id, ok := ctx.Value(contextKey{}).(int64)
	return id, ok
}
```

### Step 6.5: The authentication middleware

`RequireAuth` is a function that **returns** a middleware. It's the closure pattern from Phase 3 with one more layer, so the middleware can capture the `TokenManager`.

**File: `internal/middleware/auth.go`**
```go
package middleware

import (
	"net/http"
	"strings"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/httpx"
)

// RequireAuth rejects requests without a valid "Authorization: Bearer" token.
func RequireAuth(tokens *auth.TokenManager) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			token, found := strings.CutPrefix(r.Header.Get("Authorization"), "Bearer ")
			if !found {
				httpx.Error(w, http.StatusUnauthorized, "missing bearer token")
				return
			}

			userID, err := tokens.Verify(token)
			if err != nil {
				httpx.Error(w, http.StatusUnauthorized, "invalid or expired token")
				return
			}

			// Requests are immutable by convention, so pass on a copy that
			// carries the new context.
			next.ServeHTTP(w, r.WithContext(auth.WithUserID(r.Context(), userID)))
		})
	}
}
```

### Step 6.6: The auth handlers

The handlers live in a package named `v1`, because Phase 8 adds a second API version beside it.

The `// @...` comments are Swagger annotations. They do nothing until Phase 9, but writing docs alongside the code is much easier than adding them all at the end, so type them now.

**File: `internal/handler/v1/auth.go`**
```go
// Package v1 implements version 1 of the HTTP API.
package v1

import (
	"context"
	"errors"
	"net/http"
	"net/mail"
	"strings"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/domain"
	"github.com/yourname/taskapi/internal/httpx"
)

const (
	minPasswordLen = 8
	maxPasswordLen = 72 // bcrypt's limit.
)

// UserStore is the set of user operations these handlers need. It's defined
// here, where it's used, so tests can pass in a fake.
type UserStore interface {
	Create(ctx context.Context, email, passwordHash string) (domain.User, error)
	GetByEmail(ctx context.Context, email string) (domain.User, error)
	GetByID(ctx context.Context, id int64) (domain.User, error)
}

// AuthHandler serves the registration and login endpoints.
type AuthHandler struct {
	users  UserStore
	tokens *auth.TokenManager
}

// NewAuthHandler returns an AuthHandler.
func NewAuthHandler(users UserStore, tokens *auth.TokenManager) *AuthHandler {
	return &AuthHandler{users: users, tokens: tokens}
}

// CredentialsRequest is the body for register and login.
type CredentialsRequest struct {
	Email    string `json:"email" example:"ada@example.com"`
	Password string `json:"password" example:"correct-horse-battery"`
}

// TokenResponse is returned after a successful login.
type TokenResponse struct {
	Token string `json:"token" example:"eyJhbGciOiJIUzI1NiIs..."`
}

// Register creates a new user account.
//
//	@Summary	Register a new user
//	@Tags		auth
//	@Accept		json
//	@Produce	json
//	@Param		request	body		CredentialsRequest	true	"Email and password (8-72 characters)"
//	@Success	201		{object}	domain.User
//	@Failure	400		{object}	httpx.ErrorResponse
//	@Failure	409		{object}	httpx.ErrorResponse
//	@Router		/v1/auth/register [post]
func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
	req, err := httpx.DecodeJSON[CredentialsRequest](w, r)
	if err != nil {
		httpx.Error(w, http.StatusBadRequest, err.Error())
		return
	}

	email := normalizeEmail(req.Email)
	if _, err := mail.ParseAddress(email); err != nil {
		httpx.Error(w, http.StatusBadRequest, "invalid email address")
		return
	}
	if len(req.Password) < minPasswordLen || len(req.Password) > maxPasswordLen {
		httpx.Error(w, http.StatusBadRequest, "password must be 8-72 characters")
		return
	}

	hash, err := auth.HashPassword(req.Password)
	if err != nil {
		httpx.InternalError(w, r, err)
		return
	}

	user, err := h.users.Create(r.Context(), email, hash)
	if errors.Is(err, domain.ErrEmailTaken) {
		httpx.Error(w, http.StatusConflict, err.Error())
		return
	}
	if err != nil {
		httpx.InternalError(w, r, err)
		return
	}

	httpx.JSON(w, http.StatusCreated, user)
}

// Login verifies credentials and returns a JWT.
//
//	@Summary	Log in and get a JWT
//	@Tags		auth
//	@Accept		json
//	@Produce	json
//	@Param		request	body		CredentialsRequest	true	"Email and password"
//	@Success	200		{object}	TokenResponse
//	@Failure	400		{object}	httpx.ErrorResponse
//	@Failure	401		{object}	httpx.ErrorResponse
//	@Router		/v1/auth/login [post]
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
	req, err := httpx.DecodeJSON[CredentialsRequest](w, r)
	if err != nil {
		httpx.Error(w, http.StatusBadRequest, err.Error())
		return
	}

	user, err := h.users.GetByEmail(r.Context(), normalizeEmail(req.Email))
	if err != nil && !errors.Is(err, domain.ErrNotFound) {
		httpx.InternalError(w, r, err)
		return
	}

	// Use the same message for a wrong email and a wrong password, so that
	// attackers can't use this endpoint to find out who has an account.
	if err != nil || !auth.CheckPassword(user.PasswordHash, req.Password) {
		httpx.Error(w, http.StatusUnauthorized, "invalid email or password")
		return
	}

	token, err := h.tokens.Issue(user.ID)
	if err != nil {
		httpx.InternalError(w, r, err)
		return
	}
	httpx.JSON(w, http.StatusOK, TokenResponse{Token: token})
}

// Me returns the currently authenticated user.
//
//	@Summary	Get the current user
//	@Tags		auth
//	@Produce	json
//	@Security	BearerAuth
//	@Success	200	{object}	domain.User
//	@Failure	401	{object}	httpx.ErrorResponse
//	@Router		/v1/auth/me [get]
func (h *AuthHandler) Me(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.UserID(r.Context())
	if !ok {
		httpx.Error(w, http.StatusUnauthorized, "not authenticated")
		return
	}

	user, err := h.users.GetByID(r.Context(), userID)
	if errors.Is(err, domain.ErrNotFound) {
		// The token is valid, but its account has since been deleted.
		httpx.Error(w, http.StatusUnauthorized, "not authenticated")
		return
	}
	if err != nil {
		httpx.InternalError(w, r, err)
		return
	}
	httpx.JSON(w, http.StatusOK, user)
}

func normalizeEmail(email string) string {
	return strings.ToLower(strings.TrimSpace(email))
}
```

Note the pattern: `if err != nil { ...; return }`. The `return` matters, because writing an error response doesn't stop the function. Forgetting it is one of the most common bugs in Go handlers.

### Step 6.7: Routes for version 1

**File: `internal/handler/v1/routes.go`**
```go
package v1

import (
	"net/http"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/middleware"
)

// Routes returns the handler for all v1 endpoints. Paths are relative to the
// prefix the caller mounts it on.
func Routes(users UserStore, tokens *auth.TokenManager) http.Handler {
	mux := http.NewServeMux()
	requireAuth := middleware.RequireAuth(tokens)

	authHandler := NewAuthHandler(users, tokens)
	mux.HandleFunc("POST /auth/register", authHandler.Register)
	mux.HandleFunc("POST /auth/login", authHandler.Login)
	mux.Handle("GET /auth/me", requireAuth(http.HandlerFunc(authHandler.Me)))

	return mux
}
```

`authHandler.Register` is a **method value**: a function already bound to `authHandler`. That is how a method with access to the handler's stores can be passed where a plain handler function is expected.

### Step 6.8: Mount it in the router

`http.StripPrefix` removes `/api/v1` from the path before the inner mux sees it, so `/api/v1/auth/login` matches the pattern `POST /auth/login`.

**File: `internal/server/router.go`**
```go
// Package server assembles the application's routes and middleware.
package server

import (
	"net/http"

	"github.com/yourname/taskapi/internal/auth"
	v1 "github.com/yourname/taskapi/internal/handler/v1"
	"github.com/yourname/taskapi/internal/httpx"
	"github.com/yourname/taskapi/internal/middleware"
	"github.com/yourname/taskapi/internal/store"
)

// Deps lists everything the router needs from the outside world.
type Deps struct {
	Users  *store.UserStore
	Tasks  *store.TaskStore
	Tokens *auth.TokenManager
}

// NewRouter builds the root HTTP handler for the application.
func NewRouter(deps Deps) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", healthz)

	mux.Handle("/api/v1/", http.StripPrefix("/api/v1", v1.Routes(deps.Users, deps.Tokens)))

	return middleware.Chain(mux, middleware.Logger, middleware.Recover)
}

func healthz(w http.ResponseWriter, r *http.Request) {
	httpx.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}
```

### Step 6.9: Wire it all together in `main`

`main` is the only place that knows about every concrete type. It builds each part, and passes it to whatever needs it.

**File: `cmd/api/main.go`**
```go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/config"
	"github.com/yourname/taskapi/internal/database"
	"github.com/yourname/taskapi/internal/server"
	"github.com/yourname/taskapi/internal/store"
)

// version is replaced at build time (see Phase 11).
var version = "dev"

func main() {
	if err := run(); err != nil {
		slog.Error("fatal", "err", err)
		os.Exit(1)
	}
}

func run() error {
	cfg, err := config.Load()
	if err != nil {
		return err
	}

	db, err := database.Open(cfg.DatabasePath)
	if err != nil {
		return err
	}
	defer db.Close()

	if err := database.Migrate(db); err != nil {
		return err
	}

	router := server.NewRouter(server.Deps{
		Users:  store.NewUserStore(db),
		Tasks:  store.NewTaskStore(db),
		Tokens: auth.NewTokenManager(cfg.JWTSecret, cfg.JWTTTL),
	})

	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           router,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	// ctx is cancelled when the process receives Ctrl+C or SIGTERM.
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	// ListenAndServe blocks, so it runs in its own goroutine. The channel is
	// buffered so that the goroutine can always send and then exit.
	serverErr := make(chan error, 1)
	go func() {
		slog.Info("server starting", "addr", srv.Addr, "version", version)
		if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
			serverErr <- err
		}
	}()

	// Wait for whichever happens first: a server failure or a stop signal.
	select {
	case err := <-serverErr:
		return err
	case <-ctx.Done():
		slog.Info("shutting down")
	}

	// Give in-flight requests up to 10 seconds to finish.
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	return srv.Shutdown(shutdownCtx)
}
```

### Checkpoint

```bash
go mod tidy
air
```

```bash
# Register: 201, with no password hash in the response
curl -i -X POST localhost:8080/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"ada@example.com","password":"correct-horse-battery"}'

# Register again: 409 Conflict
curl -i -X POST localhost:8080/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"ada@example.com","password":"correct-horse-battery"}'

# Wrong password: 401
curl -i -X POST localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"ada@example.com","password":"wrong-password"}'

# Log in and save the token in a shell variable
TOKEN=$(curl -s -X POST localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"ada@example.com","password":"correct-horse-battery"}' \
  | sed -E 's/.*"token":"([^"]+)".*/\1/')
echo "$TOKEN"

# No token: 401
curl -i localhost:8080/api/v1/auth/me

# With the token: 200
curl -i localhost:8080/api/v1/auth/me -H "Authorization: Bearer $TOKEN"

# The stored password is a bcrypt hash, starting with $2a$
sqlite3 app.db "SELECT email, password_hash FROM users;"
```

### Commit

```bash
git add .
git commit -m "feat: add registration, login and JWT auth middleware"
```

### Exercises

1. `Login` returns faster for an unknown email than for a wrong password, because it skips bcrypt, and an attacker can measure that difference. Fix it by comparing against a fixed dummy hash when the user isn't found.
2. Add a password rule that rejects passwords equal to the email address.
3. Set `JWT_TTL=10s`, log in, wait 11 seconds and confirm that `/auth/me` returns 401.

---

## Phase 7: Task CRUD REST API

**Goal:** the complete REST resource, protected by auth.

> **Go concepts**
> - **Path parameters:** `r.PathValue("id")` reads `{id}` from a route pattern.
> - **`strconv`:** converting strings to numbers.
> - **`switch` with no condition:** a neat replacement for long `if/else if` chains.
> - **Functions returning `(value, bool)`:** the "comma ok" style, for when failure has only one meaning.

### REST conventions used here

| Action | Method and path | Success status | Body |
|---|---|---|---|
| Create | `POST /tasks` | 201 Created | The new task |
| List | `GET /tasks` | 200 OK | Array of tasks |
| Read | `GET /tasks/{id}` | 200 OK | The task |
| Replace | `PUT /tasks/{id}` | 200 OK | The updated task |
| Delete | `DELETE /tasks/{id}` | 204 No Content | Empty |

The errors used are: 400 for bad input, 401 for not authenticated, 404 for not found, 409 for a conflict and 500 for server failures.

A request for another user's task returns **404 rather than 403**. A 403 would confirm that the task exists.

### Step 7.1: The task handlers

**File: `internal/handler/v1/task.go`**
```go
package v1

import (
	"context"
	"errors"
	"net/http"
	"strconv"
	"strings"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/domain"
	"github.com/yourname/taskapi/internal/httpx"
)

const maxTitleLen = 200

// TaskStore is the set of task operations these handlers need.
type TaskStore interface {
	Create(ctx context.Context, userID int64, title, description string) (domain.Task, error)
	List(ctx context.Context, userID int64) ([]domain.Task, error)
	Get(ctx context.Context, userID, id int64) (domain.Task, error)
	Update(ctx context.Context, userID, id int64, title, description string, done bool) (domain.Task, error)
	Delete(ctx context.Context, userID, id int64) error
}

// TaskHandler serves the task endpoints.
type TaskHandler struct {
	tasks TaskStore
}

// NewTaskHandler returns a TaskHandler.
func NewTaskHandler(tasks TaskStore) *TaskHandler {
	return &TaskHandler{tasks: tasks}
}

// CreateTaskRequest is the body for creating a task.
type CreateTaskRequest struct {
	Title       string `json:"title" example:"Learn Go"`
	Description string `json:"description" example:"Finish the roadmap"`
}

// UpdateTaskRequest is the body for replacing a task.
type UpdateTaskRequest struct {
	Title       string `json:"title" example:"Learn Go"`
	Description string `json:"description" example:"Finish the roadmap"`
	Done        bool   `json:"done" example:"true"`
}

// Create adds a task for the current user.
//
//	@Summary	Create a task
//	@Tags		tasks
//	@Accept		json
//	@Produce	json
//	@Security	BearerAuth
//	@Param		request	body		CreateTaskRequest	true	"The new task"
//	@Success	201		{object}	domain.Task
//	@Failure	400		{object}	httpx.ErrorResponse
//	@Failure	401		{object}	httpx.ErrorResponse
//	@Router		/v1/tasks [post]
//	@Router		/v2/tasks [post]
func (h *TaskHandler) Create(w http.ResponseWriter, r *http.Request) {
	userID, ok := currentUser(w, r)
	if !ok {
		return
	}

	req, err := httpx.DecodeJSON[CreateTaskRequest](w, r)
	if err != nil {
		httpx.Error(w, http.StatusBadRequest, err.Error())
		return
	}

	title := strings.TrimSpace(req.Title)
	if message := validateTitle(title); message != "" {
		httpx.Error(w, http.StatusBadRequest, message)
		return
	}

	task, err := h.tasks.Create(r.Context(), userID, title, req.Description)
	if err != nil {
		httpx.InternalError(w, r, err)
		return
	}
	httpx.JSON(w, http.StatusCreated, task)
}

// List returns all of the current user's tasks.
//
//	@Summary	List all tasks
//	@Tags		tasks
//	@Produce	json
//	@Security	BearerAuth
//	@Success	200	{array}		domain.Task
//	@Failure	401	{object}	httpx.ErrorResponse
//	@Router		/v1/tasks [get]
func (h *TaskHandler) List(w http.ResponseWriter, r *http.Request) {
	userID, ok := currentUser(w, r)
	if !ok {
		return
	}

	tasks, err := h.tasks.List(r.Context(), userID)
	if err != nil {
		httpx.InternalError(w, r, err)
		return
	}
	httpx.JSON(w, http.StatusOK, tasks)
}

// Get returns one task.
//
//	@Summary	Get a task
//	@Tags		tasks
//	@Produce	json
//	@Security	BearerAuth
//	@Param		id	path		int	true	"Task ID"
//	@Success	200	{object}	domain.Task
//	@Failure	400	{object}	httpx.ErrorResponse
//	@Failure	401	{object}	httpx.ErrorResponse
//	@Failure	404	{object}	httpx.ErrorResponse
//	@Router		/v1/tasks/{id} [get]
//	@Router		/v2/tasks/{id} [get]
func (h *TaskHandler) Get(w http.ResponseWriter, r *http.Request) {
	userID, ok := currentUser(w, r)
	if !ok {
		return
	}
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	task, err := h.tasks.Get(r.Context(), userID, id)
	if err != nil {
		writeStoreError(w, r, err)
		return
	}
	httpx.JSON(w, http.StatusOK, task)
}

// Update replaces a task's fields.
//
//	@Summary	Replace a task
//	@Tags		tasks
//	@Accept		json
//	@Produce	json
//	@Security	BearerAuth
//	@Param		id		path		int					true	"Task ID"
//	@Param		request	body		UpdateTaskRequest	true	"The new values"
//	@Success	200		{object}	domain.Task
//	@Failure	400		{object}	httpx.ErrorResponse
//	@Failure	401		{object}	httpx.ErrorResponse
//	@Failure	404		{object}	httpx.ErrorResponse
//	@Router		/v1/tasks/{id} [put]
//	@Router		/v2/tasks/{id} [put]
func (h *TaskHandler) Update(w http.ResponseWriter, r *http.Request) {
	userID, ok := currentUser(w, r)
	if !ok {
		return
	}
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	req, err := httpx.DecodeJSON[UpdateTaskRequest](w, r)
	if err != nil {
		httpx.Error(w, http.StatusBadRequest, err.Error())
		return
	}

	title := strings.TrimSpace(req.Title)
	if message := validateTitle(title); message != "" {
		httpx.Error(w, http.StatusBadRequest, message)
		return
	}

	task, err := h.tasks.Update(r.Context(), userID, id, title, req.Description, req.Done)
	if err != nil {
		writeStoreError(w, r, err)
		return
	}
	httpx.JSON(w, http.StatusOK, task)
}

// Delete removes a task.
//
//	@Summary	Delete a task
//	@Tags		tasks
//	@Security	BearerAuth
//	@Param		id	path	int	true	"Task ID"
//	@Success	204
//	@Failure	400	{object}	httpx.ErrorResponse
//	@Failure	401	{object}	httpx.ErrorResponse
//	@Failure	404	{object}	httpx.ErrorResponse
//	@Router		/v1/tasks/{id} [delete]
//	@Router		/v2/tasks/{id} [delete]
func (h *TaskHandler) Delete(w http.ResponseWriter, r *http.Request) {
	userID, ok := currentUser(w, r)
	if !ok {
		return
	}
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	if err := h.tasks.Delete(r.Context(), userID, id); err != nil {
		writeStoreError(w, r, err)
		return
	}
	w.WriteHeader(http.StatusNoContent)
}

// currentUser reads the user ID that RequireAuth stored in the context. If it
// is missing, it writes a 401 and returns false.
func currentUser(w http.ResponseWriter, r *http.Request) (int64, bool) {
	userID, ok := auth.UserID(r.Context())
	if !ok {
		httpx.Error(w, http.StatusUnauthorized, "not authenticated")
	}
	return userID, ok
}

// parseID reads the {id} path parameter. If it is invalid, it writes a 400 and
// returns false.
func parseID(w http.ResponseWriter, r *http.Request) (int64, bool) {
	id, err := strconv.ParseInt(r.PathValue("id"), 10, 64)
	if err != nil || id <= 0 {
		httpx.Error(w, http.StatusBadRequest, "id must be a positive integer")
		return 0, false
	}
	return id, true
}

// validateTitle returns an error message, or "" if the title is valid.
func validateTitle(title string) string {
	switch {
	case title == "":
		return "title is required"
	case len(title) > maxTitleLen:
		return "title must be at most 200 characters"
	default:
		return ""
	}
}

// writeStoreError maps store errors to HTTP responses.
func writeStoreError(w http.ResponseWriter, r *http.Request, err error) {
	if errors.Is(err, domain.ErrNotFound) {
		httpx.Error(w, http.StatusNotFound, "task not found")
		return
	}
	httpx.InternalError(w, r, err)
}
```

### Step 7.2: Add the routes

**File: `internal/handler/v1/routes.go`**
```go
package v1

import (
	"net/http"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/middleware"
)

// Routes returns the handler for all v1 endpoints. Paths are relative to the
// prefix the caller mounts it on.
func Routes(users UserStore, tasks TaskStore, tokens *auth.TokenManager) http.Handler {
	mux := http.NewServeMux()
	requireAuth := middleware.RequireAuth(tokens)

	authHandler := NewAuthHandler(users, tokens)
	mux.HandleFunc("POST /auth/register", authHandler.Register)
	mux.HandleFunc("POST /auth/login", authHandler.Login)
	mux.Handle("GET /auth/me", requireAuth(http.HandlerFunc(authHandler.Me)))

	taskHandler := NewTaskHandler(tasks)
	mux.Handle("POST /tasks", requireAuth(http.HandlerFunc(taskHandler.Create)))
	mux.Handle("GET /tasks", requireAuth(http.HandlerFunc(taskHandler.List)))
	mux.Handle("GET /tasks/{id}", requireAuth(http.HandlerFunc(taskHandler.Get)))
	mux.Handle("PUT /tasks/{id}", requireAuth(http.HandlerFunc(taskHandler.Update)))
	mux.Handle("DELETE /tasks/{id}", requireAuth(http.HandlerFunc(taskHandler.Delete)))

	return mux
}
```

### Step 7.3: Update the router

Only the `v1.Routes` line changes.

**File: `internal/server/router.go`**
```go
// Package server assembles the application's routes and middleware.
package server

import (
	"net/http"

	"github.com/yourname/taskapi/internal/auth"
	v1 "github.com/yourname/taskapi/internal/handler/v1"
	"github.com/yourname/taskapi/internal/httpx"
	"github.com/yourname/taskapi/internal/middleware"
	"github.com/yourname/taskapi/internal/store"
)

// Deps lists everything the router needs from the outside world.
type Deps struct {
	Users  *store.UserStore
	Tasks  *store.TaskStore
	Tokens *auth.TokenManager
}

// NewRouter builds the root HTTP handler for the application.
func NewRouter(deps Deps) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", healthz)

	mux.Handle("/api/v1/", http.StripPrefix("/api/v1", v1.Routes(deps.Users, deps.Tasks, deps.Tokens)))

	return middleware.Chain(mux, middleware.Logger, middleware.Recover)
}

func healthz(w http.ResponseWriter, r *http.Request) {
	httpx.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}
```

### Checkpoint

With Air running, and `TOKEN` still set from Phase 6 (log in again if not):

```bash
# Create: 201
curl -i -X POST localhost:8080/api/v1/tasks \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"title":"Learn Go","description":"Finish the roadmap"}'

# List: 200
curl -s localhost:8080/api/v1/tasks -H "Authorization: Bearer $TOKEN"

# Read one: 200
curl -i localhost:8080/api/v1/tasks/1 -H "Authorization: Bearer $TOKEN"

# Replace: 200, with done now true
curl -i -X PUT localhost:8080/api/v1/tasks/1 \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"title":"Learn Go","description":"Done!","done":true}'

# Empty title: 400
curl -i -X POST localhost:8080/api/v1/tasks \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"title":"   "}'

# Bad ID: 400
curl -i localhost:8080/api/v1/tasks/abc -H "Authorization: Bearer $TOKEN"

# Delete: 204, then 404 when repeated
curl -i -X DELETE localhost:8080/api/v1/tasks/1 -H "Authorization: Bearer $TOKEN"
curl -i -X DELETE localhost:8080/api/v1/tasks/1 -H "Authorization: Bearer $TOKEN"
```

Now register a second user, log in as them and try to read the first user's task. You should get a 404.

### Commit

```bash
git add .
git commit -m "feat: add task CRUD endpoints"
```

### Exercises

1. Add `PATCH /tasks/{id}` for partial updates. Use pointer fields in the request (`Title *string`), so that you can tell "not sent" (`nil`) apart from "sent as empty".
2. `len(title)` counts bytes, not characters. Try a 150-character title in Hindi or emoji, then fix the check with `utf8.RuneCountInString`.
3. Add a `DELETE /auth/me` endpoint that deletes the account, and check that the user's tasks disappear with it.

---

## Phase 8: API versioning

**Goal:** change your API's response format without breaking any existing client.

> **Go concepts**
> - **Interface embedding:** an interface can include another interface's methods.
> - **Pointers to express "optional":** `*bool` can be `nil`, `true` or `false`.
> - **`[]any` and variadic calls:** `args...` expands a slice into separate arguments.
> - **Splitting a type's methods across files** within a package.

### Why version an API?

Once a client depends on your API, changing a response's shape breaks that client. Versioning lets old clients keep working while new ones get the improvements.

| Approach | Example | Trade-off |
|---|---|---|
| **URL path (used here)** | `/api/v2/tasks` | Obvious, easy to test and route, and the most common |
| Header | `Accept: application/vnd.taskapi.v2+json` | Keeps URLs clean, but is harder to test and discover |
| Query string | `/tasks?version=2` | Easy, but easily forgotten and awkward with caching |

**When do you need a new version?** Only for **breaking** changes: removing or renaming a field, changing its type, or changing a response's structure. Adding a new optional field or a new endpoint isn't breaking, so it doesn't need one.

**Our breaking change:** v1's `GET /tasks` returns every task as a plain array, which won't scale. v2 returns one page at a time, wrapped in an object:

```json
{
  "data": [ { "id": 1, "title": "Learn Go" } ],
  "meta": { "page": 1, "limit": 20, "total": 57, "total_pages": 3 }
}
```

v2 changes only the list endpoint. The endpoints that haven't changed simply reuse v1's handlers, so nothing gets duplicated.

### Step 8.1: Add a new migration

Pagination sorts by `created_at` for one user, so an index will keep that fast as the table grows. This is what a normal schema change looks like: a new migration file, never an edit to an old one.

```bash
goose -dir migrations -s create add_tasks_user_created_index sql
```

**File: `migrations/00003_add_tasks_user_created_index.sql`**
```sql
-- +goose Up
CREATE INDEX idx_tasks_user_created ON tasks (user_id, created_at DESC, id DESC);

-- +goose Down
DROP INDEX idx_tasks_user_created;
```

The server applies it on the next start, because `main` calls `database.Migrate`.

### Step 8.2: Add the filter type

**File: `internal/domain/task.go`**
```go
package domain

import "time"

// Task is a to-do item owned by a user.
type Task struct {
	ID          int64     `json:"id" example:"1"`
	UserID      int64     `json:"-"`
	Title       string    `json:"title" example:"Learn Go"`
	Description string    `json:"description" example:"Finish the roadmap"`
	Done        bool      `json:"done" example:"false"`
	CreatedAt   time.Time `json:"created_at"`
	UpdatedAt   time.Time `json:"updated_at"`
}

// TaskFilter selects one page of a user's tasks.
type TaskFilter struct {
	Limit  int
	Offset int
	Done   *bool // nil means "don't filter on done".
}
```

### Step 8.3: The paginated query

A type's methods don't all have to be in one file. This new file adds a method to the existing `TaskStore`.

**File: `internal/store/task_page.go`**
```go
package store

import (
	"context"
	"fmt"

	"github.com/yourname/taskapi/internal/domain"
)

// ListPage returns one page of the user's tasks, plus the total number of tasks
// that match the filter across all pages.
func (s *TaskStore) ListPage(ctx context.Context, userID int64, filter domain.TaskFilter) ([]domain.Task, int, error) {
	// Only fixed strings are ever added to the SQL. Values stay in args, so
	// this is still safe from injection.
	where := "user_id = ?"
	args := []any{userID}
	if filter.Done != nil {
		where += " AND done = ?"
		args = append(args, *filter.Done)
	}

	var total int
	err := s.db.QueryRowContext(ctx, "SELECT COUNT(*) FROM tasks WHERE "+where, args...).Scan(&total)
	if err != nil {
		return nil, 0, fmt.Errorf("count tasks: %w", err)
	}

	args = append(args, filter.Limit, filter.Offset)
	rows, err := s.db.QueryContext(ctx,
		"SELECT "+taskColumns+" FROM tasks WHERE "+where+" ORDER BY created_at DESC, id DESC LIMIT ? OFFSET ?",
		args...,
	)
	if err != nil {
		return nil, 0, fmt.Errorf("list task page: %w", err)
	}

	tasks, err := collectTasks(rows)
	if err != nil {
		return nil, 0, err
	}
	return tasks, total, nil
}
```

### Step 8.4: The v2 handler

**File: `internal/handler/v2/task.go`**
```go
// Package v2 implements version 2 of the HTTP API.
package v2

import (
	"context"
	"fmt"
	"net/http"
	"strconv"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/domain"
	v1 "github.com/yourname/taskapi/internal/handler/v1"
	"github.com/yourname/taskapi/internal/httpx"
)

const (
	defaultLimit = 20
	maxLimit     = 100
)

// TaskStore includes everything v1 needs, plus pagination.
type TaskStore interface {
	v1.TaskStore
	ListPage(ctx context.Context, userID int64, filter domain.TaskFilter) ([]domain.Task, int, error)
}

// TaskHandler serves the v2 task endpoints.
type TaskHandler struct {
	tasks TaskStore
}

// NewTaskHandler returns a TaskHandler.
func NewTaskHandler(tasks TaskStore) *TaskHandler {
	return &TaskHandler{tasks: tasks}
}

// PageMeta describes where a page sits within the full result set.
type PageMeta struct {
	Page       int `json:"page" example:"1"`
	Limit      int `json:"limit" example:"20"`
	Total      int `json:"total" example:"57"`
	TotalPages int `json:"total_pages" example:"3"`
}

// TaskListResponse is the paginated response for listing tasks.
type TaskListResponse struct {
	Data []domain.Task `json:"data"`
	Meta PageMeta      `json:"meta"`
}

// List returns one page of the current user's tasks.
//
//	@Summary	List tasks with pagination and filtering
//	@Tags		tasks
//	@Produce	json
//	@Security	BearerAuth
//	@Param		page	query		int		false	"Page number"		default(1)		minimum(1)
//	@Param		limit	query		int		false	"Items per page"	default(20)		minimum(1)	maximum(100)
//	@Param		done	query		bool	false	"Filter by completion"
//	@Success	200		{object}	TaskListResponse
//	@Failure	400		{object}	httpx.ErrorResponse
//	@Failure	401		{object}	httpx.ErrorResponse
//	@Router		/v2/tasks [get]
func (h *TaskHandler) List(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.UserID(r.Context())
	if !ok {
		httpx.Error(w, http.StatusUnauthorized, "not authenticated")
		return
	}

	page, err := positiveIntParam(r, "page", 1)
	if err != nil {
		httpx.Error(w, http.StatusBadRequest, err.Error())
		return
	}
	limit, err := positiveIntParam(r, "limit", defaultLimit)
	if err != nil {
		httpx.Error(w, http.StatusBadRequest, err.Error())
		return
	}
	if limit > maxLimit {
		httpx.Error(w, http.StatusBadRequest, fmt.Sprintf("limit must be at most %d", maxLimit))
		return
	}

	filter := domain.TaskFilter{Limit: limit, Offset: (page - 1) * limit}
	if raw := r.URL.Query().Get("done"); raw != "" {
		done, err := strconv.ParseBool(raw)
		if err != nil {
			httpx.Error(w, http.StatusBadRequest, "done must be true or false")
			return
		}
		filter.Done = &done
	}

	tasks, total, err := h.tasks.ListPage(r.Context(), userID, filter)
	if err != nil {
		httpx.InternalError(w, r, err)
		return
	}

	httpx.JSON(w, http.StatusOK, TaskListResponse{
		Data: tasks,
		Meta: PageMeta{
			Page:       page,
			Limit:      limit,
			Total:      total,
			TotalPages: (total + limit - 1) / limit, // Integer division, rounded up.
		},
	})
}

// positiveIntParam reads a query parameter, using fallback if it is absent.
func positiveIntParam(r *http.Request, name string, fallback int) (int, error) {
	raw := r.URL.Query().Get(name)
	if raw == "" {
		return fallback, nil
	}
	value, err := strconv.Atoi(raw)
	if err != nil || value < 1 {
		return 0, fmt.Errorf("%s must be a positive integer", name)
	}
	return value, nil
}
```

### Step 8.5: The v2 routes

**File: `internal/handler/v2/routes.go`**
```go
package v2

import (
	"net/http"

	"github.com/yourname/taskapi/internal/auth"
	v1 "github.com/yourname/taskapi/internal/handler/v1"
	"github.com/yourname/taskapi/internal/middleware"
)

// Routes returns the handler for all v2 endpoints. Endpoints that haven't
// changed reuse the v1 handlers; only the list endpoint is new.
func Routes(tasks TaskStore, tokens *auth.TokenManager) http.Handler {
	mux := http.NewServeMux()
	requireAuth := middleware.RequireAuth(tokens)

	taskHandler := NewTaskHandler(tasks)
	mux.Handle("GET /tasks", requireAuth(http.HandlerFunc(taskHandler.List)))

	unchanged := v1.NewTaskHandler(tasks)
	mux.Handle("POST /tasks", requireAuth(http.HandlerFunc(unchanged.Create)))
	mux.Handle("GET /tasks/{id}", requireAuth(http.HandlerFunc(unchanged.Get)))
	mux.Handle("PUT /tasks/{id}", requireAuth(http.HandlerFunc(unchanged.Update)))
	mux.Handle("DELETE /tasks/{id}", requireAuth(http.HandlerFunc(unchanged.Delete)))

	return mux
}
```

`v1.NewTaskHandler(tasks)` accepts a v2 `TaskStore`, because that interface includes all of `v1.TaskStore`'s methods.

### Step 8.6: Mount v2

**File: `internal/server/router.go`**
```go
// Package server assembles the application's routes and middleware.
package server

import (
	"net/http"

	"github.com/yourname/taskapi/internal/auth"
	v1 "github.com/yourname/taskapi/internal/handler/v1"
	v2 "github.com/yourname/taskapi/internal/handler/v2"
	"github.com/yourname/taskapi/internal/httpx"
	"github.com/yourname/taskapi/internal/middleware"
	"github.com/yourname/taskapi/internal/store"
)

// Deps lists everything the router needs from the outside world.
type Deps struct {
	Users  *store.UserStore
	Tasks  *store.TaskStore
	Tokens *auth.TokenManager
}

// NewRouter builds the root HTTP handler for the application.
func NewRouter(deps Deps) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", healthz)

	mux.Handle("/api/v1/", http.StripPrefix("/api/v1", v1.Routes(deps.Users, deps.Tasks, deps.Tokens)))
	mux.Handle("/api/v2/", http.StripPrefix("/api/v2", v2.Routes(deps.Tasks, deps.Tokens)))

	return middleware.Chain(mux, middleware.Logger, middleware.Recover)
}

func healthz(w http.ResponseWriter, r *http.Request) {
	httpx.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}
```

### Checkpoint

```bash
go mod tidy
air
```

```bash
# Create 25 tasks
for i in $(seq 1 25); do
  curl -s -o /dev/null -X POST localhost:8080/api/v1/tasks \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d "{\"title\":\"Task $i\"}"
done

# v1 still returns a plain array, with all 25 tasks
curl -s localhost:8080/api/v1/tasks -H "Authorization: Bearer $TOKEN"

# v2 returns a page of 20 (the default limit): total 25, total_pages 2
curl -s "localhost:8080/api/v2/tasks" -H "Authorization: Bearer $TOKEN"

# Page 2, with 5 tasks per page
curl -s "localhost:8080/api/v2/tasks?page=2&limit=5" -H "Authorization: Bearer $TOKEN"

# Only completed tasks
curl -s "localhost:8080/api/v2/tasks?done=true" -H "Authorization: Bearer $TOKEN"

# Bad input: 400
curl -i "localhost:8080/api/v2/tasks?limit=1000" -H "Authorization: Bearer $TOKEN"

# Check the migration ran: all three should say Applied
goose -dir migrations sqlite3 ./app.db status
```

If you have `jq` installed (`brew install jq`), pipe any of these into it for readable output: `curl -s ... | jq`.

### Commit

```bash
git add .
git commit -m "feat: add v2 API with paginated task listing"
```

### Exercises

1. Mark v1's list endpoint as deprecated: write a middleware that sets a `Deprecation: true` response header, and apply it to that route alone.
2. Add a `sort` query parameter (`newest` or `oldest`) to v2. Don't put the user's text into the SQL. Map each allowed value to a fixed `ORDER BY` string instead.
3. Offset pagination gets slower on deep pages. Read up on cursor pagination, then add a `?after=<id>` option.

---

## Phase 9: Swagger documentation

**Goal:** interactive API documentation generated from your code.

> **Go concepts**
> - **Code generation:** a tool reads your source and writes more Go code, which you commit. It's very common in Go.
> - **Blank imports for side effects:** importing the generated `docs` package registers the spec.
> - **`example` struct tags:** read by `swag`, and ignored by the compiler.

### How it works

1. You write `// @...` comments above `main` (general API info) and above each handler (each endpoint). You've already done the handlers.
2. `swag init` reads those comments and your types, and generates the `docs/` folder.
3. `http-swagger` serves an interactive page that displays the result.

### Annotation quick reference

```go
// Handler-level annotations:
//	@Summary	One-line description
//	@Tags		Groups endpoints together in the UI
//	@Accept		json                          // Request format
//	@Produce	json                          // Response format
//	@Security	BearerAuth                    // Requires the auth scheme defined in main
//	@Param		name  location  type  required  "description"
//	@Success	200  {object}  domain.Task    // A single object
//	@Success	200  {array}   domain.Task    // An array
//	@Failure	404  {object}  httpx.ErrorResponse
//	@Router		/v1/tasks/{id} [get]          // Path after @BasePath, plus method
```

The `location` of an `@Param` can be `path`, `query`, `header` or `body`. A handler can have more than one `@Router` line. The handlers that v2 reuses have one for `/v1` and one for `/v2`, so both appear in the docs.

### Step 9.1: Install the dependency and add the general annotations

```bash
go get github.com/swaggo/http-swagger/v2
go get github.com/swaggo/swag@latest
```

The second command matters. `http-swagger` asks for an old version of the `swag` library, and code generated by a recent `swag` CLI won't compile against it. Your CLI and library versions should stay close: check with `swag --version` and `grep swaggo/swag go.mod`.

These comments must sit directly above `func main`, with no blank line between them.

**File: `cmd/api/main.go`**
```go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/config"
	"github.com/yourname/taskapi/internal/database"
	"github.com/yourname/taskapi/internal/server"
	"github.com/yourname/taskapi/internal/store"
)

// version is replaced at build time (see Phase 11).
var version = "dev"

// @title TaskAPI
// @version 1.0
// @description A multi-user task manager, built for learning Go.
// @BasePath /api
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
// @description Enter "Bearer " followed by your token.
func main() {
	if err := run(); err != nil {
		slog.Error("fatal", "err", err)
		os.Exit(1)
	}
}

func run() error {
	cfg, err := config.Load()
	if err != nil {
		return err
	}

	db, err := database.Open(cfg.DatabasePath)
	if err != nil {
		return err
	}
	defer db.Close()

	if err := database.Migrate(db); err != nil {
		return err
	}

	router := server.NewRouter(server.Deps{
		Users:  store.NewUserStore(db),
		Tasks:  store.NewTaskStore(db),
		Tokens: auth.NewTokenManager(cfg.JWTSecret, cfg.JWTTTL),
	})

	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           router,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	// ctx is cancelled when the process receives Ctrl+C or SIGTERM.
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	// ListenAndServe blocks, so it runs in its own goroutine. The channel is
	// buffered so that the goroutine can always send and then exit.
	serverErr := make(chan error, 1)
	go func() {
		slog.Info("server starting", "addr", srv.Addr, "version", version)
		if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
			serverErr <- err
		}
	}()

	// Wait for whichever happens first: a server failure or a stop signal.
	select {
	case err := <-serverErr:
		return err
	case <-ctx.Done():
		slog.Info("shutting down")
	}

	// Give in-flight requests up to 10 seconds to finish.
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	return srv.Shutdown(shutdownCtx)
}
```

### Step 9.2: Generate the docs

```bash
swag init -g cmd/api/main.go -o docs
```

This creates `docs/docs.go`, `docs/swagger.json` and `docs/swagger.yaml`. Look through `swagger.json` to see what your comments turned into.

**Commit the `docs/` folder.** The router imports it, so a fresh clone wouldn't compile without it.

### Step 9.3: Serve the UI

**File: `internal/server/router.go`**
```go
// Package server assembles the application's routes and middleware.
package server

import (
	"net/http"

	httpSwagger "github.com/swaggo/http-swagger/v2"

	_ "github.com/yourname/taskapi/docs" // Registers the generated Swagger spec.
	"github.com/yourname/taskapi/internal/auth"
	v1 "github.com/yourname/taskapi/internal/handler/v1"
	v2 "github.com/yourname/taskapi/internal/handler/v2"
	"github.com/yourname/taskapi/internal/httpx"
	"github.com/yourname/taskapi/internal/middleware"
	"github.com/yourname/taskapi/internal/store"
)

// Deps lists everything the router needs from the outside world.
type Deps struct {
	Users  *store.UserStore
	Tasks  *store.TaskStore
	Tokens *auth.TokenManager
}

// NewRouter builds the root HTTP handler for the application.
func NewRouter(deps Deps) http.Handler {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", healthz)
	mux.Handle("GET /swagger/", httpSwagger.Handler(httpSwagger.URL("/swagger/doc.json")))

	mux.Handle("/api/v1/", http.StripPrefix("/api/v1", v1.Routes(deps.Users, deps.Tasks, deps.Tokens)))
	mux.Handle("/api/v2/", http.StripPrefix("/api/v2", v2.Routes(deps.Tasks, deps.Tokens)))

	return middleware.Chain(mux, middleware.Logger, middleware.Recover)
}

func healthz(w http.ResponseWriter, r *http.Request) {
	httpx.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}
```

### Step 9.4: Regenerate docs automatically with Air

Change the `cmd` line in `.air.toml`, so that every rebuild refreshes the docs first:

```toml
cmd = "swag init -g cmd/api/main.go -o docs --quiet && go build -o ./tmp/main ./cmd/api"
```

`docs` is already listed in `exclude_dir`. That matters here, since otherwise each regeneration would trigger another rebuild, forever.

### Checkpoint

```bash
go mod tidy
air
```

Open **http://localhost:8080/swagger/index.html** in a browser, then:

1. Expand `POST /v1/auth/login`, click **Try it out**, enter your credentials and click **Execute**. Copy the token from the response.
2. Click **Authorize** at the top and enter `Bearer <your token>`. You must type the word `Bearer` and the space yourself, because Swagger 2.0 has no built-in bearer token type.
3. Try `GET /v2/tasks` with different `page` and `limit` values.

### Commit

```bash
git add .
git commit -m "docs: add swagger documentation and UI"
```

### Exercises

1. Add a `@Description` to one handler, save, and see it appear in the UI.
2. Document `/healthz`. It sits outside `@BasePath`, so what happens to its URL in the UI? Think about how you could solve that.
3. The Swagger UI is public. Make it available only when an `ENV` setting isn't `production`.

---

## Phase 10: Testing

**Goal:** tests at every layer, so that you can change the code without fear.

> **Go concepts**
> - **`net/http/httptest`:** test handlers without opening a real port, or run a real test server.
> - **Fakes via interfaces:** the payoff for the small interfaces you defined in Phases 6 and 7.
> - **Embedding an interface in a fake:** implement only the methods a test needs.
> - **`-race`:** Go's built-in data race detector.
> - **Coverage:** `go test -cover`.

There are four kinds of test in this project, each with a different job:

| Kind | File | What it answers |
|---|---|---|
| Unit | `auth_test.go` | Does this one piece of logic work? |
| Store (already done) | `task_store_test.go` | Does the SQL really work? |
| Handler, with a fake | `task_test.go` | Does the handler map each case to the right HTTP status? |
| End-to-end | `router_test.go` | Does the whole API work together? |

### Step 10.1: Unit tests for auth

**File: `internal/auth/auth_test.go`**
```go
package auth

import (
	"errors"
	"testing"
	"time"
)

const testSecret = "test-secret-that-is-at-least-32-characters"

func TestPasswordRoundTrip(t *testing.T) {
	hash, err := HashPassword("correct-horse-battery")
	if err != nil {
		t.Fatalf("HashPassword: %v", err)
	}
	if !CheckPassword(hash, "correct-horse-battery") {
		t.Error("correct password was rejected")
	}
	if CheckPassword(hash, "wrong-password") {
		t.Error("wrong password was accepted")
	}
}

func TestTokenRoundTrip(t *testing.T) {
	tokens := NewTokenManager(testSecret, time.Hour)

	token, err := tokens.Issue(42)
	if err != nil {
		t.Fatalf("Issue: %v", err)
	}
	userID, err := tokens.Verify(token)
	if err != nil {
		t.Fatalf("Verify: %v", err)
	}
	if userID != 42 {
		t.Errorf("got user %d, want 42", userID)
	}
}

func TestVerifyRejectsBadTokens(t *testing.T) {
	valid := NewTokenManager(testSecret, time.Hour)
	expired := NewTokenManager(testSecret, -time.Hour)
	otherKey := NewTokenManager("a-different-secret-of-at-least-32-chars", time.Hour)

	expiredToken, err := expired.Issue(1)
	if err != nil {
		t.Fatalf("Issue: %v", err)
	}
	wrongKeyToken, err := otherKey.Issue(1)
	if err != nil {
		t.Fatalf("Issue: %v", err)
	}

	cases := []struct {
		name  string
		token string
	}{
		{"expired", expiredToken},
		{"signed with another key", wrongKeyToken},
		{"garbage", "not.a.token"},
		{"empty", ""},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			if _, err := valid.Verify(tc.token); !errors.Is(err, ErrInvalidToken) {
				t.Fatalf("got %v, want %v", err, ErrInvalidToken)
			}
		})
	}
}
```

### Step 10.2: Handler tests with a fake store

This test needs no database. The fake embeds the `TaskStore` interface, which satisfies the compiler without writing every method. Only `Get` is overridden, and calling any other method would panic, which is exactly what you want from a test that shouldn't reach them.

**File: `internal/handler/v1/task_test.go`**
```go
package v1

import (
	"context"
	"errors"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/domain"
)

type fakeTaskStore struct {
	TaskStore // Embedded, so the fake satisfies the interface.
	task      domain.Task
	err       error
}

func (f *fakeTaskStore) Get(ctx context.Context, userID, id int64) (domain.Task, error) {
	return f.task, f.err
}

func TestTaskHandler_Get(t *testing.T) {
	cases := []struct {
		name       string
		id         string
		loggedIn   bool
		storeErr   error
		wantStatus int
	}{
		{"success", "1", true, nil, http.StatusOK},
		{"not found", "1", true, domain.ErrNotFound, http.StatusNotFound},
		{"store failure", "1", true, errors.New("disk on fire"), http.StatusInternalServerError},
		{"non-numeric id", "abc", true, nil, http.StatusBadRequest},
		{"negative id", "-1", true, nil, http.StatusBadRequest},
		{"not logged in", "1", false, nil, http.StatusUnauthorized},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			handler := NewTaskHandler(&fakeTaskStore{err: tc.storeErr})

			req := httptest.NewRequest(http.MethodGet, "/tasks/"+tc.id, nil)
			req.SetPathValue("id", tc.id)
			if tc.loggedIn {
				req = req.WithContext(auth.WithUserID(req.Context(), 1))
			}
			rec := httptest.NewRecorder()

			handler.Get(rec, req)

			if rec.Code != tc.wantStatus {
				t.Errorf("got status %d, want %d, body: %s", rec.Code, tc.wantStatus, rec.Body)
			}
		})
	}
}
```

Forcing a "disk on fire" database failure would be awkward with a real database. With a fake it's one line.

### Step 10.3: End-to-end tests

These run the entire router against a real temporary database, using the actual URLs a client would use.

**File: `internal/server/router_test.go`**
```go
package server

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/http/httptest"
	"path/filepath"
	"testing"
	"time"

	"github.com/yourname/taskapi/internal/auth"
	"github.com/yourname/taskapi/internal/database"
	"github.com/yourname/taskapi/internal/domain"
	v2 "github.com/yourname/taskapi/internal/handler/v2"
	"github.com/yourname/taskapi/internal/store"
)

// testAPI is a small client for a running test server.
type testAPI struct {
	t       *testing.T
	baseURL string
}

func newTestAPI(t *testing.T) *testAPI {
	t.Helper()
	db, err := database.Open(filepath.Join(t.TempDir(), "test.db"))
	if err != nil {
		t.Fatalf("open database: %v", err)
	}
	t.Cleanup(func() { db.Close() })
	if err := database.Migrate(db); err != nil {
		t.Fatalf("migrate: %v", err)
	}

	srv := httptest.NewServer(NewRouter(Deps{
		Users:  store.NewUserStore(db),
		Tasks:  store.NewTaskStore(db),
		Tokens: auth.NewTokenManager("test-secret-that-is-at-least-32-characters", time.Hour),
	}))
	t.Cleanup(srv.Close)

	return &testAPI{t: t, baseURL: srv.URL}
}

// do sends a request, checks the status and decodes the JSON response into out
// (if out is not nil).
func (a *testAPI) do(method, path, token string, body any, wantStatus int, out any) {
	a.t.Helper()

	var reader io.Reader
	if body != nil {
		encoded, err := json.Marshal(body)
		if err != nil {
			a.t.Fatalf("encode body: %v", err)
		}
		reader = bytes.NewReader(encoded)
	}

	req, err := http.NewRequest(method, a.baseURL+path, reader)
	if err != nil {
		a.t.Fatalf("build request: %v", err)
	}
	if token != "" {
		req.Header.Set("Authorization", "Bearer "+token)
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		a.t.Fatalf("%s %s: %v", method, path, err)
	}
	defer resp.Body.Close()

	data, err := io.ReadAll(resp.Body)
	if err != nil {
		a.t.Fatalf("read body: %v", err)
	}
	if resp.StatusCode != wantStatus {
		a.t.Fatalf("%s %s: got status %d, want %d, body: %s", method, path, resp.StatusCode, wantStatus, data)
	}
	if out != nil {
		if err := json.Unmarshal(data, out); err != nil {
			a.t.Fatalf("decode %q: %v", data, err)
		}
	}
}

// signUp registers and logs in a new user, returning their token.
func (a *testAPI) signUp(email string) string {
	a.t.Helper()
	creds := map[string]string{"email": email, "password": "correct-horse-battery"}
	a.do(http.MethodPost, "/api/v1/auth/register", "", creds, http.StatusCreated, nil)

	var login struct {
		Token string `json:"token"`
	}
	a.do(http.MethodPost, "/api/v1/auth/login", "", creds, http.StatusOK, &login)
	return login.Token
}

func TestAuthFlow(t *testing.T) {
	api := newTestAPI(t)
	token := api.signUp("ada@example.com")

	var me domain.User
	api.do(http.MethodGet, "/api/v1/auth/me", token, nil, http.StatusOK, &me)
	if me.Email != "ada@example.com" {
		t.Errorf("got email %q", me.Email)
	}

	creds := map[string]string{"email": "ada@example.com", "password": "correct-horse-battery"}
	api.do(http.MethodPost, "/api/v1/auth/register", "", creds, http.StatusConflict, nil)

	creds["password"] = "wrong-password"
	api.do(http.MethodPost, "/api/v1/auth/login", "", creds, http.StatusUnauthorized, nil)

	api.do(http.MethodGet, "/api/v1/auth/me", "", nil, http.StatusUnauthorized, nil)
	api.do(http.MethodGet, "/api/v1/auth/me", "bad-token", nil, http.StatusUnauthorized, nil)
}

func TestProtectedRoutesRequireAuth(t *testing.T) {
	api := newTestAPI(t)

	routes := []struct{ method, path string }{
		{http.MethodPost, "/api/v1/tasks"},
		{http.MethodGet, "/api/v1/tasks"},
		{http.MethodGet, "/api/v1/tasks/1"},
		{http.MethodPut, "/api/v1/tasks/1"},
		{http.MethodDelete, "/api/v1/tasks/1"},
		{http.MethodPost, "/api/v2/tasks"},
		{http.MethodGet, "/api/v2/tasks"},
		{http.MethodGet, "/api/v2/tasks/1"},
		{http.MethodPut, "/api/v2/tasks/1"},
		{http.MethodDelete, "/api/v2/tasks/1"},
	}
	for _, route := range routes {
		t.Run(route.method+" "+route.path, func(t *testing.T) {
			api.do(route.method, route.path, "", nil, http.StatusUnauthorized, nil)
		})
	}
}

func TestTaskCRUD(t *testing.T) {
	api := newTestAPI(t)
	token := api.signUp("ada@example.com")

	var created domain.Task
	api.do(http.MethodPost, "/api/v1/tasks", token,
		map[string]string{"title": "Learn Go"}, http.StatusCreated, &created)

	path := fmt.Sprintf("/api/v1/tasks/%d", created.ID)

	var updated domain.Task
	api.do(http.MethodPut, path, token,
		map[string]any{"title": "Learn Go well", "done": true}, http.StatusOK, &updated)
	if updated.Title != "Learn Go well" || !updated.Done {
		t.Errorf("unexpected task after update: %+v", updated)
	}

	api.do(http.MethodPost, "/api/v1/tasks", token,
		map[string]string{"title": "  "}, http.StatusBadRequest, nil)
	api.do(http.MethodPost, "/api/v1/tasks", token,
		map[string]string{"titel": "typo"}, http.StatusBadRequest, nil)

	api.do(http.MethodDelete, path, token, nil, http.StatusNoContent, nil)
	api.do(http.MethodGet, path, token, nil, http.StatusNotFound, nil)
}

func TestUsersCannotSeeEachOthersTasks(t *testing.T) {
	api := newTestAPI(t)
	ada := api.signUp("ada@example.com")
	bob := api.signUp("bob@example.com")

	var task domain.Task
	api.do(http.MethodPost, "/api/v1/tasks", ada,
		map[string]string{"title": "Private"}, http.StatusCreated, &task)

	api.do(http.MethodGet, fmt.Sprintf("/api/v1/tasks/%d", task.ID), bob, nil, http.StatusNotFound, nil)

	var list []domain.Task
	api.do(http.MethodGet, "/api/v1/tasks", bob, nil, http.StatusOK, &list)
	if len(list) != 0 {
		t.Errorf("bob sees %d of ada's tasks", len(list))
	}
}

func TestV1AndV2ListFormats(t *testing.T) {
	api := newTestAPI(t)
	token := api.signUp("ada@example.com")

	for i := range 5 {
		var task domain.Task
		api.do(http.MethodPost, "/api/v2/tasks", token,
			map[string]string{"title": fmt.Sprintf("Task %d", i)}, http.StatusCreated, &task)
		if i%2 == 0 {
			api.do(http.MethodPut, fmt.Sprintf("/api/v2/tasks/%d", task.ID), token,
				map[string]any{"title": task.Title, "done": true}, http.StatusOK, nil)
		}
	}

	// v1 keeps its original contract: a plain array of everything.
	var v1List []domain.Task
	api.do(http.MethodGet, "/api/v1/tasks", token, nil, http.StatusOK, &v1List)
	if len(v1List) != 5 {
		t.Errorf("v1: got %d tasks, want 5", len(v1List))
	}

	cases := []struct {
		query          string
		wantItems      int
		wantTotal      int
		wantTotalPages int
	}{
		{"", 5, 5, 1},
		{"?limit=2", 2, 5, 3},
		{"?limit=2&page=3", 1, 5, 3},
		{"?limit=2&page=4", 0, 5, 3},
		{"?done=true", 3, 3, 1},
		{"?done=false", 2, 2, 1},
	}
	for _, tc := range cases {
		t.Run("v2"+tc.query, func(t *testing.T) {
			var got v2.TaskListResponse
			api.do(http.MethodGet, "/api/v2/tasks"+tc.query, token, nil, http.StatusOK, &got)
			if len(got.Data) != tc.wantItems || got.Meta.Total != tc.wantTotal || got.Meta.TotalPages != tc.wantTotalPages {
				t.Errorf("got %d items, meta %+v", len(got.Data), got.Meta)
			}
		})
	}

	for _, query := range []string{"?page=0", "?limit=0", "?limit=101", "?page=abc", "?done=maybe"} {
		t.Run("v2 invalid"+query, func(t *testing.T) {
			api.do(http.MethodGet, "/api/v2/tasks"+query, token, nil, http.StatusBadRequest, nil)
		})
	}
}
```

### Checkpoint

```bash
go test ./...                                  # Everything
go test -v -run TestTaskCRUD ./internal/server # One test by name
go test -race ./...                            # Detect data races
go test -cover ./...                           # Coverage per package

# See exactly which lines the tests miss, in a browser
go test -coverprofile=cover.out ./...
go tool cover -html=cover.out
rm cover.out
```

Don't chase 100% coverage. Use the report to find important branches that nothing tests.

### Commit

```bash
git add .
git commit -m "test: add auth, handler and end-to-end tests"
```

### Exercises

1. Write a table-driven test for `Register`'s validation: bad emails, short passwords and 73-character passwords.
2. Break something on purpose (such as removing `AND user_id = ?` from `Get` in `task_store.go`) and confirm that a test catches it. Then restore it.
3. Add a test for the `Recover` middleware, with a handler that panics.

---

## Phase 11: Makefile and your first release

**Goal:** one-word commands for common tasks, plus a tagged release whose binary knows its own version.

> **Go concepts**
> - **Linker flags:** `-ldflags "-X main.version=v1.0.0"` sets a package variable at build time. That's what `var version = "dev"` has been waiting for.
> - **Static binaries:** `go build` produces a single file with everything inside it, including your migrations.
> - **Formatting and linting:** `gofmt` and `go vet` ship with Go.

### Step 11.1: The Makefile

Makefile recipes must be indented with a real **tab**, not spaces.

**File: `Makefile`**
```makefile
VERSION := $(shell git describe --tags --always --dirty)
DB      ?= ./app.db

.PHONY: dev build test lint docs migrate-status migrate-up migrate-down migrate-new

dev: ## Run with live reload
	air

build: docs ## Build the production binary, stamped with the Git version
	go build -ldflags "-X main.version=$(VERSION)" -o bin/taskapi ./cmd/api

test: ## Run all tests with the race detector
	go test -race ./...

lint: ## Check formatting and common mistakes
	@test -z "$$(gofmt -l .)" || (echo "Run: gofmt -w ." && gofmt -l . && exit 1)
	go vet ./...

docs: ## Regenerate the Swagger docs
	swag init -g cmd/api/main.go -o docs --quiet

migrate-status: ## Show which migrations have run
	goose -dir migrations sqlite3 $(DB) status

migrate-up: ## Apply all pending migrations
	goose -dir migrations sqlite3 $(DB) up

migrate-down: ## Undo the latest migration
	goose -dir migrations sqlite3 $(DB) down

migrate-new: ## Create a migration: make migrate-new name=add_something
	goose -dir migrations -s create $(name) sql
```

### Step 11.2: Check everything

```bash
make lint
make test
make build
```

### Step 11.3: Tag a release

A Git tag marks an exact commit as a release. Use [semantic versioning](https://semver.org), `vMAJOR.MINOR.PATCH`: major for breaking changes, minor for new features, and patch for bug fixes.

Don't confuse the two kinds of version in this project. The **Git tag** (`v1.0.0`) identifies a release of your program. The **API version** (`/api/v2`) is a promise to the clients of your API. They change independently.

```bash
git add .
git commit -m "chore: add makefile"

git tag -a v1.0.0 -m "First release: auth, tasks, API v1 and v2, swagger"
git tag                      # List tags
git push origin main --tags  # If you added a GitHub remote

make build                   # Now stamped with the tag
```

### Checkpoint

Run the built binary directly. No Go, Air or source code is involved, and the migrations are inside it:

```bash
./bin/taskapi
```

The startup log should show `version=v1.0.0`. Delete `app.db` first if you want to watch it create a new database from nothing.

To build for a Linux server from your Mac, set the target operating system and CPU:

```bash
GOOS=linux GOARCH=amd64 go build -o bin/taskapi-linux ./cmd/api
```

This is possible thanks to the pure Go SQLite driver you chose in Phase 4.

---

## Phase 12: Where to go next

You now have a complete API. Each of these extends it and teaches something new, roughly in order of difficulty.

| Feature | What you learn |
|---|---|
| Refresh tokens stored in a new table | Token rotation, revocation, transactions (`db.BeginTx`) |
| Rate limiting on `/auth/login` | Concurrency with `sync.Mutex`, `golang.org/x/time/rate` |
| Request ID in every log line | Using context across layers, `slog.With` |
| Replace hand-written SQL with [sqlc](https://sqlc.dev) | Code generation from SQL |
| Dockerfile (multi-stage build) | Containers, small production images |
| GitHub Actions running `make lint test` | Continuous integration |
| Task tags (a many-to-many relationship) | Joins, transactions, more migrations |
| Background job that emails overdue tasks | Goroutines, tickers, worker pools, `errgroup` |
| Move from SQLite to Postgres | Why the store layer's separation pays off |

Recommended reading:

- [A Tour of Go](https://go.dev/tour): the official interactive tutorial
- [Effective Go](https://go.dev/doc/effective_go): how to write idiomatic Go
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments): common style issues
- *Let's Go* and *Let's Go Further* by Alex Edwards: books on building web apps and APIs in Go

---

## Appendix A: Go syntax cheat sheet

Every construct below appears somewhere in this project.

```go
// ----- Variables -----
var count int              // Zero value: 0
name := "Ada"              // Declare and infer the type (inside functions only)
const maxItems = 100

// ----- Functions -----
func add(a, b int) int { return a + b }
func divide(a, b int) (int, error) {            // Multiple return values
	if b == 0 {
		return 0, errors.New("divide by zero")
	}
	return a / b, nil
}
func sum(nums ...int) int { /* nums is a []int */ return 0 } // Variadic

// ----- Structs and methods -----
type User struct {
	ID    int64  `json:"id"`                    // Struct tag
	Email string `json:"email"`
}
u := User{ID: 1, Email: "ada@example.com"}      // Create
p := &u                                         // Pointer to u
func NewUser(email string) *User { return &User{Email: email} }  // Constructor
func (u *User) SetEmail(e string) { u.Email = e }  // Pointer receiver: can modify
func (u User) Domain() string { return "" }        // Value receiver: gets a copy

// ----- Interfaces (satisfied automatically) -----
type Store interface {
	Get(ctx context.Context, id int64) (User, error)
}

// ----- Errors -----
var ErrNotFound = errors.New("not found")       // Sentinel error
err := fmt.Errorf("load user %d: %w", id, ErrNotFound)  // Wrap with context
errors.Is(err, ErrNotFound)                     // Is it, or does it wrap, this value?
var target *sqlite.Error
errors.As(err, &target)                         // Is it, or does it wrap, this type?

// ----- Slices and maps -----
items := []string{"a", "b"}
items = append(items, "c")
for i, item := range items { fmt.Println(i, item) }
for i := range 5 { fmt.Println(i) }             // 0 to 4
ages := map[string]int{"ada": 36}
age, ok := ages["bob"]                          // ok is false if the key is missing

// ----- Control flow -----
if err := doThing(); err != nil { return err }  // Scoped to the if
switch {
case x < 0:
	return "negative"
default:
	return "non-negative"
}

// ----- Defer, goroutines, channels, select -----
defer file.Close()                              // Runs when the function returns
ch := make(chan error, 1)                       // Buffered channel
go func() { ch <- doWork() }()                  // Run concurrently, send the result
select {
case err := <-ch:
	fmt.Println(err)
case <-ctx.Done():
	fmt.Println("cancelled")
}

// ----- Generics -----
func First[T any](items []T) (T, bool) {
	var zero T
	if len(items) == 0 {
		return zero, false
	}
	return items[0], true
}

// ----- Type assertion -----
id, ok := value.(int64)                         // ok is false if it isn't an int64

// ----- HTTP -----
func handler(w http.ResponseWriter, r *http.Request) {}           // Handler
func mw(next http.Handler) http.Handler {                         // Middleware
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		next.ServeHTTP(w, r)
	})
}
mux.HandleFunc("GET /tasks/{id}", handler)      // Route with a method and parameter
r.PathValue("id")                               // Path parameter
r.URL.Query().Get("page")                       // Query parameter
r.Header.Get("Authorization")                   // Request header

// ----- Tests -----
func TestAdd(t *testing.T) {
	cases := []struct {
		name       string
		a, b, want int
	}{
		{"positives", 1, 2, 3},
		{"negatives", -1, -2, -3},
	}
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			if got := add(tc.a, tc.b); got != tc.want {
				t.Errorf("got %d, want %d", got, tc.want)
			}
		})
	}
}
```

---

## Appendix B: Troubleshooting

| Problem | Cause and fix |
|---|---|
| `air: command not found` | `$(go env GOPATH)/bin` isn't on your `PATH`. See Step 0.1. |
| `no required module provides package ...` | Run `go mod tidy`. |
| `found packages x and y in one folder` | All `.go` files in a folder must share one `package` name (only `_test` packages are an exception). |
| `import cycle not allowed` | Two packages import each other. Move the shared types into `domain`. |
| `declared and not used` or `imported and not used` | Go treats these as compile errors. Remove them, or assign to `_` while experimenting. |
| `JWT_SECRET must be at least 32 characters` | `.env` is missing or wrong. See Step 3.2. Run the server from the project root so that `.env` gets found. |
| `address already in use` | A previous server is still running. Find it with `lsof -i :8080`, then stop it with `kill <PID>`. |
| Air rebuilds in an endless loop | A generated folder is being watched. Check `exclude_dir` includes `docs` and `tmp`. |
| `pattern ... conflicts with pattern ...` panic at startup | Two routes overlap on your mux. Check for duplicates in `routes.go`. |
| Request hangs forever | With `SetMaxOpenConns(1)`, you started a new query before closing `rows` or ending a transaction. See Step 4.4. |
| `SQL logic error: no such table` | Migrations haven't run on this database. Check `DATABASE_PATH`, and run `make migrate-status`. |
| `goose: missing "-- +goose Up"` or similar | The annotation must be exactly `-- +goose Up`, at the start of a line. |
| Goose says `migrations directory does not exist` | Run from the project root, and create the folder with `mkdir -p migrations`. |
| `pattern *.sql: no matching files found` | `embed.go` must be in the same folder as the `.sql` files. |
| `/swagger/index.html` is a 404 or shows an old API | Run `swag init -g cmd/api/main.go -o docs`. Check the route is registered, and hard-refresh your browser. |
| `unknown field LeftDelim in struct literal` in `docs/docs.go` | The `swag` library is older than the CLI. Run `go get github.com/swaggo/swag@latest`. |
| `swag` says `cannot find type definition` | Check the type's name and package in the annotation, such as `domain.Task`. |
| Swagger "Authorize" still gets a 401 | Enter `Bearer <token>`, including the word `Bearer` and a space. |
| Every protected request returns 401 after a restart | You changed `JWT_SECRET`, so older tokens no longer verify. Log in again. |
| Task list returns `null` instead of `[]` | The slice is `nil`. Initialise it with `[]domain.Task{}`. |
| Handler writes two responses (`superfluous WriteHeader` in logs) | You forgot a `return` after writing an error. |
| Accidentally committed `.env` | Run `git rm --cached .env`, commit, and **change the secret**, because it remains in your Git history. |
