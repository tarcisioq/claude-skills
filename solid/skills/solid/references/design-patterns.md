# Design Patterns

Reusable solutions to recurring problems. **Patterns emerge from refactoring — they are NOT applied upfront.** This file is a recognition guide + when-NOT-to-apply matrix.

## Decision Matrix — Should I Use a Pattern?

```
Have I seen this exact problem shape 2+ times before?
├── NO  → Don't force a pattern. Write straight code.
└── YES ↓
    Does the pattern make this SIMPLER (less code, clearer intent)?
    ├── NO  → Don't use it. Pattern is overhead, not virtue.
    └── YES ↓
        Will the team understand it without explanation?
        ├── NO  → Use only with comment + reference link
        └── YES → Apply pattern
```

**Anti-pattern:** "Pattern-first design." Patterns should emerge from the third concrete need (Rule of Three), not be sprinkled on for "good design."

---

## Pattern Recognition Guide

When reading code, identify patterns to share vocabulary. When writing code, recognize when a pattern naturally fits.

### Creational

| Pattern | Solves | When to use | When NOT |
|---------|--------|-------------|----------|
| Factory | Object creation hidden from caller | Multiple types from a single API; complex construction | Single concrete class with simple constructor |
| Builder | Complex objects with many optional parts | > 4 optional params; staged construction | Simple objects with few attributes |
| Singleton | Exactly one instance | Truly process-wide resource (rare) | Almost always — use DI instead |
| Prototype | Clone existing object | Expensive creation; tweak a copy | Simple value objects (just construct) |

### Structural

| Pattern | Solves | When to use | When NOT |
|---------|--------|-------------|----------|
| Adapter | Incompatible interfaces | Wrapping a third-party library | Interface compatible already |
| Decorator | Add behavior dynamically | Composing optional features | Few combinations; better as direct subclass |
| Proxy | Control access | Lazy load, caching, access check | Adds indirection without value |
| Composite | Tree of objects, treated uniformly | File systems, UI trees, expression trees | Linear data |
| Bridge | Decouple abstraction from impl | 2 dimensions of variation | One dimension |
| Facade | Simplify complex subsystem | Hide library complexity behind simpler API | Subsystem already simple |

### Behavioral

| Pattern | Solves | When to use | When NOT |
|---------|--------|-------------|----------|
| Strategy | Interchangeable algorithms | Behavior varies at runtime | Behavior fixed at compile time |
| Observer | Notify multiple objects of change | Event system, pub/sub | Simple direct call works |
| Template Method | Common algorithm with varying steps | Step skeleton + slot for customization | No common steps |
| Command | Encapsulate request | Undo/redo, queuing, logging | Direct call suffices |
| State | Object behavior changes with state | > 3 states with rules | Simple flag |
| Visitor | Operations on heterogeneous types | Multiple ops over fixed type hierarchy | Few ops; better as method per class |
| Iterator | Traverse without exposing internals | Custom container | Use language's built-in iteration |

---

## Creational Patterns

### Factory

**Purpose:** create objects without exposing the concrete class to the caller.

**When to use:**
- Construction is complex (validation, dependency wiring)
- Returned type varies by input
- You want to hide which subclass is created

```typescript
interface Notifier {
  send(message: string): void;
}

class EmailNotifier implements Notifier { /* ... */ }
class SMSNotifier implements Notifier { /* ... */ }
class PushNotifier implements Notifier { /* ... */ }

class NotifierFactory {
  create(channel: Channel): Notifier {
    if (channel.equals(Channel.EMAIL)) return new EmailNotifier();
    if (channel.equals(Channel.SMS)) return new SMSNotifier();
    if (channel.equals(Channel.PUSH)) return new PushNotifier();
    throw new UnknownChannel(channel);
  }
}
```

**When NOT:**
- A single class with a simple constructor — `new EmailNotifier()` is fine
- The returned class is always the same — factory adds noise

**Anti-pattern:** factory that wraps `new ConcreteClass()` and returns it. Just call the constructor.

### Builder

**Purpose:** construct complex objects step-by-step.

**When to use:**
- > 4 constructor parameters, many optional
- Test data setup (`UserBuilder` is canonical)
- Domain-specific construction (SQL builders, query builders)

```typescript
class UserBuilder {
  private email = Email.of("default@example.com");
  private age = Age.of(30);
  private premium = false;

  withEmail(e: string): this { this.email = Email.of(e); return this; }
  withAge(a: number): this { this.age = Age.of(a); return this; }
  premiumUser(): this { this.premium = true; return this; }

  build(): User {
    return new User(this.email, this.age, this.premium);
  }
}

const user = new UserBuilder().premiumUser().withAge(25).build();
```

**When NOT:**
- Few constructor params (just use the constructor)
- All params required (use the constructor)

### Singleton

**Purpose:** ensure exactly one instance exists.

**When to use:** truly process-wide resource (configuration loaded from disk once, app-wide event bus). Rare.

**When NOT (almost always):**
- Use Dependency Injection instead
- "Global access" is the symptom, not the requirement

```typescript
// ❌ — singleton hides dependency, breaks testability
class Logger {
  private static instance: Logger;
  static getInstance(): Logger {
    if (!Logger.instance) Logger.instance = new Logger();
    return Logger.instance;
  }
}

class OrderService {
  process() {
    Logger.getInstance().info("processing");  // hidden dep
  }
}

// ✅ — DI: explicit, testable
class OrderService {
  constructor(private readonly logger: Logger) {}
  process() {
    this.logger.info("processing");
  }
}

// Wire once at composition root
const logger = new Logger();
const service = new OrderService(logger);
```

**Anti-pattern alert:** singletons accessed via `getInstance()` are global state in disguise. They make tests harder, hide dependencies, and create implicit coupling.

### Prototype

**Purpose:** create new objects by cloning existing ones.

**When to use:**
- Object creation is expensive
- You need many slight variations of an object

```typescript
interface Cloneable<T> { clone(): T; }

class Document implements Cloneable<Document> {
  constructor(
    public readonly title: string,
    public readonly content: string,
    public readonly metadata: Metadata,
  ) {}

  clone(): Document {
    return new Document(this.title, this.content, this.metadata.clone());
  }
}
```

**When NOT:**
- Value objects — just construct (`new Money(100, USD)`)
- Modern alternative: `structuredClone(obj)` (browsers, Node 17+) handles deep copy

---

## Structural Patterns

### Adapter

**Purpose:** make incompatible interfaces work together. Wrapper around third-party libs.

**When to use:**
- Integrating a library with a different shape from your domain
- Wrapping legacy code while you migrate

```typescript
// Third-party API (different shape)
class StripeOldAPI {
  charge(amountCents: number, cardToken: string): { success: boolean } { /* ... */ }
}

// Domain interface
interface PaymentGateway {
  pay(amount: Money, card: Card): PaymentResult;
}

// Adapter
class StripeAdapter implements PaymentGateway {
  constructor(private readonly stripe: StripeOldAPI) {}

  pay(amount: Money, card: Card): PaymentResult {
    const result = this.stripe.charge(amount.cents(), card.token());
    return result.success ? PaymentResult.success() : PaymentResult.failed();
  }
}
```

**When NOT:** if you control both sides, adjust one rather than wrap.

### Decorator

**Purpose:** add behavior to an object dynamically without subclassing.

**When to use:**
- Optional features that compose
- Cross-cutting concerns layered onto core behavior

```typescript
interface Notifier {
  send(message: string): void;
}

class EmailNotifier implements Notifier {
  send(msg: string): void { /* ... */ }
}

// Decorator: adds SMS without modifying EmailNotifier
class WithSMSDecorator implements Notifier {
  constructor(private readonly wrapped: Notifier) {}
  send(msg: string): void {
    this.wrapped.send(msg);
    sendSMS(msg);
  }
}

// Decorator: adds Slack
class WithSlackDecorator implements Notifier {
  constructor(private readonly wrapped: Notifier) {}
  send(msg: string): void {
    this.wrapped.send(msg);
    postToSlack(msg);
  }
}

// Compose
const notifier = new WithSlackDecorator(new WithSMSDecorator(new EmailNotifier()));
notifier.send("Alert!");  // sends email + SMS + Slack
```

**When NOT:** few combinations and stable — use direct subclasses or composition.

### Proxy

**Purpose:** control access to an object — for lazy loading, caching, security, logging.

**When to use:**
- Expensive object that should be loaded on first use
- Need to intercept calls (auth, audit)

```typescript
interface Image {
  display(): void;
}

class RealImage implements Image {
  constructor(private readonly filename: string) {
    this.loadFromDisk();  // expensive
  }
  private loadFromDisk(): void { /* ... */ }
  display(): void { /* ... */ }
}

class LazyImageProxy implements Image {
  private real: RealImage | null = null;

  constructor(private readonly filename: string) {}

  display(): void {
    if (!this.real) this.real = new RealImage(this.filename);
    this.real.display();
  }
}
```

**When NOT:** the indirection itself is overhead — only worth it when interception adds value.

### Composite

**Purpose:** treat individual objects and compositions uniformly.

**When to use:**
- Tree structures (file systems, UI components, organizational hierarchy)

```typescript
interface PriceComponent {
  totalPrice(): Money;
}

class Product implements PriceComponent {
  constructor(private readonly price: Money) {}
  totalPrice(): Money { return this.price; }
}

class Box implements PriceComponent {
  private contents: PriceComponent[] = [];

  add(item: PriceComponent): void { this.contents.push(item); }

  totalPrice(): Money {
    return this.contents.reduce((sum, c) => sum.add(c.totalPrice()), Money.zero());
  }
}

const small = new Box();
small.add(new Product(Money.dollars(10)));
small.add(new Product(Money.dollars(20)));

const big = new Box();
big.add(small);
big.add(new Product(Money.dollars(50)));

big.totalPrice();  // 80
```

**When NOT:** linear data — use an array.

### Facade

**Purpose:** provide a simplified interface to a complex subsystem.

**When to use:** wrapping a library or service that has many ways to use it but most clients want one simple API.

```typescript
// Complex subsystem (multiple classes, complex setup)
class InventorySystem { /* ... */ }
class PricingEngine { /* ... */ }
class TaxCalculator { /* ... */ }
class ShippingCalculator { /* ... */ }

// Facade — simple API for the common case
class CheckoutFacade {
  constructor(
    private readonly inventory: InventorySystem,
    private readonly pricing: PricingEngine,
    private readonly tax: TaxCalculator,
    private readonly shipping: ShippingCalculator,
  ) {}

  computeTotal(cart: Cart, address: Address): Money {
    const subtotal = this.pricing.calculate(cart);
    const taxAmount = this.tax.calculate(subtotal, address);
    const shippingCost = this.shipping.calculate(cart, address);
    return subtotal.add(taxAmount).add(shippingCost);
  }
}
```

**When NOT:** the underlying system is already simple.

---

## Behavioral Patterns

### Strategy

**Purpose:** define a family of algorithms; make them interchangeable at runtime.

**When to use:**
- Multiple ways to do the same thing
- Choice depends on runtime context

```typescript
interface DiscountPolicy {
  apply(amount: Money): Money;
}

class NoDiscount implements DiscountPolicy {
  apply(amount: Money): Money { return amount; }
}
class PercentageDiscount implements DiscountPolicy {
  constructor(private readonly percent: Percent) {}
  apply(amount: Money): Money { return amount.subtract(amount.percentOf(this.percent)); }
}
class FlatDiscount implements DiscountPolicy {
  constructor(private readonly off: Money) {}
  apply(amount: Money): Money { return amount.subtract(this.off).orZero(); }
}

class Cart {
  constructor(private readonly discount: DiscountPolicy) {}
  total(items: ReadonlyArray<Item>): Money {
    const subtotal = items.reduce((sum, i) => sum.add(i.price), Money.zero());
    return this.discount.apply(subtotal);
  }
}
```

**When NOT:** only 2 cases that won't grow — boolean works.

### Observer

**Purpose:** notify multiple subscribers when an object's state changes.

**When to use:**
- Event-driven systems
- One-to-many notifications
- Decoupled subscribers

```typescript
type Listener<T> = (event: T) => void;

class EventEmitter<T> {
  private listeners: Set<Listener<T>> = new Set();

  on(listener: Listener<T>): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);  // unsubscribe fn
  }

  emit(event: T): void {
    for (const l of this.listeners) {
      try { l(event); } catch (err) { console.error(err); }  // isolate
    }
  }
}

class OrderService {
  readonly orderPlaced = new EventEmitter<Order>();

  place(order: Order): void {
    /* ... */
    this.orderPlaced.emit(order);
  }
}

// Subscribers
service.orderPlaced.on(order => emailService.sendConfirmation(order));
service.orderPlaced.on(order => analytics.track(order));
```

**When NOT:** simple direct call works (one caller, one callee).

### Template Method

**Purpose:** define an algorithm skeleton; let subclasses override specific steps.

**When to use:** common algorithm with varying steps. Often enforced by framework (lifecycle hooks).

```typescript
abstract class DataExporter {
  // Template method — invariant order; subclasses override hooks
  export(data: Data[]): void {
    this.validate(data);
    const formatted = this.format(data);
    this.write(formatted);
    this.notify();
  }

  // Common steps
  private validate(data: Data[]): void { /* ... */ }
  private notify(): void { /* ... */ }

  // Hooks — each subclass overrides
  protected abstract format(data: Data[]): string;
  protected abstract write(content: string): void;
}

class CSVExporter extends DataExporter {
  protected format(data: Data[]): string {
    return data.map(d => d.toCSV()).join("\n");
  }
  protected write(content: string): void {
    fs.writeFileSync("export.csv", content);
  }
}

class JSONExporter extends DataExporter {
  protected format(data: Data[]): string {
    return JSON.stringify(data);
  }
  protected write(content: string): void {
    fs.writeFileSync("export.json", content);
  }
}
```

**When NOT:** prefer composition (Strategy pattern) when possible. Template Method uses inheritance, which has costs (see `object-design.md`).

### Command

**Purpose:** encapsulate a request as an object — for queueing, logging, undo/redo.

**When to use:**
- Undo/redo
- Queueing actions
- Audit logs of operations

```typescript
interface Command {
  execute(): void;
  undo(): void;
}

class AddItemCommand implements Command {
  constructor(private readonly cart: Cart, private readonly item: Item) {}
  execute(): void { this.cart.add(this.item); }
  undo(): void { this.cart.remove(this.item); }
}

class CommandHistory {
  private history: Command[] = [];

  execute(command: Command): void {
    command.execute();
    this.history.push(command);
  }

  undo(): void {
    const cmd = this.history.pop();
    cmd?.undo();
  }
}
```

**When NOT:** simple operation, no need for queueing/undo — direct method call.

### State

**Purpose:** alter object behavior when its internal state changes.

**When to use:** > 3 states with state-dependent behavior and transition rules.

```typescript
interface OrderState {
  pay(order: Order): void;
  cancel(order: Order): void;
  ship(order: Order): void;
}

class PendingState implements OrderState {
  pay(order: Order): void { order.setState(new PaidState()); }
  cancel(order: Order): void { order.setState(new CancelledState()); }
  ship(_order: Order): void { throw new InvalidTransition(); }
}

class PaidState implements OrderState {
  pay(_order: Order): void { throw new InvalidTransition(); }
  cancel(order: Order): void { order.setState(new CancelledState()); }
  ship(order: Order): void { order.setState(new ShippedState()); }
}

// ... ShippedState, CancelledState

class Order {
  private state: OrderState = new PendingState();
  setState(s: OrderState): void { this.state = s; }
  pay(): void { this.state.pay(this); }
  cancel(): void { this.state.cancel(this); }
  ship(): void { this.state.ship(this); }
}
```

**When NOT:** ≤ 3 states — a string + switch is clearer.

---

## Anti-Patterns to Avoid

| Anti-Pattern | What it is | Fix |
|--------------|-----------|-----|
| **God Object** | One class does everything | Split per responsibility (SRP) |
| **Spaghetti Code** | No structure; tangled flow | Refactor to layers |
| **Golden Hammer** | Using one pattern everywhere | Match pattern to problem |
| **Singleton Abuse** | Singleton for everything "shared" | Use DI |
| **Premature Optimization** | Optimizing before measuring | YAGNI; profile first |
| **Speculative Generality** | Abstractions for hypothetical futures | Delete; add when needed |
| **Copy-Paste Programming** | Duplication across the codebase | Extract after Rule of Three |
| **Pattern-First Design** | Sprinkling patterns for "good design" | Let patterns emerge from refactoring |
| **Yo-Yo Problem** | Inheritance hierarchy too deep — readers bounce up/down | Compose instead |
| **Cargo Cult** | Copying patterns without understanding why | Read the original problem the pattern solves |

---

## Decision Trees — Pattern Selection

### "I have a class with many optional construction parameters"

```
> 4 optional params?
├── YES → Builder
└── NO  → Constructor with default values
```

### "I have a switch on type that keeps growing"

```
Adding new variants frequently?
├── YES → Strategy or Polymorphism
└── NO  ↓
    Stable enumeration?
    ├── YES → keep the switch (simple)
    └── NO  → maybe Visitor if many ops over types
```

### "I need to wrap a third-party library"

```
Different interface shape?
├── YES → Adapter
└── NO  ↓
    Add cross-cutting behavior (cache, log)?
    ├── YES → Proxy or Decorator
    └── NO  → use the library directly
```

### "Multiple subscribers to an event"

```
Truly need multiple listeners?
├── YES → Observer
└── NO  → direct method call
```

### "Object behavior depends on state"

```
> 3 states with rules?
├── YES → State pattern
└── NO  → string + switch
```

### "Need to undo / queue / log operations"

```
Need any of: undo, queueing, logging actions, scheduling?
├── YES → Command
└── NO  → direct method call
```

---

## Quick Reference

| Need | Pattern |
|------|---------|
| Hide construction complexity | Factory |
| Many optional params | Builder |
| One instance, process-wide | (Avoid Singleton; use DI) |
| Wrap incompatible API | Adapter |
| Compose optional features | Decorator |
| Lazy load / control access | Proxy |
| Tree of objects | Composite |
| Simplify complex subsystem | Facade |
| Interchangeable algorithms | Strategy |
| One-to-many notification | Observer |
| Algorithm with hooks | Template Method |
| Encapsulate operation | Command |
| Behavior changes by state | State |
| Pattern doesn't fit | Don't force it. Write straight code. |

**Final reminder:** the best code uses few patterns. Most code is straight, simple methods on focused classes. Patterns are for the genuine recurring shapes, not embellishment.
