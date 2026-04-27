# Working with Legacy Code

> **Legacy code = code without tests** (Michael Feathers' definition).

When you need to change code that has no tests, the standard SOLID/refactor playbook DOESN'T apply directly. You can't refactor safely without tests; you can't easily add tests because the code is tangled. This file is the workflow for that situation.

## The Legacy Code Dilemma

```
Code change requested.
├── Tests exist? → Standard TDD/refactor cycle. Stop reading this file.
└── No tests ↓
    Do I REALLY need to change this?
    ├── NO  → Don't. Code that works can stay as-is.
    └── YES ↓
        Can I make the change WITHOUT modifying the existing code?
        ├── YES → Sprout Method / Sprout Class / Wrap Method
        └── NO  → Add characterization tests first, THEN refactor + change
```

---

## Decision Matrix — Legacy Workflow

| Situation | Approach |
|-----------|----------|
| Add new behavior, can sit alongside existing | **Sprout Method / Sprout Class** |
| Need to wrap existing behavior with new logic | **Wrap Method / Wrap Class** |
| Need to modify existing logic, untested | **Characterization tests + then refactor** |
| Existing code reaches into globals/singletons | **Extract via Seam** |
| Need to test code that calls untestable code | **Find seam** to insert test double |
| Massive untested codebase | **Hot spots first** — refactor only files you actively change |

---

## Characterization Tests

**Tests that pin down what the code CURRENTLY does**, not what it SHOULD do. Buy you a safety net.

### Steps

1. Find a piece of code you need to change.
2. Identify input → output behavior at the boundaries.
3. Write a test that asserts the **current actual** behavior.
4. Run it — should pass (we're capturing reality).
5. Now the code is "tested" enough to refactor.

### Example

```typescript
// Legacy function — you need to add a feature
function calculateTax(price: number, customer: any): number {
  let rate = 0.07;
  if (customer.country === "US" && customer.state === "CA") rate = 0.0875;
  if (customer.country === "BR") rate = 0.18;
  if (customer.exemptFromTax) rate = 0;
  return price * rate;
}

// Step 1: characterization tests pin down current behavior
describe("calculateTax (characterization)", () => {
  it("returns 7% for non-US, non-BR customer", () => {
    expect(calculateTax(100, { country: "MX" })).toBeCloseTo(7);
  });

  it("returns 8.75% for California customer", () => {
    expect(calculateTax(100, { country: "US", state: "CA" })).toBeCloseTo(8.75);
  });

  it("returns 18% for Brazil customer", () => {
    expect(calculateTax(100, { country: "BR" })).toBeCloseTo(18);
  });

  it("returns 0 for tax-exempt customer regardless of location", () => {
    expect(calculateTax(100, { country: "BR", exemptFromTax: true })).toBe(0);
  });
});

// Now you can refactor with confidence — tests catch regressions
```

### Characterization tests vs unit tests

- **Characterization:** tests existing behavior, even if it's wrong (the bug becomes "the spec for now")
- **Unit tests:** tests intended behavior

After refactoring + understanding the code, characterization tests can become proper unit tests (with intentional, named cases).

---

## Seams

A **seam** is a place where you can change behavior **without editing that location**. Required for inserting test doubles into untestable code.

### Types of Seams

#### 1. Object Seam (most common in OO)

Replace dependency at runtime.

```typescript
// ❌ — no seam: hardcoded dependency
class OrderService {
  process() {
    const db = new PostgresClient(); // can't replace in test
    db.save(/* ... */);
  }
}

// ✅ — Object Seam via constructor injection
class OrderService {
  constructor(private readonly db: Database) {}
  process() { this.db.save(/* ... */); }
}

// In test:
const fakeDb = new InMemoryDatabase();
const service = new OrderService(fakeDb);
```

#### 2. Preprocessing Seam (compile-time)

C/C++ macros, conditional compilation. Less common in modern languages.

#### 3. Link Seam (build-time)

Replace dependency at build/link time. In JS/TS, this is module replacement (`vi.mock()`).

```typescript
// In test:
vi.mock("./db", () => ({ saveToDatabase: vi.fn() }));
```

**Use sparingly.** Module mocks couple tests to file paths, not to interfaces.

### Adding a seam where there isn't one

The point of refactoring legacy code is **inserting seams** so you can add tests.

```typescript
// Step 1: find a hardcoded dep
class OrderService {
  process() {
    Logger.getInstance().info("processing");  // ← hardcoded singleton
    new Database().save(/* ... */);            // ← hardcoded class
  }
}

// Step 2: introduce parameter (object seam) without changing behavior
class OrderService {
  process(logger: Logger = Logger.getInstance(), db: Database = new Database()) {
    logger.info("processing");
    db.save(/* ... */);
  }
}

// Step 3: now you can test
const fakeLogger = new FakeLogger();
const fakeDb = new FakeDb();
new OrderService().process(fakeLogger, fakeDb);

// Step 4 (later, when more confident): convert to constructor injection
class OrderService {
  constructor(
    private readonly logger: Logger,
    private readonly db: Database,
  ) {}
  process() {
    this.logger.info("processing");
    this.db.save(/* ... */);
  }
}
```

---

## Sprout Method / Sprout Class

**When you need to add new behavior, SPROUT it as a new method or class — don't modify existing untested code.**

### Sprout Method

Add the new behavior as a new method on the same class. Call it from the existing method.

```typescript
// Legacy untested method
class OrderProcessor {
  process(order: Order) {
    // 50 lines of untested logic
    /* ... */
    saveOrder(order);
    /* ... */
  }
}

// You need to add: send a confirmation email after save.
// ❌ Don't modify the 50 lines.
// ✅ Sprout — add a new method:

class OrderProcessor {
  process(order: Order) {
    /* ... 50 untouched lines ... */
    saveOrder(order);
    this.notifyConfirmation(order);  // ← sprouted
    /* ... */
  }

  // NEW METHOD — fully tested via TDD
  private notifyConfirmation(order: Order): void {
    if (!order.customer.wantsNotifications) return;
    this.emailService.send(order.customer.email, "Order confirmed");
  }
}
```

The legacy method gains one line; the new method is fully tested.

### Sprout Class

When the new behavior is bigger, sprout an entire class.

```typescript
class OrderProcessor {
  constructor(private readonly fraudCheck: FraudCheck = new FraudCheck()) {}  // sprouted

  process(order: Order) {
    /* 50 untouched lines */
    this.fraudCheck.evaluate(order);  // ← one new line
    /* ... */
  }
}

// New class — fully tested
class FraudCheck {
  evaluate(order: Order): FraudVerdict {
    if (order.total.greaterThan(Money.dollars(10_000))) return FraudVerdict.suspicious();
    if (order.customer.age().lessThan(Days.of(7))) return FraudVerdict.suspicious();
    return FraudVerdict.clean();
  }
}
```

The legacy method is barely touched; the new class is greenfield.

---

## Wrap Method / Wrap Class

**When you need to wrap existing behavior with new logic.**

### Wrap Method

Rename the original method, create a new one with the original name that calls the renamed one + adds the wrapping behavior.

```typescript
// Original
class PayrollProcessor {
  payDay(employee: Employee): void {
    // legacy untested logic
  }
}

// You need to: log every payday execution.

// Step 1: rename original to internal
class PayrollProcessor {
  private payDayInternal(employee: Employee): void {
    // legacy logic untouched
  }

  // Step 2: new public method wraps + adds behavior
  payDay(employee: Employee): void {
    this.payDayInternal(employee);
    this.logger.info(`Paid ${employee.id}`);   // ← new behavior, tested via TDD
  }
}
```

External callers see the same public API; the new logic is testable.

### Wrap Class

The original class becomes wrapped by a new class implementing the same interface.

```typescript
// Original
class StripeGateway implements PaymentGateway {
  charge(amount: Money): ChargeResult { /* legacy untested */ }
}

// You need to: add retry logic without modifying StripeGateway.

class RetryingPaymentGateway implements PaymentGateway {
  constructor(
    private readonly underlying: PaymentGateway,
    private readonly maxAttempts: number = 3,
  ) {}

  charge(amount: Money): ChargeResult {
    for (let i = 0; i < this.maxAttempts; i++) {
      const result = this.underlying.charge(amount);
      if (result.isSuccess()) return result;
    }
    return ChargeResult.failed("max retries");
  }
}

// At composition root: wrap
const gateway = new RetryingPaymentGateway(new StripeGateway());
```

This is the Decorator pattern, applied as a refactoring move.

---

## Edit-and-Pray vs Cover-and-Modify

Two strategies when faced with legacy code that needs change.

### Edit-and-Pray (default in legacy environments)

> "Make the change, ship it, hope nothing breaks."

This is what most people do. It's how legacy code stays legacy: every change adds risk and complexity without adding safety.

### Cover-and-Modify (the discipline)

```
1. Identify the change point
2. Find a seam (insert one if needed)
3. Write characterization tests for the affected behavior
4. Refactor (with tests as safety net)
5. Add the new behavior with TDD
6. Tests stay forever — code is now less legacy
```

**Trade-off:** slower per change, but the codebase **gets healthier** with every change instead of decaying.

---

## Strangler Fig Pattern

For replacing legacy systems gradually.

```
1. Build new code alongside the old (don't replace yet)
2. Route traffic / calls to new code for one specific use case
3. Verify behavior matches; expand to more use cases
4. When 100% of traffic flows to new, retire old
```

```typescript
// Old legacy controller
class LegacyOrderController {
  create(req: Request): Response { /* old code */ }
}

// New controller built alongside
class OrderController {
  create(req: Request): Response { /* new clean code */ }
}

// Routing: gradually shift traffic
function route(req: Request): Response {
  if (FeatureFlags.useNewOrderFlow(req.user)) {
    return new OrderController().create(req);
  }
  return new LegacyOrderController().create(req);
}
```

The old code is "strangled" — kept alive in shrinking scope until safely removable.

**When to use:** big legacy modules that can't be incrementally refactored. Build the replacement, gradually shift, retire old.

---

## Effective Order of Operations

When you encounter legacy code, follow this order:

```
1. Don't change it if you don't have to
2. Sprout / Wrap if you can add without modifying
3. Identify the change point exactly
4. Find or insert a seam
5. Write characterization tests
6. Refactor surgically (smallest possible change)
7. Make the actual change with TDD
8. Run all tests
9. Commit small steps (easy to revert)
```

Each step is a checkpoint. If any fails, back up — don't push through.

---

## Specific Techniques

### Extract Interface (to introduce a seam)

When you need to test a class that depends on a hard-to-fake collaborator.

```typescript
// Before — depends on concrete class with side effects
class Reporter {
  generateReport(): string {
    const db = new Database();  // can't fake
    return db.query("SELECT ...");
  }
}

// Step 1: Extract Interface
interface DatabaseReader {
  query(sql: string): any[];
}

class Database implements DatabaseReader {
  query(sql: string): any[] { /* real impl */ }
}

// Step 2: parameterize
class Reporter {
  constructor(private readonly db: DatabaseReader) {}
  generateReport(): string {
    return this.db.query("SELECT ...");
  }
}

// Step 3: test with fake
const fakeDb: DatabaseReader = { query: () => [{ id: 1 }] };
const reporter = new Reporter(fakeDb);
```

### Subclass and Override (when extraction is too risky)

When you can't restructure but need to override one method.

```typescript
// Legacy class you can't restructure
class LegacyOrderService {
  process(order: Order) {
    /* 200 lines */
    const result = this.callExternalAPI(order);  // need to fake this
    /* ... */
  }

  protected callExternalAPI(order: Order): Result { /* real impl */ }
}

// In test: subclass to override the one method
class TestableOrderService extends LegacyOrderService {
  protected callExternalAPI(order: Order): Result {
    return Result.success({ id: "fake" });  // fake response
  }
}

// Test
const service = new TestableOrderService();
service.process(order);
```

Risky because it relies on subclassing; use only when you can't refactor more cleanly.

### Pull Up / Push Down

Move methods up or down a hierarchy to introduce / remove abstraction.

### Replace Static Method

Convert static method calls into method calls on an injectable object.

```typescript
// Before
class Validator {
  validate(email: string): boolean {
    return EmailUtils.isValid(email);  // static call
  }
}

// After
class EmailValidator {
  isValid(email: string): boolean { /* impl */ }
}

class Validator {
  constructor(private readonly emailValidator: EmailValidator) {}
  validate(email: string): boolean {
    return this.emailValidator.isValid(email);
  }
}
```

---

## Common Legacy Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Refactoring without tests first | Tests pass, prod breaks | ALWAYS characterization tests first |
| Big-bang rewrite | Project never finishes | Strangler fig pattern |
| "I'll add tests later" | Never happens | Add tests with the change |
| Breaking encapsulation to test | Public methods exposed for testing only | Find proper seam; don't widen API |
| Mocking what you don't own | Tests break on lib upgrade | Wrap third-party in your own interface |
| Touching too much in one change | Hard to review, hard to revert | Smallest possible step; many commits |
| Ignoring the rules of the existing code | Style drift | Match existing conventions until the area is fully refactored |

---

## When to Stop

Legacy refactoring has diminishing returns. Stop when:

- The change you needed is shipped
- The code you touched has tests
- The boundary you crossed has a seam
- The next file is unrelated to your work

**Don't try to fix the whole codebase.** Pay attention to **hot spots** (files that change often). Refactor those incrementally; leave cold files alone.

---

## Quick Reference

| Situation | Move |
|-----------|------|
| Add new behavior, untested area | **Sprout Method / Sprout Class** |
| Wrap existing behavior | **Wrap Method / Wrap Class** |
| Need to test untestable code | **Find / insert a seam** |
| Need to modify untested logic | **Characterization tests** → refactor → modify |
| Hardcoded dependency blocks testing | **Extract Interface + inject** |
| Static call blocks testing | **Replace Static Method** |
| Big system to replace | **Strangler Fig Pattern** |
| Untouched legacy code | **Leave it.** Don't refactor for fun. |

### The seam types (most useful first)

| Seam | Mechanism |
|------|-----------|
| Object | Constructor injection / parameter |
| Subclass + Override | Inherit and override the testable method |
| Link | Module mock (`vi.mock`) — last resort |

### The legacy mantra

> "I will not change a line of untested code without first writing a test."

Discipline at the gate — every change becomes safer. Eventually, the codebase isn't legacy anymore.
