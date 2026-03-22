# Guideline Categories Reference

This is a lookup table for `/guidelines-init`. It provides concrete example rules
per tech stack and category. Init should read only the sections relevant to the
detected stack and use these as inspiration — not copy them verbatim. Adapt
examples to the specific project's patterns.

---

## Code Style

### JavaScript / TypeScript
<!-- severity: warning -->
- Use `camelCase` for variables and functions, `PascalCase` for classes and types, `UPPER_SNAKE_CASE` for constants
  ```ts
  // good
  const maxRetries = 3;
  function fetchUserById(id: string): Promise<User> { ... }
  class UserService { ... }
  type RequestOptions = { ... };
  const MAX_RETRY_DELAY = 5000;

  // bad
  const max_retries = 3;
  function FetchUser(id: string) { ... }
  ```
- Prefer named exports over default exports for better refactoring and auto-import support
- Use `const` by default; use `let` only when reassignment is necessary; never use `var`

### Python
<!-- severity: warning -->
- Follow PEP 8: `snake_case` for functions/variables, `PascalCase` for classes, `UPPER_SNAKE_CASE` for module-level constants
  ```python
  # good
  def fetch_user_by_id(user_id: str) -> User: ...
  class UserService: ...
  MAX_RETRY_DELAY = 5000

  # bad
  def fetchUserById(userId): ...
  ```
- Use type hints on all public function signatures
- Prefer f-strings over `.format()` or `%` formatting

### Go
<!-- severity: warning -->
- Follow Go conventions: `camelCase` for unexported, `PascalCase` for exported
- Use short variable names in small scopes (`i`, `err`, `ctx`), descriptive names in larger scopes
- Receivers: use short 1-2 letter names consistent within the type (`func (s *Server) Start()`)

### Rust
<!-- severity: warning -->
- Follow Rust conventions: `snake_case` for functions/variables, `PascalCase` for types/traits, `UPPER_SNAKE_CASE` for constants
- Prefer `impl Trait` over `dyn Trait` for function parameters when a single type is expected
- Use `clippy` lint attributes to suppress false positives, not to hide real issues

---

## Architecture

### JavaScript / TypeScript
<!-- severity: error -->
- Group by feature, not by type: `features/auth/` not `controllers/`, `models/`, `services/`
  ```
  # good
  src/features/auth/auth.controller.ts
  src/features/auth/auth.service.ts
  src/features/auth/auth.model.ts

  # bad
  src/controllers/auth.controller.ts
  src/services/auth.service.ts
  src/models/auth.model.ts
  ```
- No circular imports between feature directories
- Shared utilities go in `src/shared/` or `src/lib/`, never imported from another feature's internals

### Python
<!-- severity: error -->
- Separate domain logic from framework code: business rules should not import Flask/Django/FastAPI
- Use `__init__.py` to define the public API of each package; internal modules start with `_`

### Go
<!-- severity: error -->
- Follow standard project layout: `cmd/` for entrypoints, `internal/` for private packages, `pkg/` for public libraries
- No imports from `internal/` across module boundaries
- Keep `main.go` thin — wire dependencies and call into internal packages

### Rust
<!-- severity: error -->
- Use workspace members for distinct binaries/libraries
- Keep `mod.rs` files as re-exports only; logic goes in named modules

---

## Error Handling

### JavaScript / TypeScript
<!-- severity: error -->
- Never use bare `catch {}` or `catch (e) {}` without handling or rethrowing
  ```ts
  // good
  try { await save(data); }
  catch (err) {
    logger.error("Failed to save", { err, data });
    throw new AppError("SAVE_FAILED", { cause: err });
  }

  // bad
  try { await save(data); }
  catch (e) { /* ignore */ }
  ```
- Use typed error classes that extend a base `AppError` for domain errors
- Always await promises or return them; never fire-and-forget without explicit `void`

### Python
<!-- severity: error -->
- Never use bare `except:` or `except Exception:` without logging or rethrowing
  ```python
  # good
  try:
      save(data)
  except DatabaseError as e:
      logger.error("Failed to save", exc_info=e)
      raise AppError("SAVE_FAILED") from e

  # bad
  try:
      save(data)
  except:
      pass
  ```
- Use custom exception classes that inherit from a project-specific base exception
- Let unexpected exceptions propagate to the global handler

### Go
<!-- severity: error -->
- Always check returned errors; never assign to `_` without a comment explaining why
  ```go
  // good
  if err := db.Save(data); err != nil {
      return fmt.Errorf("saving user: %w", err)
  }

  // bad
  db.Save(data)
  ```
- Wrap errors with context using `fmt.Errorf("doing X: %w", err)`
- Use sentinel errors (`var ErrNotFound = errors.New(...)`) for expected failure modes

### Rust
<!-- severity: error -->
- Use `Result<T, E>` for fallible operations; avoid `.unwrap()` outside of tests
- Define a crate-level `Error` enum with `thiserror` for library crates
- Use `anyhow::Result` in application code, `thiserror` in library code

---

## Testing

### JavaScript / TypeScript
<!-- severity: warning -->
- Test files co-located with source: `user.service.ts` → `user.service.test.ts`
- Name tests descriptively: `it("returns 404 when user does not exist")`
- Use `describe` blocks to group related tests by function or behavior
- Mock external services (HTTP, database) but not internal modules

### Python
<!-- severity: warning -->
- Test files in `tests/` directory mirroring `src/` structure, or co-located with `test_` prefix
- Use `pytest` with fixtures for setup/teardown
- Use `pytest.mark.parametrize` for data-driven tests
- Prefer `unittest.mock.patch` on the import path, not the definition path

### Go
<!-- severity: warning -->
- Test files co-located: `user.go` → `user_test.go`
- Use table-driven tests for multiple input/output scenarios
  ```go
  tests := []struct {
      name    string
      input   string
      want    int
      wantErr bool
  }{ ... }
  for _, tt := range tests {
      t.Run(tt.name, func(t *testing.T) { ... })
  }
  ```
- Use `testify/assert` or `testify/require` for assertions, or stick to stdlib `if got != want`

### Rust
<!-- severity: warning -->
- Unit tests in `#[cfg(test)] mod tests` at the bottom of the source file
- Integration tests in `tests/` directory
- Use `#[should_panic]` or `Result<(), Error>` return types for error path tests

---

## Git Workflow

### General
<!-- severity: warning -->
- Use conventional commit format: `type(scope): description`
  - Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `ci`
  - Example: `feat(auth): add password reset flow`
- Branch naming: `type/short-description` (e.g., `feat/password-reset`, `fix/login-timeout`)
- Keep PRs focused — one logical change per PR
- Squash-merge feature branches to keep main history clean

---

## Documentation

### General
<!-- severity: warning -->
- Public functions/methods must have docstrings/JSDoc explaining purpose, parameters, and return value
- README must include: project description, setup instructions, how to run tests, how to deploy
- Use `TODO(name):` format for todos so they are attributable and searchable
- Inline comments explain *why*, not *what* — the code shows what

---

## Security

### General
<!-- severity: error -->
- Never commit secrets, API keys, or credentials — use environment variables or a secrets manager
- Validate and sanitize all external input at system boundaries (API endpoints, CLI args, file uploads)
- Use parameterized queries for all database access — never concatenate user input into SQL
  ```ts
  // good
  db.query("SELECT * FROM users WHERE id = $1", [userId]);

  // bad
  db.query(`SELECT * FROM users WHERE id = '${userId}'`);
  ```
- Pin dependencies to exact versions or use lockfiles; review dependency updates before merging
- Never use `eval()`, `Function()`, or `innerHTML` with untrusted input

---

## Performance

### General
<!-- severity: info -->
- Avoid N+1 query patterns — use eager loading, joins, or batch queries
  ```ts
  // good: single query with join
  const orders = await db.query("SELECT * FROM orders JOIN users ON ...");

  // bad: N+1
  const users = await db.query("SELECT * FROM users");
  for (const user of users) {
    user.orders = await db.query("SELECT * FROM orders WHERE user_id = $1", [user.id]);
  }
  ```
- Add pagination to all list endpoints; never return unbounded result sets
- Prefer async/non-blocking I/O; avoid synchronous file or network operations in request handlers
- Cache expensive computations and frequently-read data with appropriate TTLs
