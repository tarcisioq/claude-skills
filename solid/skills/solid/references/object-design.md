# Object-Oriented Design

How to design objects from responsibilities. Operational rules for choosing stereotypes, deciding value object vs entity, designing aggregates.

## Decision Matrix — Designing a New Class

| Question | Answer guides |
|----------|---------------|
| Does this object have identity that survives attribute changes? | Entity vs Value Object |
| Does this object hold state, perform work, or coordinate? | Stereotype |
| Does this object need a lifecycle (init/destroy)? | Service vs domain object |
| Does this object cluster other objects under invariants? | Aggregate root |
| Could 2+ implementations exist? | Interface + dependency inversion |

---

## Responsibility-Driven Design (RDD)

**Objects are defined by their responsibilities, not their data.**

### Finding objects (steps)

1. Read the requirement / use case.
2. Underline **nouns** → candidate objects.
3. Underline **verbs** → candidate methods/behaviors.
4. Identify **domain concepts** (currencies, ids, statuses) → value objects.
5. Identify which nouns have identity (survive change of attributes) → entities.

### Finding responsibilities

For each object, answer:

- What does it **know** (data, state)?
- What does it **do** (behavior, operations)?
- What does it **decide** (rules, branching)?

If you can't answer clearly, the object isn't well-designed. Refactor.

### The two operational questions

For every class:

1. **"What pattern is this?"** → which stereotype? which design pattern?
2. **"Is it doing too much?"** → run object calisthenics: > 50 lines, > 2 vars, "and" in description?

If you can't answer #1 cleanly, the class lacks intent.
If #2 fires, the class needs splitting.

---

## Object Stereotypes

Every class fits one (or maybe two) of these. Knowing the stereotype guides the design.

| Stereotype | Purpose | Examples | Telltale |
|------------|---------|----------|----------|
| **Information Holder** | Knows things, holds data + minimal behavior | `User`, `Product`, `Address` | Mostly readers; few mutators |
| **Structurer** | Maintains relationships between objects | `OrderItems`, `UserGroup`, `Tree` | Wraps a collection; manages access |
| **Service Provider** | Performs work, often stateless | `EmailSender`, `PasswordHasher` | No state; methods take inputs, return outputs |
| **Coordinator** | Orchestrates workflow across objects | `OrderFulfillmentService`, `CheckoutFlow` | Calls 3-5 collaborators; little logic of its own |
| **Controller** | Makes decisions, delegates work | `CheckoutController`, HTTP handlers | Receives request, picks action, delegates |
| **Interfacer** | Transforms between systems | `UserAPIAdapter`, `DatabaseMapper`, `JsonSerializer` | Converts between formats |

### Stereotype decision tree

```
Does the class CONTAIN data?
├── YES, immutable → Information Holder (value object)
├── YES, mutable with identity → Information Holder (entity)
├── YES, manages a collection → Structurer (first-class collection)
└── NO  ↓
    Does it perform work without state?
    ├── YES → Service Provider
    └── NO  ↓
        Does it orchestrate other objects?
        ├── YES, many-step workflow → Coordinator
        ├── YES, request → action mapping → Controller
        └── NO, format translation → Interfacer
```

**Anti-pattern:** classes that span multiple stereotypes. A `User` that's also a `UserService` and a `UserMapper` is three classes glued together.

---

## Value Objects vs Entities

The most fundamental distinction in domain modeling.

### Value Object

- **No identity** — equality by attributes
- **Immutable** — operations return new instances
- **Self-validating** — invariants enforced in constructor
- **Examples:** `Money`, `Email`, `Address`, `DateRange`, `Color`, `OrderId`

```typescript
class Money {
  constructor(
    private readonly amountCents: number,
    private readonly currency: Currency,
  ) {
    if (amountCents < 0 || !Number.isInteger(amountCents)) throw new InvalidAmount();
  }

  equals(other: Money): boolean {
    return this.amountCents === other.amountCents
        && this.currency.equals(other.currency);
  }

  add(other: Money): Money {
    if (!this.currency.equals(other.currency)) throw new CurrencyMismatch();
    return new Money(this.amountCents + other.amountCents, this.currency);
  }

  // No setters. Operations return new Money.
}

// Two Money objects are equal if same amount + currency
const a = new Money(100, USD);
const b = new Money(100, USD);
console.log(a.equals(b)); // true (even though a !== b in memory)
```

### Entity

- **Has identity** — equality by id, not attributes
- **Mutable** through methods (state evolves over time)
- **Lifecycle** — created, modified, perhaps deleted
- **Examples:** `User`, `Order`, `Product`, `Account`

```typescript
class User {
  constructor(
    private readonly id: UserId,        // identity — never changes
    private email: Email,                // can change
    private name: Name,                  // can change
  ) {}

  equals(other: User): boolean {
    return this.id.equals(other.id);    // identity comparison, NOT attribute
  }

  changeEmail(newEmail: Email): void {
    this.email = newEmail;               // still the same User
  }
}

// Two Users are equal iff same id, regardless of attributes
const u1 = new User(UserId.of("u1"), Email.of("a@x.com"), Name.of("Alice"));
const u2 = new User(UserId.of("u1"), Email.of("b@x.com"), Name.of("Bob"));
console.log(u1.equals(u2)); // true — same identity
```

### Decision: Value Object or Entity?

```
Does this object need to be tracked over time as the same conceptual thing,
even if all its attributes change?
├── YES → Entity (has identity)
└── NO  → Value Object (defined by attributes)
```

| Concept | VO or Entity? |
|---------|---------------|
| `User` | Entity (the same person, changing email/name) |
| `Email` | Value Object (no identity beyond the address) |
| `Order` | Entity (order #1234 stays #1234 through state changes) |
| `Money` | Value Object ($100 USD = $100 USD anywhere) |
| `Product` | Entity (SKU stays the same; price/description change) |
| `Address` | Value Object (no separate identity from its attributes) |
| `OrderItem` | Usually Value Object (defined by product+quantity) — sometimes Entity if has its own lifecycle |

### Why this matters operationally

- **VOs** can be freely shared, copied, compared by value, never mutated
- **Entities** require careful lifecycle management; mutation rules; identity-based equality

Confusing the two leads to bugs: mutating a VO that's shared, or comparing entities by attributes (and getting "different" when only the email changed).

---

## Tell, Don't Ask

**Command objects to do work. Don't interrogate them and do the work yourself.**

### The pattern

```typescript
// ❌ — caller asks, caller decides, caller acts
if (account.getBalance() >= amount) {
  account.setBalance(account.getBalance() - amount);
  logger.info(`Withdrew ${amount}`);
} else {
  throw new InsufficientFunds();
}

// ✅ — caller tells; account decides
const result = account.withdraw(amount);
if (result.isSuccess()) {
  logger.info(`Withdrew ${amount}`);
} else {
  throw new InsufficientFunds();
}
```

### When you find yourself asking

If you write `if (obj.getX() ...)` followed by mutating `obj`, the logic belongs INSIDE `obj`.

```typescript
// ❌
if (user.subscriptionLevel >= 2 && !user.banned && user.emailVerified) {
  user.grantPremium();
}

// ✅
if (user.canAccessPremium()) {
  user.grantPremium();
}

// or even better
user.tryGrantPremium();  // method handles eligibility internally
```

### Rare exception: read-only views

For pure data display, getters are fine. The rule fires when there's a query-then-mutate pattern.

---

## Design by Contract (DbC)

Every method has:

- **Preconditions** — what must be true BEFORE calling
- **Postconditions** — what will be true AFTER calling
- **Invariants** — what is ALWAYS true about the object

```typescript
class BankAccount {
  private balance: Money;

  // INVARIANT: balance is never negative

  /**
   * Withdraws funds.
   *
   * @precondition amount > 0
   * @precondition balance >= amount (else returns InsufficientFunds)
   * @postcondition on success: new balance = old balance - amount
   * @postcondition on failure: balance unchanged
   * @returns success result with new balance, or InsufficientFunds
   */
  withdraw(amount: Money): WithdrawResult {
    if (amount.isNegativeOrZero()) {
      return WithdrawResult.invalidAmount();
    }
    if (this.balance.lessThan(amount)) {
      return WithdrawResult.insufficientFunds();
    }
    this.balance = this.balance.minus(amount);
    return WithdrawResult.success(this.balance);
  }
}
```

### Encoding contracts

| Contract type | Method |
|---------------|--------|
| Precondition (input validation) | Throw early or return error result |
| Postcondition | Tests verify; or invariant assertion in code |
| Invariant | Constructor enforces; private state ensures it stays |

**Tell of broken contract:** consumer of class needs to pre-validate before calling. Move that validation INSIDE the method.

---

## Composition Over Inheritance

**Default: compose. Inherit only when forced.**

### Why inheritance is problematic

- **Tight coupling** between parent and child (fragile base class)
- **Single dimension** of variation (can extend only one parent)
- **"Is-a" forced** even when not natural fit
- **LSP risks** — easy to violate parent's contract

### When to use inheritance

ALL three must hold:

1. True "is-a" relationship (passes substitution test)
2. Framework requires it (extending `Error`, `EventTarget`, `Component`)
3. Single level deep (no `Dog extends Animal extends LivingThing`)

Otherwise: compose.

### Composition pattern

```typescript
// ❌ — inheritance for code reuse
class User { /* base */ }
class PremiumUser extends User {
  getDiscount(): Percent { return Percent.of(20); }
}
class StandardUser extends User {
  getDiscount(): Percent { return Percent.zero(); }
}

// ✅ — compose: discount strategy injected
class User {
  constructor(
    private readonly id: UserId,
    private readonly discountPolicy: DiscountPolicy,
  ) {}

  getDiscount(): Percent {
    return this.discountPolicy.calculate();
  }
}

interface DiscountPolicy {
  calculate(): Percent;
}

class PremiumDiscount implements DiscountPolicy {
  calculate(): Percent { return Percent.of(20); }
}
class StandardDiscount implements DiscountPolicy {
  calculate(): Percent { return Percent.zero(); }
}
class BlackFridayDiscount implements DiscountPolicy {
  calculate(): Percent { return Percent.of(50); }
}

// Multiple variation dimensions composable
const user = new User(id, new PremiumDiscount());
```

### Tell: which to use?

```
Is the relationship truly "is-a" (Dog IS-A Animal, Square IS-A Shape)?
├── NO  → composition
└── YES ↓
    Will subclasses honor parent's contract perfectly (LSP)?
    ├── NO  → composition
    └── YES ↓
        Is the hierarchy ≤ 1 level deep AND framework-forced?
        ├── NO  → composition
        └── YES → inheritance OK
```

**Rule of thumb:** if you find yourself writing `instanceof` checks or `if (parent.feature) doX()`, the inheritance is wrong. Refactor to composition.

---

## Law of Demeter (Principle of Least Knowledge)

**Only talk to your immediate friends.**

A method should only call:

1. Methods on `this`
2. Methods on parameters
3. Methods on objects it creates
4. Methods on its direct components (instance variables)

### Pattern

```typescript
// ❌ — train wreck (chains through object graph)
const city = order.getCustomer().getAddress().getCity();

// ✅ — ask immediate friend; let it ask its friends
const city = order.shippingCity();

class Order {
  shippingCity(): City {
    return this.customer.shippingAddress().city();  // delegates internally
  }
}
```

### Tell

More than one `.` per chain on object accesses. Examples that are NOT violations:

- Builder chains: `query.where().orderBy().limit()` (fluent interface, intentional)
- Array/string method chains: `arr.filter().map().reduce()` (functional pipelines)
- Same object: `obj.method().value` (one access)

### Why it matters

If `order.getCustomer().getAddress()` becomes `order.getCustomer().getMailingAddress()`, every caller breaks. With `order.shippingCity()`, only Order knows the chain — change it once.

---

## Encapsulation Levels

| Level | Hides | Achieved via |
|-------|-------|--------------|
| **Data** | Internal state | `private` fields |
| **Implementation** | How things work | `private` methods, internal classes |
| **Type** | Concrete class identity | Interface / abstract class |
| **Design** | Architectural choices | Module boundaries, public APIs |

### Pattern

```typescript
// ❌ — exposed internals
class Order {
  public items: Item[] = [];
  public total: number = 0;
}

// Caller can corrupt state
order.items.push(item);             // bypasses any rule
order.total = -999;                 // invariant broken
order.items.length = 0;             // mass mutation

// ✅ — encapsulated
class Order {
  private readonly items: OrderItems;
  private total: Money;

  addItem(item: OrderItem): void {
    this.items.add(item);
    this.recalculateTotal();
  }

  removeItem(id: ItemId): void {
    this.items.remove(id);
    this.recalculateTotal();
  }

  getTotal(): Money {
    return this.total;             // immutable Money — safe to expose
  }

  // Internals can change freely without breaking callers
  private recalculateTotal(): void { /* ... */ }
}
```

### Decision: expose or hide?

```
Is this internal to how the class achieves its job?
├── YES → private (data + helper methods)
└── NO  ↓
    Is this part of the contract callers depend on?
    ├── YES → public (and document as @public)
    └── NO  → private (don't widen API "just in case")
```

---

## Polymorphism — Replace Conditionals with Types

```typescript
// ❌ — type-switch
function calculateShipping(method: string, value: number): number {
  if (method === "standard") return value < 50 ? 5 : 0;
  if (method === "express") return 15;
  if (method === "overnight") return 25;
  throw new Error("Unknown method");
}

// ✅ — polymorphism
interface ShippingMethod {
  cost(orderValue: Money): Money;
}

class StandardShipping implements ShippingMethod {
  cost(value: Money): Money { return value.lessThan(Money.of(50)) ? Money.of(5) : Money.zero(); }
}
class ExpressShipping implements ShippingMethod {
  cost(): Money { return Money.of(15); }
}
class OvernightShipping implements ShippingMethod {
  cost(): Money { return Money.of(25); }
}

function calculateShipping(method: ShippingMethod, value: Money): Money {
  return method.cost(value);
}
```

### When NOT to polymorphism

- 2 cases that won't grow (boolean flag is fine)
- Truly fixed enumeration (HTTP method, weekday)
- Simple value lookup (use `Map<key, value>` instead)

---

## Aggregates

A cluster of objects treated as a single unit for data changes.

### Concept

- One object is the **aggregate root**
- External code only references the root
- The root enforces invariants for the cluster
- Persistence happens via the root

```typescript
// Order is the aggregate root; OrderItem is internal to the cluster
class Order {
  private readonly items: OrderItem[] = [];

  // ALL access through the root
  addItem(product: Product, quantity: Quantity): void {
    const item = new OrderItem(product, quantity);
    this.items.push(item);
    this.enforceInvariants();
  }

  removeItem(itemId: ItemId): void {
    const idx = this.items.findIndex(i => i.id.equals(itemId));
    if (idx === -1) throw new ItemNotFound(itemId);
    this.items.splice(idx, 1);
    this.enforceInvariants();
  }

  // Root enforces invariants for the entire cluster
  private enforceInvariants(): void {
    if (this.items.length === 0) throw new EmptyOrder();
    if (this.calculateTotal().greaterThan(MAX_ORDER_VALUE)) throw new OrderTotalExceeded();
  }
}

// ❌ — accessing items directly bypasses invariants
order.items.push(new OrderItem(...));  // invariant check skipped

// ✅ — through the root
order.addItem(product, quantity);
```

### Aggregate boundaries (operational rules)

- **One transaction = one aggregate.** Don't update multiple aggregates atomically; use eventual consistency between them.
- **Reference between aggregates by ID, not by object.** `Order` holds `customerId: CustomerId`, not `customer: Customer`.
- **Repositories per aggregate root.** No `OrderItemRepository` — only `OrderRepository`.

### Decision: where's the aggregate boundary?

```
Group objects that:
- Must change together atomically (invariant requires it)
- Have lifecycle tied (one is created/destroyed with the other)
- The application thinks of as a unit

Don't group:
- Things that change for different reasons
- Things that have separate access patterns
- Things that need separate transactions
```

---

## Quick Reference

| Concern | Pattern |
|---------|---------|
| New object | Identify nouns + verbs; pick stereotype |
| Has identity? | Entity (id-based equality) |
| Defined by attributes? | Value object (immutable, attribute-based equality) |
| Holds collection + other state? | Split — first-class collection class |
| Asks-then-mutates pattern | Tell, Don't Ask — move logic into the object |
| Inheritance candidate | Default: compose. Inherit only when truly is-a + framework-forced |
| Cross-graph access (`a.b.c`) | Hide Delegate — Demeter |
| Type switch | Replace with polymorphism |
| Cluster with invariants | Aggregate with root enforcing |
| Cross-aggregate reference | By ID, not by object |
| Encapsulation | Default to private; widen only when contract requires |
| Method contract | Document preconditions, postconditions, invariants |
