# Testing Strategy

Test types, structure, doubles, and how testing connects to SOLID. Operational rules for writing tests that protect behavior without locking implementation.

## Decision Matrix — Which Test Type?

| What you're testing | Layer | Test type | Speed | Doubles |
|---------------------|-------|-----------|-------|---------|
| Pure logic, value object, entity | Domain | Unit | < 1ms | None — real objects |
| Use case orchestrating multiple objects | Application | Integration (with fakes) | < 50ms | Fakes for infra |
| Database query, HTTP client adapter | Infrastructure | Integration (real deps) | < 500ms | Real DB/API (test instance) |
| User-facing flow end-to-end | All layers | E2E / Acceptance | seconds | Real or staging stack |
| Mathematical property / invariant | Any | Property-based | varies | Generators |
| Verifying interaction protocol | Boundary | Mockist | < 10ms | Mocks |

**Default ratio (rough):** 70% unit, 25% integration, 5% E2E. Adjust per layer (more unit in domain, more integration in infra).

---

## The Testing Pyramid

```
              ╱╲
             ╱  ╲     E2E / Acceptance (5%)
            ╱────╲    - Slow, brittle, high confidence
           ╱      ╲   - Critical paths only
          ╱────────╲
         ╱          ╲ Integration (25%)
        ╱            ╲ - Multiple components
       ╱──────────────╲ - Boundaries, contracts
      ╱                ╲
     ╱      Unit (70%)  ╲ - Single class/function
    ╱                    ╲ - Fast, isolated
   ╱──────────────────────╲
```

**Inverted pyramid (anti-pattern):** mostly E2E, few units. Tests slow, brittle, give false confidence.

**Diamond / hourglass:** lots of unit, lots of E2E, no integration. Hides bugs at boundaries (where most production bugs live).

**Trophy (modern):** weighted toward integration. Valid for some apps. **Whatever shape you choose, it must be deliberate.**

---

## Unit Tests

Test ONE class or function in isolation.

**Characteristics:**
- < 1ms each
- No I/O (no DB, HTTP, FS)
- Verify behavior, not implementation
- Deterministic (no time, no random — inject those seams)

```typescript
describe("Order", () => {
  it("calculates total of single item", () => {
    const order = new Order();
    order.add(OrderItem.of(Money.dollars(100)));

    expect(order.total().equals(Money.dollars(100))).toBe(true);
  });

  it("calculates total of multiple items", () => {
    const order = new Order();
    order.add(OrderItem.of(Money.dollars(100)));
    order.add(OrderItem.of(Money.dollars(50)));

    expect(order.total().equals(Money.dollars(150))).toBe(true);
  });

  it("rejects items with non-positive quantity", () => {
    const order = new Order();

    expect(() => order.add(OrderItem.of(Money.dollars(100), Quantity.of(0))))
      .toThrow(InvalidQuantity);
  });
});
```

**Anti-patterns:**
- Tests that need a database (they're integration tests, not unit)
- Tests that mock everything (you're testing the mocks, not the code)
- Tests that test private methods (test only via public API)
- Tests that share state (each test must be independent)

---

## Integration Tests

Test multiple components working together. Verify boundaries.

**Characteristics:**
- < 500ms each
- Real or in-memory infrastructure
- Test the contract between components
- Fewer than unit tests

### Application layer (use case + fakes)

```typescript
describe("CreateOrderUseCase", () => {
  let orderRepo: InMemoryOrderRepo;
  let emailService: FakeEmailService;
  let useCase: CreateOrderUseCase;

  beforeEach(() => {
    orderRepo = new InMemoryOrderRepo();
    emailService = new FakeEmailService();
    useCase = new CreateOrderUseCase(orderRepo, emailService);
  });

  it("creates order and sends confirmation", async () => {
    const result = await useCase.execute({
      customer: Customer.test(),
      items: [OrderItem.of(Money.dollars(100))],
    });

    expect(result.isSuccess()).toBe(true);
    expect(orderRepo.count()).toBe(1);
    expect(emailService.sentTo(Customer.test().email)).toBe(true);
  });

  it("does not save order when payment fails", async () => {
    const useCase = new CreateOrderUseCase(orderRepo, emailService, FailingPaymentGateway);

    const result = await useCase.execute(/* ... */);

    expect(result.isFailure()).toBe(true);
    expect(orderRepo.count()).toBe(0);
  });
});
```

### Infrastructure layer (real DB)

```typescript
describe("PostgresOrderRepo", () => {
  let repo: PostgresOrderRepo;

  beforeAll(async () => {
    await testDb.migrate();
  });

  beforeEach(async () => {
    await testDb.clear("orders");
    repo = new PostgresOrderRepo(testDb);
  });

  it("persists and retrieves an order", async () => {
    const order = anOrder().build();
    await repo.save(order);

    const retrieved = await repo.findById(order.id);

    expect(retrieved).toEqual(order);
  });

  it("returns null for missing order", async () => {
    const retrieved = await repo.findById(OrderId.of("nonexistent"));
    expect(retrieved).toBeNull();
  });
});
```

**Test database setup:**
- Containerized DB (`testcontainers`) per test run, or
- Schema-isolated test database, cleared between tests
- **Never the production DB.** Never shared with other developers.

---

## E2E / Acceptance Tests

Verify critical user paths through the entire stack.

**Characteristics:**
- Seconds per test
- Real browser, real backend, real (test) database
- Test only critical flows: signup, checkout, login
- Brittle by nature — keep them few

```typescript
describe("Checkout flow", () => {
  it("logged-in user completes purchase", async () => {
    await login(page, "test@example.com", "password");
    await page.goto("/products/widget");
    await page.click('[data-testid="add-to-cart"]');
    await page.click('[data-testid="checkout"]');
    await page.fill('[name="card-number"]', "4242 4242 4242 4242");
    await page.fill('[name="card-expiry"]', "12/30");
    await page.fill('[name="card-cvc"]', "123");
    await page.click('[data-testid="pay"]');

    await expect(page.locator("h1")).toHaveText("Order Confirmed");
    await expect(page.locator('[data-testid="order-id"]')).toBeVisible();
  });
});
```

**Rules:**
- Use `data-testid` for element selection (not CSS classes — they change for styling)
- Reset state before each test (database, auth, cookies)
- Run sparingly — slow suite kills feedback loop
- Critical paths only. NOT every form/button.

---

## Arrange-Act-Assert (AAA)

Every test has three sections, separated by blank lines.

```typescript
it("applies 20% discount to premium users", () => {
  // ARRANGE — set up world
  const user = aUser().premium().build();
  const cart = new Cart(user);
  cart.add(Money.dollars(100));

  // ACT — execute the behavior under test
  const total = cart.calculateTotal();

  // ASSERT — verify outcome
  expect(total.equals(Money.dollars(80))).toBe(true);
});
```

### One ACT per test

```typescript
// ❌ — multiple acts, one test verifies many things
it("can do checkout flow", () => {
  cart.add(item1);
  expect(cart.count()).toBe(1);

  cart.applyDiscount(discount);
  expect(cart.total()).toBe(80);

  cart.checkout();
  expect(cart.isEmpty()).toBe(true);
});

// ✅ — one act per test
it("counts added items", () => {
  cart.add(item1);
  expect(cart.count()).toBe(1);
});
it("applies discount to total", () => {
  cart.add(item1);
  cart.applyDiscount(discount);
  expect(cart.total()).toBe(80);
});
it("empties on checkout", () => {
  cart.add(item1);
  cart.checkout();
  expect(cart.isEmpty()).toBe(true);
});
```

Multi-act tests are usually a sign you're testing implementation flow rather than discrete behaviors.

---

## Test Doubles

| Double | Purpose | Use when |
|--------|---------|----------|
| **Dummy** | Object passed but not used | Required by signature, irrelevant |
| **Stub** | Returns canned values | Need controlled inputs from a dep |
| **Spy** | Records invocations | Want to verify method was called (without dictating return) |
| **Mock** | Verifies expected interactions | The interaction itself is the contract |
| **Fake** | Working in-memory implementation | Reusable substitute for infrastructure |

### Dummy

```typescript
const dummyLogger = {} as Logger; // never called in this test
new UserService(realRepo, dummyLogger);
```

### Stub

```typescript
const stubRepo: UserRepo = {
  findById: () => Promise.resolve(aUser().build()),
  save: () => Promise.resolve(),
};
```

### Spy

```typescript
const sentEmails: Array<{ to: string; msg: string }> = [];
const emailSpy = {
  send(to: string, msg: string) { sentEmails.push({ to, msg }); },
};

// later
expect(sentEmails).toHaveLength(1);
expect(sentEmails[0].to).toBe("user@example.com");
```

### Mock

```typescript
const mockRepo = vi.mocked<UserRepo>({} as UserRepo);
mockRepo.save.mockResolvedValue();

await service.register(user);

expect(mockRepo.save).toHaveBeenCalledWith(user);
```

### Fake (preferred default for infra)

```typescript
class InMemoryUserRepo implements UserRepo {
  private users = new Map<string, User>();
  async save(user: User): Promise<void> { this.users.set(user.id.toString(), user); }
  async findById(id: UserId): Promise<User | null> { return this.users.get(id.toString()) ?? null; }
  count(): number { return this.users.size; } // helper for tests
  clear(): void { this.users.clear(); }
}
```

### When to use which (decision tree)

```
Am I testing pure domain logic (value object, entity)?
├── YES → No doubles. Use real objects.
└── NO  ↓
    Is the dep at an infrastructure boundary (DB, HTTP, FS, time, random)?
    ├── YES ↓
    │   Will I use it in 5+ tests?
    │   ├── YES → Build a Fake
    │   └── NO  → Stub it once for this test
    └── NO  ↓
        Is the interaction itself the contract under test?
        ├── YES → Mock — verify interaction
        └── NO  → Use real object
```

**Strong default: real objects + Fakes for infra.** Mocks only when verifying *call* behavior (e.g. "ensure email was sent").

---

## Testing for SOLID

Tests reveal SOLID violations. Use them as design feedback.

### Test reveals SRP violation

**Symptom:** test setup is huge; you instantiate 5 collaborators just to test one behavior.

```typescript
// ❌ — to test order pricing, must set up: payment, shipping, inventory, email, logger
it("calculates premium discount", () => {
  const payment = new Stub();
  const shipping = new Stub();
  const inventory = new Stub();
  const email = new Stub();
  const logger = new Stub();
  const service = new OrderService(payment, shipping, inventory, email, logger);
  // ... actually test pricing
});
```

**Fix:** the unit doing pricing has too many responsibilities. Extract `OrderPricer` and test it in isolation.

### Test reveals OCP violation

**Symptom:** every new payment method requires a new test for the same `if/else` chain.

```typescript
it("handles credit card", () => {});
it("handles paypal", () => {});
it("handles apple pay", () => {});
// each one re-tests the dispatch logic
```

**Fix:** polymorphism. Test each payment class once; the dispatcher is trivial and tested via 1-2 tests.

### Test reveals LSP violation

**Symptom:** `instanceof` checks in test code; tests for subclass don't apply same assertions as parent.

```typescript
// ❌ — subclass behaves differently; tests need to know which subclass
function testRectangle(r: Rectangle) {
  r.setWidth(10);
  r.setHeight(20);
  expect(r.area()).toBe(200);
}

testRectangle(new Rectangle(0, 0));        // passes
testRectangle(new Square(0));              // FAILS — Square forces width=height
```

**Fix:** Square is not a Rectangle. Different shapes, both `Shape`.

### Test reveals ISP violation

**Symptom:** to test class A, you have to provide a fake/stub that implements 10 methods, but A only uses 2.

```typescript
// ❌ — must implement all of `UserRepo`, but `Greeter` only uses `findById`
class Greeter {
  constructor(private readonly users: UserRepo) {}
  greet(id: UserId): string {
    const user = this.users.findById(id);
    return `Hello ${user.name}`;
  }
}
// Test stub must implement findById, save, delete, search, count, ...
```

**Fix:** narrow the interface.

```typescript
interface UserReader { findById(id: UserId): Promise<User | null>; }

class Greeter {
  constructor(private readonly users: UserReader) {} // narrower
}
```

### Test reveals DIP violation

**Symptom:** test cannot run without setting up real DB / network / filesystem.

```typescript
// ❌ — UserService instantiates Postgres directly; can't test without it
class UserService {
  private repo = new PostgresUserRepo(); // hardcoded
}
```

**Fix:** inject the dependency.

```typescript
class UserService {
  constructor(private readonly repo: UserRepository) {}
}
// Test with InMemoryUserRepo; production with PostgresUserRepository
```

---

## Test Builders (data construction patterns)

Make complex test data cheap and readable.

```typescript
class UserBuilder {
  private email = Email.of("default@example.com");
  private age = Age.of(30);
  private premium = false;
  private name = Name.of("Test User");
  private balance = Money.zero();

  withEmail(e: string): this { this.email = Email.of(e); return this; }
  withAge(a: number): this { this.age = Age.of(a); return this; }
  premiumUser(): this { this.premium = true; return this; }
  withBalance(amount: number): this { this.balance = Money.dollars(amount); return this; }

  build(): User {
    return new User(this.email, this.age, this.name, this.premium, this.balance);
  }
}

// Convenience factory
export function aUser(): UserBuilder { return new UserBuilder(); }

// Tests stay readable
const user = aUser().premiumUser().withAge(25).build();
const broke = aUser().withBalance(0).build();
const overdrawn = aUser().withBalance(-50).build(); // if invalid, throws — design the validation
```

**Why builders matter:**
- Domain models change shape; builders centralize defaults
- Tests stay readable when only relevant attributes are set
- Refactors update the builder, not 100 test files

### Object Mother (alternative)

```typescript
const Users = {
  premiumWithFunds: (): User => aUser().premiumUser().withBalance(1000).build(),
  newcomer: (): User => aUser().build(),
  banned: (): User => aUser().banned().build(),
};

// In tests
const user = Users.premiumWithFunds();
```

Object Mother gives **named** prototypical objects. Use builder for arbitrary one-offs; use mother for canonical scenarios.

---

## Contract Tests

Verify ALL implementations satisfy the same contract.

```typescript
function testUserRepoContract(makeRepo: () => UserRepository) {
  describe("UserRepository contract", () => {
    let repo: UserRepository;

    beforeEach(() => { repo = makeRepo(); });

    it("returns null for missing user", async () => {
      const found = await repo.findById(UserId.of("nonexistent"));
      expect(found).toBeNull();
    });

    it("persists and retrieves user", async () => {
      const user = aUser().build();
      await repo.save(user);
      const found = await repo.findById(user.id);
      expect(found).toEqual(user);
    });

    it("updates existing user", async () => {
      const user = aUser().build();
      await repo.save(user);
      user.changeEmail(Email.of("new@example.com"));
      await repo.save(user);

      const found = await repo.findById(user.id);
      expect(found.email).toEqual(Email.of("new@example.com"));
    });
  });
}

// Apply to all implementations
testUserRepoContract(() => new InMemoryUserRepo());
testUserRepoContract(() => new PostgresUserRepo(testDb));
testUserRepoContract(() => new RedisUserRepo(testRedis));
```

Single test file, multiple implementations validated. Catches Liskov violations between implementations.

---

## Test Quality Indicators

Healthy test suite:

| Metric | Target | Symptom of trouble |
|--------|--------|-------------------|
| Suite speed (unit) | < 30s | > 60s = something doing I/O |
| Test failure isolation | One failure = one cause | Cascading failures = shared state |
| Refactor without breaking tests | Yes | Tests break on rename = testing implementation |
| Mutation score | > 80% on critical code | < 60% = false confidence |
| Flaky tests | 0% | Any flake = race condition / hidden time / hidden random |

---

## Common Testing Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Testing implementation | Tests break on internal rename | Test via public API only |
| Too many mocks | "Test passed but production broke" | Use real objects + fakes for infra |
| Shared state between tests | Order-dependent failures | Reset state in `beforeEach` |
| No assertions | False green | Always assert something meaningful |
| Trivial tests (`expect(getX()).toBe(x)`) | Wasted maintenance | Skip or test via behavior |
| Time-dependent tests | Flake at midnight, in different TZ | Inject `now()` seam |
| Random-dependent tests | Occasional failures | Inject `random()` or seed |
| Slow unit tests | Reduced feedback loop | Find I/O leaks; use fakes |
| Snapshot tests as primary | Snapshots updated blindly | Use only for serialization, not behavior |
| Testing third-party libs | Wasted effort | Test integration boundary, not the lib |

---

## Quick Reference

| Concern | Pattern |
|---------|---------|
| Test type ratio | 70% unit / 25% integration / 5% E2E |
| Structure | Arrange-Act-Assert with blank lines |
| Test name | Concrete + domain language |
| Acts per test | One |
| Domain logic | Real objects, no doubles |
| Infrastructure | Fake (in-memory implementation) |
| Interaction protocol | Mock, verify calls |
| External system | Integration test against real (test) instance |
| Critical user flow | One E2E test |
| Test data | Builder + Object Mother |
| Multiple implementations | Contract tests |
| Test reveals SRP/OCP/LSP/ISP/DIP issue | Use as design feedback — refactor production code |
| Mutation score | > 80% on critical code |
| Flake tolerance | 0 |
