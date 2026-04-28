# Software Architecture

How to organize code at scale: layers, boundaries, dependencies, feature structure. **Operationalized** — every guideline has a tell to detect violation.

## Decision Matrix — Architecture Choice

| Project shape | Recommended | Why |
|---------------|-------------|-----|
| Solo dev / MVP / < 5K lines | Layered or none | Architecture overhead exceeds benefit |
| Team 2-10 / domain logic / 5K-50K lines | Hexagonal / Clean | Right tradeoff for testability + decoupling |
| Multiple teams / 50K+ lines / multiple bounded contexts | Modular monolith → microservices | Boundaries become organizational |
| Distributed system / independent deployment | Microservices | Independent scaling + deploy |

**Don't pick the architecture before the problem demands it.** Most "you need microservices" arguments are premature.

---

## The Goal of Architecture

Enable the team to:

1. **Add** features with minimal friction
2. **Change** existing features safely
3. **Remove** features cleanly
4. **Test** features in isolation
5. **Deploy** independently when valuable

If your architecture makes any of these harder, it's wrong for your context.

---

## Two Boundaries: Vertical and Horizontal

### Vertical (features / slices) — organize by what changes together

```
❌ Layer-first — changes ripple across folders
src/
  controllers/
    UserController.ts
    OrderController.ts
  services/
    UserService.ts
    OrderService.ts
  repositories/
    UserRepository.ts
    OrderRepository.ts

✅ Feature-first — changes localized
src/
  modules/
    users/
      UserController.ts
      UserService.ts
      UserRepository.ts
    orders/
      OrderController.ts
      OrderService.ts
      OrderRepository.ts
```

**Tell for vertical violation:** adding a feature requires editing files in 4+ folders.

### Horizontal (layers) — separate concerns

```
┌──────────────────────────────────────┐
│  Presentation                        │  ← UI, controllers, CLI
├──────────────────────────────────────┤
│  Application                         │  ← use cases, orchestration
├──────────────────────────────────────┤
│  Domain                              │  ← business logic, entities
├──────────────────────────────────────┤
│  Infrastructure                      │  ← DB, HTTP, FS, external APIs
└──────────────────────────────────────┘
```

**Tell for horizontal violation:** Domain code imports from Infrastructure (e.g. `import { PostgresClient } from "../infrastructure/db"` inside an entity).

---

## The Dependency Rule (most important)

**Source code dependencies point INWARD.**

```
Infrastructure → Application → Domain
      ↓               ↓            ↓
   (outer)        (middle)      (inner)

Outer can import Inner.
Inner CANNOT import Outer.
```

### Operational tells of violation

```typescript
// ❌ — Domain importing Infrastructure
// src/domain/User.ts
import { PostgresClient } from "../infrastructure/db";  // ← VIOLATION

class User {
  save() {
    new PostgresClient().query("INSERT ...");          // ← VIOLATION
  }
}

// ❌ — Domain reaching for global infrastructure
class User {
  log(msg: string) {
    Logger.getInstance().info(msg);                    // ← VIOLATION (singleton)
  }
}

// ❌ — Domain knowing HTTP/DB serialization format
class User {
  toJSON() { return { /* JSON shape */ } }              // ← VIOLATION (presentation concern)
}
```

### How to detect at scale

**ESLint rule:**

```json
{
  "rules": {
    "no-restricted-imports": ["error", {
      "patterns": [
        { "group": ["**/infrastructure/**"], "message": "Domain cannot import infrastructure", "importNames": ["*"] }
      ]
    }]
  }
}
```

Or use `dependency-cruiser` / `madge` to enforce import boundaries in CI.

```bash
npx madge --circular src/  # detect circular deps
npx depcruise --validate .dependency-cruiser.cjs src/
```

### Inverting the dependency

When domain needs something from infra, **define the interface in domain; implement in infra**.

```typescript
// src/domain/UserRepository.ts  ← interface in domain
export interface UserRepository {
  findById(id: UserId): Promise<User | null>;
  save(user: User): Promise<void>;
}

// src/infrastructure/PostgresUserRepository.ts  ← implementation in infra
import { UserRepository } from "../domain/UserRepository";  // ← OK: outer → inner

export class PostgresUserRepository implements UserRepository {
  async findById(id: UserId): Promise<User | null> { /* ... */ }
  async save(user: User): Promise<void> { /* ... */ }
}

// src/main.ts  ← composition root wires the deps
const repo = new PostgresUserRepository(pgClient);
const userService = new UserService(repo);
```

---

## Architectural Styles

### Layered Architecture (traditional)

```
┌─────────────────┐
│  Presentation   │
├─────────────────┤
│  Business       │
├─────────────────┤
│  Persistence    │
└─────────────────┘
```

**Pros:** simple, well-known, easy onboarding.
**Cons:** without discipline, becomes "big ball of mud" — business depends on persistence directly.

**Use when:** small team, simple domain, no need to swap infra.

### Hexagonal Architecture (Ports & Adapters)

```
       ┌────────────────────┐
       │   HTTP Adapter     │
       └─────────┬──────────┘
                 │ Port (interface)
       ┌─────────▼──────────┐
       │      DOMAIN        │
       │   ┌────────────┐   │
       │   │ Use Cases  │   │
       │   └────────────┘   │
       └─────────┬──────────┘
                 │ Port (interface)
       ┌─────────▼──────────┐
       │ Database Adapter   │
       └────────────────────┘
```

- **Ports** are interfaces defined by the domain.
- **Adapters** are implementations that connect to outside (HTTP, DB, queue, etc.).
- Domain knows nothing about how it's reached.

**Use when:** logic-heavy applications; testability is key; you might swap infra.

### Clean Architecture (Bob Martin)

Concentric circles. Same idea as hexagonal, with explicit layer names:

1. **Entities** — enterprise-wide business rules (least likely to change)
2. **Use Cases** — application-specific business rules
3. **Interface Adapters** — controllers, presenters, gateways
4. **Frameworks & Drivers** — web, DB, external interfaces

Dependencies point inward. Outer layers know inner; never the reverse.

### Event-Driven / CQRS / Event Sourcing

Reach for when:
- Multiple write models with different invariants
- Audit / replay requirements (financial, healthcare)
- Different consumers need different read shapes
- Eventual consistency is acceptable

**Don't reach for** these in CRUD apps — overkill.

### Microservices

Service-per-bounded-context. Each owns its data, deploys independently.

**Pre-requisites (mandatory):**
- Mature CI/CD per service
- Distributed tracing
- Service mesh or API gateway
- Operational sophistication (monitoring, alerting, on-call rotation)

**Tell you might be ready:** team size > 30, monolith deploys are bottlenecking, bounded contexts are clear.

**Anti-pattern:** "Microservices for the resume." Splitting prematurely creates a distributed monolith with all the problems and none of the benefits.

---

## Module Structure (Backend)

```
src/
  modules/
    users/
      domain/
        User.ts                    ← entity
        UserId.ts                   ← value object
        UserRepository.ts           ← interface (in domain)
        events/
          UserRegistered.ts         ← domain event
      application/
        CreateUser.ts               ← use case
        GetUser.ts
        UpdateUserEmail.ts
      infrastructure/
        PostgresUserRepository.ts   ← implementation
        UserMapper.ts               ← DB row ↔ entity
      presentation/
        UserController.ts           ← HTTP handler
        UserDTO.ts                  ← request/response shape
      index.ts                      ← module's public API

    orders/
      domain/
      application/
      infrastructure/
      presentation/
      index.ts

  shared/
    domain/                         ← shared value objects (Money, Email, etc.)
    infrastructure/                 ← shared infra (logger, db pool)
```

### Module's public API (`index.ts`)

```typescript
// src/modules/users/index.ts
// Only re-export what other modules can use.
export { User } from "./domain/User";
export { UserId } from "./domain/UserId";
export { CreateUser } from "./application/CreateUser";

// NOTHING from infrastructure or presentation is re-exported.
// Other modules get the use case interface only.
```

**Tell of leaky module:** other modules import from `users/infrastructure/` or `users/presentation/`. Block via lint rule.

---

## Module Structure (Frontend)

```
src/
  features/
    auth/
      components/
        LoginForm.tsx
        SignupForm.tsx
      hooks/
        useAuth.ts
      services/
        authService.ts
      types/
        auth.types.ts
      index.ts                      ← public API of the feature
    checkout/
      components/
      hooks/
      services/
      types/
      index.ts
  shared/
    components/                     ← truly shared UI
    hooks/                          ← truly shared hooks
    utils/                          ← truly shared utilities
```

Same rules: features have public APIs (`index.ts`); other features only consume that.

---

## Cross-Cutting Concerns

Concerns that span features: logging, auth, validation, error handling, observability.

### Options (in order of preference)

1. **Middleware / interceptors** — run on every request, transparent to feature code

```typescript
// HTTP middleware
function loggingMiddleware(req: Request, next: Handler): Response {
  logger.info(`→ ${req.method} ${req.path}`);
  const start = Date.now();
  const res = next(req);
  logger.info(`← ${res.status} (${Date.now() - start}ms)`);
  return res;
}

// Application: chain it once at the boundary
app.use(loggingMiddleware);
```

2. **Decorators** — wrap specific operations

```typescript
function logged(label: string) {
  return function (target: any, key: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    descriptor.value = function (...args: any[]) {
      logger.info(`${label} call`);
      return original.apply(this, args);
    };
  };
}

class OrderService {
  @logged("createOrder")
  create(input: CreateOrderInput) { /* ... */ }
}
```

3. **Aspect-oriented (AOP)** — full-blown, framework-specific (NestJS interceptors, Spring AOP)

4. **Base classes** — sparingly. Prefer composition.

### Anti-pattern: cross-cutting in business logic

```typescript
// ❌ — auth check inline
class OrderService {
  create(input: CreateOrderInput, user: User) {
    if (!user.isAuthenticated) throw new Unauthorized();  // ← cross-cutting
    if (!user.canCreateOrder) throw new Forbidden();      // ← cross-cutting
    // ... actual order creation
  }
}

// ✅ — auth at the boundary; business logic stays clean
@RequiresAuth()
@RequiresPermission("CREATE_ORDER")
class CreateOrderHandler {
  handle(input: CreateOrderInput) { /* business only */ }
}
```

---

## The Walking Skeleton

Start with the **thinnest possible end-to-end slice**:

1. Touches every layer (UI → controller → use case → domain → repository → DB)
2. Deployable from day one (CI/CD pipeline working)
3. Proves the architecture's mechanics

```
Walking skeleton for e-commerce:
- User can view ONE hardcoded product
- User can add it to cart (in-memory)
- User can "checkout" (logs to console)
- App deploys to staging via CI

From here: flesh out features one slice at a time.
```

**Anti-pattern:** building all the infrastructure first ("we'll add features later"). You don't know if your architecture works until a feature flows through it.

---

## Testing Strategy by Layer

```
┌────────────────────────────────────────┐
│  E2E / Acceptance (5%)                 │  Critical paths
├────────────────────────────────────────┤
│  Integration (25%)                     │  Boundaries
├────────────────────────────────────────┤
│  Unit (70%)                            │  Domain logic
└────────────────────────────────────────┘
```

| Layer | Test type | Doubles |
|-------|-----------|---------|
| Domain | Unit (real objects) | None — never mock value objects/entities |
| Application | Integration (with fakes) | Fakes for repos, gateways |
| Infrastructure | Integration (real deps in test instance) | None — verify against real DB/API |
| E2E | Acceptance (full stack) | None — staging environment |

See `testing.md` for details.

---

## Architecture Decision Records (ADRs)

Document significant decisions in `docs/adr/`. One ADR per decision.

```markdown
# ADR-001: Use PostgreSQL for primary persistence

## Status
Accepted (2026-04-15)

## Context
We need a primary data store. Candidates: PostgreSQL, MySQL, MongoDB.

Requirements:
- ACID transactions (financial data)
- JSON support for flexible product attributes
- Mature ecosystem
- Team familiarity

## Decision
PostgreSQL 15.

## Consequences

Positive:
- Strong consistency for orders, payments
- JSONB for product attributes
- Excellent query capabilities
- Team has 5+ years experience

Negative:
- Manual schema migrations needed
- Vertical scaling limit; sharding requires effort
- More complex backup/restore than MongoDB

## Alternatives considered
- MongoDB: rejected — no transactions across docs at the time of decision
- MySQL: rejected — JSON support less mature than Postgres'

## Revisit when
- Write throughput exceeds single-instance capacity
- Multi-region deployment becomes a requirement
```

**Rules:**
- Number sequentially: ADR-001, ADR-002, etc.
- Status: Proposed / Accepted / Deprecated / Superseded
- Never delete an ADR. Mark superseded with link to new ADR.
- Reference ADRs in PRs and code comments when relevant

---

## Red Flags in Architecture

| Symptom | Likely cause |
|---------|--------------|
| Circular dependencies between modules | Boundaries violated; should be one module or use events |
| Domain code imports infrastructure | Dependency rule violation |
| Framework code in business logic | Tight coupling to framework — hard to upgrade |
| No clear boundaries between features | Will become big ball of mud |
| Shared mutable state across modules | Hidden coupling; race conditions |
| "Util" or "Common" packages that grow forever | Missing concept; extract per-domain or kill |
| Database schema driving domain model | DDD inverted — model should drive schema |
| One test instance shared across PRs in CI | Flaky tests; isolate per run |
| Microservices for a 3-person team | Distributed monolith with extra latency |
| Generic "manager" or "service" classes | Names hiding poor SRP |
| Folder structure mirrors layers everywhere | Likely should be feature-first |

---

## Decision Tree — When to Apply Architectural Patterns

### Should I use Hexagonal/Clean Architecture?

```
Is the application primarily CRUD with thin business logic?
├── YES → Layered is enough. Hexagonal is overhead.
└── NO  ↓
    Will I need to swap infrastructure (DB, queue, gateway)?
    ├── YES → Hexagonal — define ports
    └── NO  ↓
        Is testability of domain logic critical (no DB needed)?
        ├── YES → Hexagonal pays off in test speed + clarity
        └── NO  → Layered or modular monolith
```

### Should I use Microservices?

```
Are there multiple bounded contexts with truly different lifecycles?
├── NO → Modular monolith. Move to services later if needed.
└── YES ↓
    Do you have ALL of: distributed tracing, service mesh, mature CI/CD,
    on-call rotation, observability stack?
    ├── NO  → Build operational maturity first. Microservices without ops is suicide.
    └── YES ↓
        Are teams organized around the bounded contexts (Conway's Law)?
        ├── NO  → Reorganize teams first
        └── YES → Proceed
```

### Should I use Event Sourcing / CQRS?

```
Do you have:
- Audit / replay requirements (financial, healthcare, regulated)
- Read models that diverge wildly from write models
- Eventual consistency tolerable
- Team experience with event-driven systems
?

If YES to most → consider it (it's expensive but pays in those contexts).
If NO → don't. Standard repository pattern is far simpler.
```

---

## Conway's Law

> "Organizations design systems that mirror their communication structure."

**Operational implication:** team structure → architecture. Plan both deliberately.

- Two teams that need to constantly coordinate → likely should be one team
- One team owning a giant monolith → likely should be split into multiple teams or modules
- Service boundaries that don't match team boundaries → coordination tax forever

**Inverse Conway maneuver:** if you want a specific architecture, structure teams to match.

---

## Quick Reference

| Concern | Default | Use when |
|---------|---------|----------|
| Module structure | Feature-first | Always |
| Layers | Domain / Application / Infrastructure / Presentation | Logic-heavy apps |
| Dependency direction | Inward (outer → inner) | Always |
| Interfaces in | Domain layer (consumer side) | Always |
| Cross-cutting | Middleware / decorators | Anything that's per-request |
| Module API | `index.ts` re-exports public surface | Always |
| Style | Layered → Hexagonal → Microservices (escalate as needed) | Match to context |
| Walking skeleton | Thinnest end-to-end slice | Day 1 of any project |
| ADR | Document significant decisions | Architecture, framework, DB, deployment |
| Tools | ESLint import rules, dependency-cruiser, madge | CI enforcement |
