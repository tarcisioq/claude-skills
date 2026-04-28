# Cohesion and Coupling

The two metrics that determine whether a codebase scales. **High cohesion + low coupling = healthy.** This file operationalizes both.

## Decision Matrix

| Question | Direction |
|----------|-----------|
| What's IN this module/class? Should it stay together? | **Cohesion** — measure internal relatedness |
| Who DEPENDS on what? Are dependencies minimal and stable? | **Coupling** — measure external connections |
| Code feels brittle, ripple effects on changes | Low cohesion OR high coupling |
| Code feels stable, changes localize | High cohesion AND low coupling |

**Rule:** when refactoring, you're trying to **increase cohesion** (group related things) and **decrease coupling** (reduce dependencies between things that change for different reasons).

---

## Cohesion Levels (highest is best)

From most-cohesive to least-cohesive. Strive for **functional**; tolerate **sequential**; refactor if **temporal** or below.

### 1. Functional Cohesion (BEST)

All elements contribute to a single, well-defined task.

**Tell:** the module/class name describes what it does in 3-5 words and you don't need "and".

```typescript
// ✅ Functional — every method serves "validate an email"
class EmailValidator {
  validate(email: string): ValidationResult { /* ... */ }
  private hasAtSymbol(email: string): boolean { /* ... */ }
  private hasValidDomain(email: string): boolean { /* ... */ }
  private isWithinLengthLimits(email: string): boolean { /* ... */ }
}
```

### 2. Sequential Cohesion

Elements work on the same data, output of one is input of next.

**Tell:** linear processing pipeline.

```typescript
// ✅ Sequential — each step transforms the output of the previous
function processOrder(rawData: string): ProcessedOrder {
  const parsed = parseOrder(rawData);          // raw → parsed
  const validated = validateOrder(parsed);     // parsed → validated
  const enriched = enrichWithCustomer(validated); // validated → enriched
  return enriched;
}
```

Acceptable. Could be refactored into a more cohesive class if the pipeline is reused.

### 3. Communicational Cohesion

Elements operate on the same data structure but for different reasons.

**Tell:** shared data structure; different operations.

```typescript
// ✓ Communicational — all operate on Order data
class OrderQueries {
  totalRevenue(orders: Order[]): Money { /* ... */ }
  averageOrderValue(orders: Order[]): Money { /* ... */ }
  ordersByStatus(orders: Order[]): Map<OrderStatus, Order[]> { /* ... */ }
}
```

OK in some cases (read models, query helpers). Less cohesive than functional but still acceptable.

### 4. Procedural Cohesion

Elements are part of a procedure where the order matters.

**Tell:** "do this, then this, then this" — but the operations don't share data.

```typescript
// ⚠ Procedural — order matters but the steps are unrelated
function startOfDay() {
  resetCaches();         // unrelated to logging
  rotateLogFiles();      // unrelated to retries
  retryFailedJobs();     // unrelated to caches
}
```

Suspect. Often a symptom that scheduling/orchestration is conflated with multiple unrelated concerns. Split.

### 5. Temporal Cohesion

Elements are grouped because they execute at the same time.

**Tell:** initialization or shutdown procedures with unrelated steps.

```typescript
// ⚠ Temporal — only relation is "happens at startup"
class Startup {
  init() {
    connectDatabase();
    loadConfiguration();
    startMetricsServer();
    initializeCache();
    registerSignalHandlers();
  }
}
```

Common in `main.ts` or app initialization — acceptable there. Avoid creating temporal-cohesion classes elsewhere.

### 6. Logical Cohesion

Elements are grouped because they do similar kinds of things.

**Tell:** big switch or mixed responsibilities behind one method name.

```typescript
// ❌ Logical — methods loosely "input" but for different sources
class Input {
  read(source: string): Data {
    if (source === "file") return readFile();
    if (source === "network") return readNetwork();
    if (source === "database") return readDb();
  }
}
```

Refactor — replace with polymorphism.

### 7. Coincidental Cohesion (WORST)

Elements have no relation at all; grouped randomly.

**Tell:** the class name is `Utilities`, `Helpers`, `Common`, `Misc`. Methods don't share state or domain.

```typescript
// ❌ Coincidental — "utilities" hides the lack of cohesion
class Utilities {
  formatDate(d: Date): string { /* ... */ }
  hashPassword(p: string): string { /* ... */ }
  sendEmail(to: string, body: string): void { /* ... */ }
  parseCSV(s: string): string[][] { /* ... */ }
}
```

**Always refactor.** Split into focused modules:
- `DateFormatter` (or part of `Date` value object)
- `PasswordHasher`
- `EmailSender`
- `CSVParser`

---

## Cohesion Decision Tree

```
Look at the methods of a class.
├── All serve one task? → Functional ✅
├── Linear pipeline (output → input)? → Sequential ✓
├── Operate on same data, different purposes? → Communicational ✓
├── Sequential but unrelated data? → Procedural ⚠
├── Run together, otherwise unrelated? → Temporal ⚠
├── Do similar-shaped things over different inputs? → Logical ❌ (refactor)
└── No relation at all? → Coincidental ❌ (always split)
```

---

## Coupling Levels (lowest is best)

From least-coupled to most-coupled.

### 1. No Coupling (independent modules)

Best, when achievable. Modules don't know each other exists.

**Achieved by:** event-driven architecture, plugin systems, separate services with no shared state.

### 2. Data Coupling (BEST realistic)

Modules communicate via parameters that are simple data (numbers, strings, value objects).

**Tell:** functions take only the data they need; no shared structures.

```typescript
// ✅ Data — only the price is passed, not the whole order
function applyTax(price: Money): Money { /* ... */ }
```

### 3. Stamp Coupling

Modules share a structure (e.g. an object) but use only some of its fields.

**Tell:** function takes a big object but uses 1-2 fields.

```typescript
// ⚠ Stamp — function takes Order but only uses customer.country
function shippingCost(order: Order): Money {
  return order.customer.country === "US" ? Money.dollars(5) : Money.dollars(15);
}
```

Refactor to data coupling:

```typescript
// ✅ Data
function shippingCost(country: Country): Money {
  return country.equals(Country.US) ? Money.dollars(5) : Money.dollars(15);
}
```

### 4. Control Coupling

One module controls the flow of another via flags.

**Tell:** boolean parameter that switches behavior.

```typescript
// ⚠ Control coupling
function process(data: Data, isSpecial: boolean) {
  if (isSpecial) { /* ... */ } else { /* ... */ }
}
```

Refactor — split into named methods or use polymorphism.

### 5. External Coupling

Modules share an external format (file, protocol, database schema).

**Tell:** changing the external format breaks multiple modules.

```typescript
// ⚠ External — every module reads/writes the same JSON schema
class OrderReader { read(): Order { /* parse JSON */ } }
class OrderWriter { write(o: Order): void { /* serialize JSON */ } }
class OrderValidator { validate(json: any): boolean { /* ... */ } }
```

Mitigate by encapsulating the format in one place (single mapper).

### 6. Common Coupling

Modules share global mutable state.

**Tell:** singletons holding mutable state, global variables.

```typescript
// ❌ Common — multiple modules read/write the same singleton
class Config {
  static settings: any = {};
}

class A { useFeature() { return Config.settings.feature; } }
class B { setFeature() { Config.settings.feature = true; } }
```

Refactor — inject dependencies, eliminate global mutable state.

### 7. Content Coupling (WORST)

One module modifies the internals of another.

**Tell:** reaching into private state, monkey-patching, prototype mutation.

```typescript
// ❌ Content — modifies another class's internal state
class A {
  private items: Item[] = [];
}

class B {
  doSomething(a: A) {
    (a as any).items.push(...);  // ❌
  }
}
```

Always refactor. Pure violation of encapsulation.

---

## Coupling Metrics — Numerical

### Afferent Coupling (Ca)

Number of classes that depend on THIS class. ("How many people care if I change?")

- High Ca → many things depend on you → changes ripple widely → be very careful and stable
- Low Ca → few depend on you → free to change

### Efferent Coupling (Ce)

Number of classes THIS class depends on. ("How many things break me if they change?")

- High Ce → fragile, sensitive to upstream changes
- Low Ce → independent, robust

### Instability (I)

```
I = Ce / (Ca + Ce)
```

| I | Meaning |
|---|---------|
| 0.0 | Maximally stable — many things depend on you, you depend on nothing. Good for abstractions/interfaces. |
| 1.0 | Maximally unstable — depends on many, nothing depends on you. Good for concrete implementations / leaves. |

### Ideal pattern

Stable abstractions, unstable concretes. Domain interfaces should have I close to 0; concrete adapters/UI should have I close to 1.

```
Domain interface:    I = 0.1  (stable, abstract)
Use case impl:       I = 0.5  (medium)
HTTP adapter:        I = 0.9  (unstable, concrete)
```

If a concrete leaf has I = 0.2, something's depending on it that shouldn't be. Refactor.

### Tools

```bash
# JS/TS — dependency-cruiser
npx depcruise --output-type metrics src/

# Java — JDepend
# .NET — NDepend
```

---

## How Cohesion and Coupling Interact

```
                  Coupling
              Low          High
            ┌──────────┬──────────┐
       High │  IDEAL   │ FRAGILE  │  ← high cohesion + high coupling
Cohesion    │          │          │     = stable BUT brittle
            ├──────────┼──────────┤
        Low │ ROTTING  │  CHAOS   │  ← god classes, big balls of mud
            │          │          │
            └──────────┴──────────┘
```

Goal: **top-left**. Each module is internally cohesive (does one thing well) and externally minimally coupled (doesn't reach into others).

---

## Operational Refactoring for Cohesion / Coupling

### Increase cohesion

| Symptom | Action |
|---------|--------|
| Class with `Utilities` / `Helpers` name | Split per concern (`DateFormatter`, `EmailSender`, etc.) |
| Class with > 2 verb domains | Extract Class per domain |
| Methods grouped only by "happen at startup" | Move to composition root, not a class |
| Mixed read/write/transform on same data | Consider CQRS-style split if it helps |

### Decrease coupling

| Symptom | Action |
|---------|--------|
| Function takes `Order` but uses `order.customer.country` | Pass `Country` directly |
| Boolean flag changes behavior | Split into named methods or polymorphism |
| Singleton holds mutable state | Inject as dependency, make immutable |
| Class reaches into another's private state | Stop. Use methods. |
| Global config object accessed everywhere | Pass via constructor; immutable Config |
| Classes share an external format | Encapsulate format in one mapper class |

---

## Connascence (advanced model)

A more nuanced model: how much do two pieces of code need to change together?

### Static (compile-time) connascence — easier to manage

| Type | Example | Strength |
|------|---------|----------|
| Connascence of Name | Two places use `userId` | Weak |
| Connascence of Type | Two places agree on type | Weak |
| Connascence of Meaning | Magic value `1` means "active" | Strong |
| Connascence of Position | Two callers depend on parameter order | Strong |
| Connascence of Algorithm | Two pieces implement same hash | Strong |

### Dynamic (runtime) connascence — harder

| Type | Example | Strength |
|------|---------|----------|
| Connascence of Execution | Order of calls matters (`init` before `start`) | Strong |
| Connascence of Timing | Timeout in one piece must match another | Stronger |
| Connascence of Value | Two values must be consistent (start ≤ end) | Stronger |
| Connascence of Identity | Two refs must point to the same object | Strongest |

### Operational rules

1. **Stronger connascence → encapsulate in smaller scope.** Connascence of Identity inside one class is fine; across modules is a bug.
2. **Convert dynamic to static when possible.** Compiler can catch static; runtime can't catch dynamic.
3. **Reduce strength.** Magic number (Meaning) → Named constant (Name).

```typescript
// ❌ Connascence of Meaning (1 means "active")
if (user.status === 1) { /* ... */ }

// Convert to Connascence of Name (still some connascence, but weaker)
const ACTIVE = 1;
if (user.status === ACTIVE) { /* ... */ }

// Even better: type-safe (compiler enforces)
class UserStatus {
  static ACTIVE = new UserStatus("active");
  static SUSPENDED = new UserStatus("suspended");
}
if (user.status.equals(UserStatus.ACTIVE)) { /* ... */ }
```

---

## Quick Reference

### Cohesion levels (highest = best)

```
Functional > Sequential > Communicational > Procedural > Temporal > Logical > Coincidental
   ✅           ✓              ✓               ⚠           ⚠           ❌          ❌
```

### Coupling levels (lowest = best)

```
None > Data > Stamp > Control > External > Common > Content
 ✅    ✅      ⚠       ⚠         ⚠          ❌        ❌
```

### Refactoring targets

| Observation | Action |
|-------------|--------|
| `Utilities` / `Helpers` class | Coincidental cohesion → split |
| Function takes object, uses 1 field | Stamp → Data |
| Boolean flag parameter | Control → split or polymorphism |
| Singleton with mutable state | Common → DI |
| Class reaches into private state | Content → encapsulate |
| Magic numbers | Connascence of Meaning → Name |

### Metrics to track

- **Instability (I)** — should be 0 for abstractions, 1 for concretes
- **Afferent coupling (Ca)** — high for stable interfaces; low for leaves
- **Efferent coupling (Ce)** — keep low for any single class
- Tool: `dependency-cruiser`, `madge`, language-specific equivalents

### Decision principle

**High cohesion within → low coupling between.** If you can't have both, prioritize cohesion (one focused thing) over zero coupling (some communication is fine if it's clean).
