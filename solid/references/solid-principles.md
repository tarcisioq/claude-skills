# SOLID Principles

Five principles for OO design. Operationalized with **mechanical tells** — not "feel". If the tell fires, apply the principle.

## Decision Matrix — Apply Before Designing

| Question | Principle | Trigger / Tell |
|----------|-----------|----------------|
| Does this class have one reason to change? | **SRP** | Methods span 2+ verb domains; class > 50 lines; multiple stakeholders |
| Will adding new behavior require editing existing code? | **OCP** | `if/else` chain or `switch` on type discriminator |
| Can subclasses replace the parent without breaking callers? | **LSP** | Need `instanceof` checks; subclass throws on parent methods |
| Are clients forced to depend on methods they don't use? | **ISP** | Empty implementations; `throw new Error("not implemented")` |
| Does business logic depend on infrastructure details? | **DIP** | `new ConcreteX()` inside business class; direct DB/HTTP calls |

---

## S — Single Responsibility Principle (SRP)

> "A class should have one, and only one, reason to change."

### When to apply (mechanical tells)

Split when **any** of these fire:

- Class > 50 lines
- Class has > 2 instance variables
- Class has > 5 public methods
- Methods span 2+ verb domains (e.g. `save*` AND `validate*` AND `format*`)
- You can't describe the class without using "and"
- Two different stakeholders would request changes to different parts

### Decision tree — find the seams

```
Look at the class's methods and instance variables.
├── Group methods that share state (same instance vars)
├── Group methods that share a verb domain (save/load = persistence; format/render = presentation)
└── Each group becomes a separate class.
```

### Example progression

```typescript
// Step 1 — start (acceptable while < 50 lines)
class Order {
  private items: OrderItem[] = [];
  addItem(item: OrderItem): void { this.items.push(item); }
  calculateTotal(): Money { /* ... */ }
}

// Step 2 — grew. Now does persistence + presentation. Multiple "and"s.
class Order {
  addItem() { } calculateTotal() { }
  saveToDatabase() { }     // ← persistence
  generateInvoice() { }    // ← presentation
  sendConfirmation() { }   // ← notification
}

// Step 3 — split per verb domain
class Order {
  addItem() { } calculateTotal() { }
}
class OrderRepository { save(order: Order): Promise<void> { } }
class InvoiceGenerator { generate(order: Order): Invoice { } }
class OrderNotifier { sendConfirmation(order: Order): void { } }
```

### When NOT to split

- Value objects (immutable, behaves as one concept — `Money`, `Email`)
- Result-type wrappers (`OrderResult`, `WithdrawResult`)
- Aggregate roots that legitimately encapsulate behavior over a cluster
- Classes already < 30 lines doing one obvious thing

### Common SRP mistakes

- **Splitting too early** — classes with 1 method aren't a goal. Wait until the smell fires.
- **Confusing "responsibility" with "behavior"** — a class can have many methods if they all serve one responsibility (e.g. `Order.addItem`, `Order.removeItem`, `Order.calculateTotal` all serve "manage order state").
- **Treating SRP as "do one thing"** — it's "have one reason to change". A class can do multiple things that change for the same reason.

---

## O — Open/Closed Principle (OCP)

> "Software entities should be open for extension but closed for modification."

### When to apply (mechanical tells)

- You see `if/else` chain or `switch` on a `type`/`kind`/`category` field
- Adding a new variant requires editing N files (change amplification)
- Tests in the existing class break when you add a new variant
- Comments like "// add new payment methods here"

### Operationalized

```typescript
// ❌ — every new shipping method = edit this function
function calculateShipping(type: string, value: number): number {
  if (type === "standard") return value < 50 ? 5 : 0;
  if (type === "express") return 15;
  if (type === "overnight") return 25;
  // To add "same-day" → modify here
  throw new Error(`Unknown: ${type}`);
}

// ✅ — new shipping = new class. Existing code untouched.
interface ShippingMethod {
  cost(orderValue: Money): Money;
}

class StandardShipping implements ShippingMethod {
  cost(value: Money): Money { return value.lessThan(Money.of(50)) ? Money.of(5) : Money.of(0); }
}
class ExpressShipping implements ShippingMethod {
  cost(): Money { return Money.of(15); }
}
class SameDayShipping implements ShippingMethod {
  cost(): Money { return Money.of(40); }
}

// Caller code is unchanged — just gets a different ShippingMethod
function calculateShipping(method: ShippingMethod, value: Money): Money {
  return method.cost(value);
}
```

### Decision tree — abstract or not?

```
How often do you add a new variant?
├── 1-2 known variants, no expected growth → Don't abstract. KISS.
├── Variants from a stable bounded set (3-5, won't change) → Polymorphism if it simplifies; OK to keep switch otherwise
└── Open-ended variant set (plugins, payment methods, file formats) → Polymorphism mandatory
```

### When NOT to apply

- Two-variant cases where the abstraction adds more code than it removes
- Truly fixed enumerations (days of week, HTTP methods) — embrace the switch
- Pre-emptive interfaces "in case we need more" — speculative generality

### OCP at architecture level

```
New feature → new module / new file / new class
NEVER → "edit existing tested code to add a feature"
```

This is what makes a codebase scale to large teams.

---

## L — Liskov Substitution Principle (LSP)

> "Subtypes must be substitutable for their base types without altering correctness."

### When to apply (mechanical tells)

- Subclass throws on a parent method (`throw new Error("not supported")`)
- Caller has `if (x instanceof Subclass)` checks
- Subclass narrows preconditions or widens postconditions
- Subclass behavior surprises callers expecting parent's contract

### Operationalized — the substitution test

For each subclass, verify:

| Aspect | Rule |
|--------|------|
| **Preconditions** | Subclass can require LESS (or equal), never MORE than parent |
| **Postconditions** | Subclass can guarantee MORE (or equal), never LESS than parent |
| **Invariants** | Subclass must preserve all parent invariants |
| **Exceptions** | Subclass can throw same or narrower types than parent declares |
| **Side effects** | Subclass cannot introduce side effects parent doesn't have |

### Example violation

```typescript
// Parent contract: returns a non-negative number
class DiscountPolicy {
  getDiscount(amount: Money): Money {
    return Money.zero();
  }
}

// ❌ — violates: returns negative (charges extra)
class WeirdDiscount extends DiscountPolicy {
  getDiscount(amount: Money): Money {
    return Money.of(-5); // Breaks every caller assuming non-negative
  }
}

// ✅ — enforce contract via construction; LSP-safe by design
class DiscountPolicy {
  constructor(private readonly value: NonNegativeMoney) {}
  getDiscount(): Money { return this.value; }
}
```

### Square / Rectangle classic

```typescript
// ❌ — Square IS-A Rectangle? Algebraically yes. Behaviorally no.
class Rectangle {
  constructor(public width: number, public height: number) {}
  setWidth(w: number) { this.width = w; }
  setHeight(h: number) { this.height = h; }
  area(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  setWidth(w: number) { this.width = w; this.height = w; }   // Surprises caller
  setHeight(h: number) { this.width = h; this.height = h; }
}

function expandTo10x20(r: Rectangle) {
  r.setWidth(10);
  r.setHeight(20);
  console.assert(r.area() === 200); // Fails when r is Square
}
```

**Fix:** don't inherit. Square and Rectangle are different shapes — both implement `Shape`.

### Decision: should this be a subclass?

```
Does the subclass strengthen any precondition? → NOT LSP-safe
Does it weaken any postcondition?              → NOT LSP-safe
Does it throw where parent doesn't?             → NOT LSP-safe
Does caller need instanceof / type check?      → NOT LSP-safe
Otherwise → LSP-safe
```

If not LSP-safe, **don't inherit**. Use composition (delegate to parent's class as a field).

---

## I — Interface Segregation Principle (ISP)

> "Clients should not be forced to depend on methods they do not use."

### When to apply (mechanical tells)

- Implementations have empty methods or `throw new Error("not implemented")`
- Different clients use disjoint subsets of a class's methods
- Interface has > 5 methods that aren't tightly cohesive

### Operationalized

```typescript
// ❌ — fat interface; printer is forced to "implement" scanner methods
interface WarehouseDevice {
  printLabel(orderId: OrderId): void;
  scanBarcode(): Barcode;
  packageItem(orderId: OrderId): void;
}

class BasicPrinter implements WarehouseDevice {
  printLabel(orderId: OrderId): void { /* works */ }
  scanBarcode(): Barcode { throw new Error("not supported"); }      // ❌
  packageItem(orderId: OrderId): void { throw new Error("not supported"); } // ❌
}

// ✅ — segregated interfaces; printer implements only what it does
interface LabelPrinter { printLabel(orderId: OrderId): void; }
interface BarcodeScanner { scanBarcode(): Barcode; }
interface ItemPackager { packageItem(orderId: OrderId): void; }

class BasicPrinter implements LabelPrinter {
  printLabel(orderId: OrderId): void { /* only what it does */ }
}
class MultiDevice implements LabelPrinter, BarcodeScanner, ItemPackager { /* implements all */ }
```

### Decision tree — split an interface?

```
Do all clients use all methods?
├── YES → Keep one interface
└── NO  ↓
    Are the unused methods grouped (multiple clients ignore the same subset)?
    ├── YES → Split along that boundary
    └── NO  → Roles too entangled — review the abstraction itself
```

### Role interfaces

ISP often produces small "role interfaces" — a single method per interface in extreme cases.

```typescript
// Tiny role interfaces compose into capabilities
interface CanReadUsers  { findUser(id: UserId): Promise<User | null>; }
interface CanWriteUsers { saveUser(user: User): Promise<void>; }

// Concrete implementation has all roles
class UserRepository implements CanReadUsers, CanWriteUsers { /* ... */ }

// Read-only consumer takes only what it needs
class UserDisplayService {
  constructor(private readonly users: CanReadUsers) {} // can't accidentally write
}
```

### When NOT to apply

- Cohesive interfaces where every method is used by every client
- Genuine "kit" interfaces (data store with CRUD) where clients use most operations

---

## D — Dependency Inversion Principle (DIP)

> "High-level modules should not depend on low-level modules. Both should depend on abstractions."

### When to apply (mechanical tells)

- `new ConcreteClass()` inside a business/domain class
- Direct DB/HTTP/Filesystem calls in domain logic
- Singleton (`X.getInstance()`) accessed from business code
- Cannot test the class without setting up real infrastructure

### Operationalized

```typescript
// ❌ — domain depends on concrete infrastructure
class OrderService {
  private email = new SendGridEmailService();  // hardcoded vendor
  private db = new PostgresClient();           // hardcoded DB

  confirmOrder(orderId: OrderId): Promise<void> {
    const order = await this.db.findOrder(orderId);
    return this.email.send(order.customerEmail, "confirmed");
  }
}

// ✅ — domain depends on abstractions; infrastructure injected
interface OrderRepository {
  findOrder(id: OrderId): Promise<Order | null>;
}
interface EmailService {
  send(to: Email, message: string): Promise<void>;
}

class OrderService {
  constructor(
    private readonly orders: OrderRepository,
    private readonly email: EmailService,
  ) {}

  async confirmOrder(id: OrderId): Promise<void> {
    const order = await this.orders.findOrder(id);
    if (!order) throw new OrderNotFound(id);
    await this.email.send(order.customerEmail, "confirmed");
  }
}

// Wiring (composition root, NOT in domain)
const service = new OrderService(
  new PostgresOrderRepository(db),
  new SendGridEmailService(apiKey),
);

// In tests
const service = new OrderService(
  new InMemoryOrderRepository(),
  new FakeEmailService(),
);
```

### The Dependency Rule (architecture)

```
┌──────────────────────────────────────┐
│      Frameworks & Drivers            │  ← outer
├──────────────────────────────────────┤
│      Interface Adapters              │
├──────────────────────────────────────┤
│      Use Cases (Application)         │
├──────────────────────────────────────┤
│      Entities (Domain)               │  ← inner
└──────────────────────────────────────┘

Source code dependencies point INWARD.
Domain code knows NOTHING about outer layers.
```

The interface lives in the inner layer (domain). The implementation lives in the outer layer (infrastructure). Outer depends on inner; inner never depends on outer.

### Where to define the interface — operational rule

**The interface belongs in the layer that USES it, not the layer that IMPLEMENTS it.**

```
domain/
  OrderRepository.ts        ← interface defined here (used by domain)
  Order.ts

infrastructure/
  PostgresOrderRepository.ts ← implementation here (depends on domain interface)
```

### Common DIP mistakes

- **Defining interface in infrastructure layer** — defeats the purpose
- **Interface mirrors database tables** — the interface should reflect the DOMAIN's needs (`findActiveOrders` not `selectFromOrdersWhereStatus`)
- **Adding interface for everything** — only when you need substitutability (test fakes, multiple environments). For DIP-as-architecture-rule, yes always between layers; for in-layer code, only when needed.

---

## SOLID at Scale

| Principle | Class level | Module level | Service level |
|-----------|-------------|--------------|---------------|
| SRP | One reason to change | One bounded context | One business capability |
| OCP | New variant = new class | New feature = new module | New capability = new service |
| LSP | Subclasses substitutable | Module impls substitutable | Services with same contract substitutable |
| ISP | Small interfaces | Thin module APIs | Narrow service APIs |
| DIP | Depend on interfaces | Domain doesn't import infra | Business services don't depend on infra services |

---

## Quick Reference

| Principle | Mechanical Tell | Fix |
|-----------|-----------------|-----|
| **SRP** | > 50 lines, > 2 vars, "and" in description | Split per verb domain |
| **OCP** | `switch`/`if-else` on type field | Replace with polymorphism |
| **LSP** | `instanceof` checks, subclass throws | Don't inherit; use composition |
| **ISP** | Empty / "not implemented" methods | Split interface per client |
| **DIP** | `new ConcreteX()` in domain | Inject abstraction via constructor |

---

## When NOT to apply SOLID

- **Throwaway scripts** — overkill, ship the procedural version
- **Spike code** — exploring; rewrite if it survives
- **Truly trivial CRUD with no logic** — domain layer is just data shape
- **MVP under explicit "we'll rewrite this" agreement** — but document the debt

See `when-not-to-apply.md` for the full decision tree.
