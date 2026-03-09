---
name: clean-ddd-hexagonal
description: Proactively apply when designing APIs, microservices, or scalable backend structure. Triggers on DDD, Clean Architecture, Hexagonal, ports and adapters, entities, value objects, domain events, CQRS, event sourcing, repository pattern, use cases, onion architecture, outbox pattern, aggregate root, anti-corruption layer. Use when working with domain models, aggregates, repositories, or bounded contexts. Clean Architecture + DDD + Hexagonal patterns for backend services, language-agnostic (Go, Rust, Python, TypeScript, Java, C#).
---

# Clean Architecture + DDD + Hexagonal (Node.js / TypeScript)

Hexagonal + DDD with **Vertical Slicing** per module inside each layer. Dependency flow strictly inward: `infrastructure → application → domain`. Domain layer has zero external dependencies.

## Architecture

```
src/
├── domain/
│   ├── ports/             # Cross-module ports (e.g. ILogger)
│   └── {module}/
│       ├── entities/
│       ├── errors/
│       └── ports/         # Repository + external service interfaces
├── application/
│   └── {module}/
│       ├── use-cases/
│       └── dto/
├── infrastructure/
│   ├── {module}/
│   │   ├── adapters/      # Port implementations for this module
│   │   └── entry-points/  # controller, routes/, schemas/, middlewares/
│   ├── adapters/          # GLOBAL adapters used by multiple modules (e.g. pino-logger.adapter.ts)
│   ├── entry-points/      # SHARED HTTP infra: BaseController, server.ts, docs/
│   └── config/            # env vars, prisma client, bootstrap
├── shared/
│   └── errors/            # DomainError, NotFoundError base classes
└── app.ts                 # Composition root — all DI happens here
```

**Global `adapters/`**: only adapters that implement cross-cutting ports (ILogger, error reporting).

---

## TypeScript & Import Rules

Rules enforced by ESLint and TypeScript compiler. A lint or type error means the task is **not done**.

### Path aliases — never use relative `../../` imports

| Alias | Maps to |
|---|---|
| `@domain/*` | `src/domain/*` |
| `@application/*` | `src/application/*` |
| `@infra/*` | `src/infrastructure/*` |
| `@shared/*` | `src/shared/*` |

### `.js` extension — always required in imports

Node ESM resolves extensions literally. Always write `.js` even inside `.ts` files.

```typescript
// ✅
import { CreateUserUseCase } from '@application/user/use-cases/create-user.use-case.js';
// ❌
import { CreateUserUseCase } from '@application/user/use-cases/create-user.use-case';
```

### `import type` — required for type-only imports

`verbatimModuleSyntax: true` is enabled. Interfaces, type aliases, and anything used only as a type annotation → `import type`.

```typescript
// ✅
import type { IUserRepository } from '@domain/user/ports/user.repository.port.js';
// ❌
import { IUserRepository } from '@domain/user/ports/user.repository.port.js';
```

### `any` is forbidden

Use `unknown` for unknown values. For type casts in tests use `as unknown as TargetType`.

---

## Key Conventions

**File naming:**
- Port interfaces: `*.port.ts`
- Adapters (port implementations): `*.adapter.ts`
- Use cases: `*.use-case.ts`
- Tests: `*.spec.ts`

**One use case = one file = one class = one `execute()` method.** No exceptions. Never add methods other than `execute`.

**Dependency injection:** All dependencies injected via constructor. Use cases receive ports (interfaces), never concrete adapters.

---

## ID Generation

Three distinct roles, three distinct tools — ALWAYS separate, never collapsed:

| Role | Tool | Why |
|------|------|-----|
| Entity PK (stored in DB) | `uuidv7()` from `uuidv7` package | Timestamp-prefixed → sequential inserts |
| Opaque security token (sent to client) | `randomUUID()` from `node:crypto` | Must be unpredictable; UUID v7 leaks timestamp |
| Token lookup hash (stored in DB) | `createHash('sha256')` from `node:crypto` | Deterministic → O(1) lookup |

```typescript
// ✅ Three separate values
const tokenId        = uuidv7();
const plaintextToken = randomUUID();
const tokenHash      = createHash('sha256').update(plaintextToken).digest('hex');
```

Repository ports for token lookup must expose `findByTokenHash(hash: string)`, NOT `findById(tokenValue)`.

---

## Module Creation Playbook

Create files in this order — domain first, infrastructure last.

```
1. src/domain/{module}/ports/{name}.repository.port.ts
2. src/domain/{module}/entities/{name}.entity.ts
3. src/domain/{module}/errors/{name}-not-found.error.ts
   src/domain/{module}/errors/invalid-{rule}.error.ts
4. src/application/{module}/dto/create-{module}.dto.ts
   src/application/{module}/dto/get-{module}.dto.ts
   src/application/{module}/dto/list-{module}s.dto.ts
   src/application/{module}/dto/update-{module}.dto.ts
   src/application/{module}/dto/delete-{module}.dto.ts   ← Command only, no Result
5. src/application/{module}/use-cases/create-{module}.use-case.ts
   src/application/{module}/use-cases/get-{module}.use-case.ts
   src/application/{module}/use-cases/list-{module}s.use-case.ts
   src/application/{module}/use-cases/update-{module}.use-case.ts
   src/application/{module}/use-cases/delete-{module}.use-case.ts
6. src/infrastructure/{module}/adapters/{name}.adapter.ts
7. src/infrastructure/{module}/entry-points/{name}.controller.ts
   src/infrastructure/{module}/entry-points/routes/{name}.routes.ts
   src/infrastructure/{module}/entry-points/schemas/{name}.schemas.ts
8. Wire in app.ts
```

**Tests are created at the same time as the implementation — never after.**

Each file has exactly one spec file in a `__test__/` folder inside the same directory:

```
src/application/{module}/use-cases/
  {action}.use-case.ts
  __test__/{action}.use-case.spec.ts

src/domain/{module}/entities/
  {name}.entity.ts
  __test__/{name}.entity.spec.ts

src/infrastructure/{module}/adapters/
  {name}.adapter.ts
  __test__/{name}.adapter.spec.ts

src/infrastructure/{module}/entry-points/
  {name}.controller.ts
  __test__/{name}.controller.spec.ts
```

Every spec MUST cover: happy path, every domain error, every port failure, every input edge case.
Goal: **100% branch coverage** on use case files. Uncovered branches = task not done.

---

## Entity Pattern

Private constructor, two static factories. ID always received as parameter.

```typescript
export class User {
  private constructor(
    public readonly id: string,
    public readonly email: string,
    public readonly name: string,
  ) {}

  static create(id: string, email: string, name: string): User {
    if (!email.includes('@')) throw new InvalidEmailError(email);
    return new User(id, email, name);
  }

  static reconstitute(id: string, email: string, name: string): User {
    return new User(id, email, name);
  }
}
```

- `create()` → new entities: validate invariants, throw DomainError if violated
- `reconstitute()` → loading from DB: no validation, just reconstruct state

---

## DTOs

Plain TypeScript interfaces — no Zod, no classes. One file per operation.

```typescript
export interface Create{Module}Command { id: string; /* domain fields */ }
export interface Create{Module}Result  { id: string; /* return fields */ }

// Delete has only Command, no Result — execute() returns Promise<void>
export interface Delete{Module}Command { id: string; }
```

Use cases return **plain objects** matching the Result interface — never entity instances.

---

## Repository Port Contract

CRUD baseline — `save()` = insert, `update()` = update, never a single upsert.

```typescript
export interface I{Module}Repository {
  findAll(): Promise<{Entity}[]>;
  findById(id: string): Promise<{Entity} | null>;
  save(entity: {Entity}): Promise<void>;
  update(entity: {Entity}): Promise<void>;
  delete(id: string): Promise<void>;
}
```

---

## Domain Errors

```typescript
// src/shared/errors/domain.error.ts
export abstract class DomainError extends Error {
  constructor(message: string) { super(message); this.name = this.constructor.name; }
}

// src/shared/errors/not-found.error.ts
export abstract class NotFoundError extends DomainError {}

// src/domain/{module}/errors/{entity}-not-found.error.ts
export class UserNotFoundError extends NotFoundError {
  constructor(id: string) { super(`User not found: ${id}`); }
}
```

`BaseController` maps automatically: `NotFoundError → 404`, `DomainError → 400`, anything else `→ 500`.
Never `throw new Error()` — always a typed subclass.

---

## ILogger

Cross-module port. Never import Pino directly in domain or application.

```typescript
// src/domain/ports/logger.port.ts
export interface ILogger {
  info(message: string, context?: Record<string, unknown>): void;
  error(message: string, context?: Record<string, unknown>): void;
  warn(message: string, context?: Record<string, unknown>): void;
  debug(message: string, context?: Record<string, unknown>): void;
}
```

---

## Central Error Handler

All controllers extend `BaseController`. No individual error handling in controllers.

```typescript
// src/infrastructure/entry-points/base.controller.ts
export interface HttpRequest  { body?: unknown; params?: Record<string, string>; query?: Record<string, string>; }
export interface HttpResponse { status: number; body: unknown; }
export interface ErrorResponse { status: number; message: string; }

export abstract class BaseController {
  protected async handleRequest<T>(
    action: () => Promise<T>,
    onSuccess: (result: T) => HttpResponse,
    onError: (error: ErrorResponse) => HttpResponse,
  ): Promise<HttpResponse> {
    try {
      return onSuccess(await action());
    } catch (error) {
      if (error instanceof NotFoundError)  return onError({ status: 404, message: error.message });
      if (error instanceof DomainError)    return onError({ status: 400, message: error.message });
      return onError({ status: 500, message: 'Internal server error' });
    }
  }
}
```

Controller pattern — one use case injected per operation:

```typescript
export class UserController extends BaseController {
  constructor(private readonly createUseCase: CreateUserUseCase) { super(); }

  async create(req: HttpRequest): Promise<HttpResponse> {
    const parsed = CreateUserSchema.safeParse(req.body);
    if (!parsed.success) return { status: 400, body: { error: parsed.error.message } };

    return this.handleRequest(
      () => this.createUseCase.execute(parsed.data),
      (result) => ({ status: 201, body: result }),
      (error)  => ({ status: error.status, body: { error: error.message } }),
    );
  }
}
```

---

## Validation

Zod belongs **only** in `infrastructure/{module}/entry-points/schemas/`. Never in domain or application.

- Schema files: `{module}.schemas.ts`
- Request schemas: `PascalCase` (e.g. `CreateUserSchema`)
- Reusable sub-schemas: `camelCase` (e.g. `passwordSchema`)

Always use `safeParse` (never `parse`). Always use `parsed.error.message` (never `.format()` or `.issues`).

---

## HTTP Status Codes

| Operation | Success |
|---|---|
| Create (POST) | `201` |
| Get / List (GET) | `200` |
| Update (PUT/PATCH) | `200` |
| Delete (DELETE) | `204` — body is `null` |
| Validation failure | `400` — returned before `handleRequest` |

Error body shape: `{ error: string }`.

---

## Testing Patterns

**Mock ports as classes** — never `vi.fn()` directly on a port. Implement the interface — catches type mismatches.

```typescript
class MockLogger implements ILogger {
  info = vi.fn(); error = vi.fn(); warn = vi.fn(); debug = vi.fn();
}

class MockUserRepository implements IUserRepository {
  private store: User[] = [];
  async findAll() { return [...this.store]; }
  async findById(id: string) { return this.store.find(u => u.id === id) ?? null; }
  async save(u: User) { this.store.push(u); }
  async update(u: User) { const i = this.store.findIndex(x => x.id === u.id); if (i >= 0) this.store[i] = u; }
  async delete(id: string) { this.store = this.store.filter(u => u.id !== id); }
}
```

---

## Anti-Patterns

| Never do this | Why |
|---|---|
| Import `express` or any HTTP lib outside `infrastructure/entry-points/` | Couples business logic to a protocol |
| Import a repository adapter directly in a use case | Breaks DI — use the port interface |
| Use `vi.fn()` to mock a port | Doesn't enforce the interface contract |
| Call `console.log` in domain or application | Inject `ILogger` instead |
| Put Zod schemas in domain or application | Validation is an infrastructure concern |
| Define Zod schemas inline in a controller | Schemas go in `schemas/{module}.schemas.ts` |
| Throw `new Error()` from domain | Use a typed `DomainError` subclass |
| Add business logic to a controller | Controllers only translate, never decide |
| Access `process.env` outside `infrastructure/config/` | Centralizes env coupling |
| Use `create()` in adapters when loading from DB | Use `reconstitute()` |
| Generate IDs inside the entity | ID source is an infrastructure decision |
| Merge `save()` and `update()` into upsert | Keep intent explicit |
| Handle errors in individual controllers | Use `handleRequest()` from `BaseController` |
| Use `any` | `no-explicit-any` is an ESLint error — use `unknown` |
| Use `as any` | Use `as unknown as TargetType` |

---

## Tooling

- **Runtime:** Node.js ESM — `import/export` only, never `require()`
- **TypeScript:** `strict: true`, `verbatimModuleSyntax: true`, `isolatedModules: true`
- **ESLint:** Flat config. `no-empty-interface` is OFF (domain ports use empty interfaces as markers)
