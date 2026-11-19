# Domain Modeling & Code Examples

These language-agnostic examples illustrate core architectural concepts: **Rich vs. Anemic Models**, **Value Objects**, **Aggregate Roots**, **Ports & Adapters**, and the strict boundary between **Domain Services** and **Application Services**.

When generating `.ai-factory/ARCHITECTURE.md`, adapt these patterns to the project's chosen programming language and framework idioms.

> ⚠️ **Zero Naming & Casing Lock-in (Conceptual Code Notation):**  
> TypeScript is used here strictly as a readable, universally understandable pseudo-code notation. Class names (`OrderId`, `OrderRepositoryPort`), method signatures, and directory paths in comments (`src/ordering/domain/Order.ts`, `Domain/Ports/`) are **purely illustrative architectural roles**.  
> - **In Python:** Translates to `snake_case` files (`order.py`, `order_repository.py`), standard `@dataclass` models, and `typing.Protocol` interfaces.
> - **In Go:** Translates to lowercase packages (`internal/order/`), with ports and domain types co-located in `order.go` and `ports.go`.
> - **In C# / Java:** Translates to `Order.cs`, `IOrderRepository.cs`, or Java records.
> - **File Co-location:** Value Objects (`Money`), Domain Events, and Ports do NOT require dogmatic subdirectories (`ValueObjects/`, `Events/`); in smaller projects or flat package layouts, they can sit directly alongside the entity.

---

## 1. Rich Domain Model vs Anemic Domain Model

### ✅ GOOD: Rich Domain Model (Logic Encapsulated in the Model)
The entity is an **Aggregate Root**: it protects its own invariants, enforces business constraints, and buffers domain events. Outside callers cannot bypass rules or put the entity into an invalid state.

```typescript
// ── DOMAIN LAYER: Aggregate Root ──
class Order {
  private _id: OrderId;
  private _customerId: CustomerId; // Reference by ID only, never by direct Customer object!
  private _items: OrderItem[] = [];
  private _status: OrderStatus = OrderStatus.Draft;
  private _total: Money = Money.zero("USD");
  private _version: number = 1; // Optimistic Concurrency Control (OCC)
  private _domainEvents: DomainEvent[] = [];

  // Factory method: guarantees an Order is never created in an invalid state
  public static create(id: OrderId, customerId: CustomerId): Order {
    const order = new Order(id, customerId, OrderStatus.Draft, Money.zero("USD"), [], 1);
    order._domainEvents.push(new OrderCreatedEvent(id, customerId));
    return order;
  }

  // Reconstitution factory: used strictly by Data Mappers to restore persisted state
  // without re-firing creation events or re-validating historical state transitions
  public static reconstitute(
    id: OrderId,
    customerId: CustomerId,
    status: OrderStatus,
    total: Money,
    items: OrderItem[],
    version: number
  ): Order {
    return new Order(id, customerId, status, total, items, version);
  }

  private constructor(
    id: OrderId,
    customerId: CustomerId,
    status: OrderStatus,
    total: Money,
    items: OrderItem[],
    version: number = 1
  ) {
    this._id = id;
    this._customerId = customerId;
    this._status = status;
    this._total = total;
    this._items = items;
    this._version = version;
  }

  public get version(): number { return this._version; }
    this._items = items;
  }

  // Business behavior encapsulated inside the Aggregate Root
  public addItem(product: Product, quantity: number): void {
    if (this._status !== OrderStatus.Draft) {
      throw new CannotModifyFinalizedOrderException(this._id);
    }
    if (quantity <= 0) {
      throw new InvalidQuantityException("Quantity must be greater than zero");
    }
    if (product.stock < quantity) {
      throw new InsufficientStockException(product.id, quantity);
    }

    // Invariant: If item already exists in order, increment quantity
    const existingIndex = this._items.findIndex(item => item.productId.equals(product.id));
    if (existingIndex >= 0) {
      this._items[existingIndex] = this._items[existingIndex].increaseQuantity(quantity);
    } else {
      this._items.push(new OrderItem(product.id, product.price, quantity));
    }

    this.recalculateTotal(); // State mutation is strictly internal
  }

  public finalize(computedTotal: Money): void {
    if (this._items.length === 0) {
      throw new CannotFinalizeEmptyOrderException(this._id);
    }
    if (this._status !== OrderStatus.Draft) {
      throw new CannotModifyFinalizedOrderException(this._id);
    }

    this._status = OrderStatus.Finalized;
    this._total = computedTotal;
    
    // Domain event buffered for post-transaction dispatch
    this._domainEvents.push(new OrderFinalizedEvent(this._id, this._customerId, this._total));
  }

  private recalculateTotal(): void {
    this._total = this._items.reduce(
      (sum, item) => sum.plus(item.subtotal),
      Money.zero(this._total.currency)
    );
  }

  // Pull-based domain events pattern
  public pullDomainEvents(): DomainEvent[] {
    const events = [...this._domainEvents];
    this._domainEvents = [];
    return events;
  }

  // Read-only accessors
  public get id(): OrderId { return this._id; }
  public get customerId(): CustomerId { return this._customerId; }
  public get status(): OrderStatus { return this._status; }
  public get total(): Money { return this._total; }
  public get items(): readonly OrderItem[] { return [...this._items]; }
}
```

---

### ❌ BAD: Anemic Model + Fat Service (Procedural Anti-Pattern)
The model is reduced to a dumb data structure ("bag of getters and setters"). Business rules leak across various services and controllers, leading to duplication, broken invariants, and untestable code.

```typescript
// ── ANTI-PATTERN: Anemic model (dumb data holder) ──
class Order {
  public id: string;
  public customerId: string;
  public items: OrderItem[] = [];
  public status: string;
  public total: number;
}

// ── ANTI-PATTERN: Fat procedural service ──
class OrderService {
  public addItem(order: Order, product: Product, quantity: number): void {
    // ❌ WRONG: Invariant validation leaked into service layer
    if (order.status !== "DRAFT") {
      throw new Error("Cannot modify finalized order");
    }
    if (product.stock < quantity) {
      throw new Error("Insufficient stock");
    }

    // ❌ WRONG: Outside code manipulates internal entity collection directly
    order.items.push({ productId: product.id, price: product.price, quantity });
    
    // ❌ WRONG: Calculation performed outside the model, prone to calculation drift
    order.total = order.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  }
}
```

---

## 2. Value Objects (Immutability & Structural Equality)

A **Value Object** has no conceptual identity (`id`). It is defined entirely by its attributes, is strictly **immutable**, and self-validates upon construction:

```typescript
// ── DOMAIN LAYER: Value Object ──
class Money {
  private readonly _amount: number; // Stored in minor units (cents) to avoid floating-point errors
  private readonly _currency: string;

  public static cents(amountInCents: number, currency: string): Money {
    if (amountInCents < 0) {
      throw new NegativeMoneyException("Money amount cannot be negative");
    }
    return new Money(amountInCents, currency.toUpperCase());
  }

  public static zero(currency: string = "USD"): Money {
    return new Money(0, currency.toUpperCase());
  }

  private constructor(amountInCents: number, currency: string) {
    this._amount = Math.round(amountInCents);
    this._currency = currency;
  }

  // Immutability: operations always return a new Value Object instance
  public plus(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  public applyDiscountPercentage(percentage: number): Money {
    if (percentage < 0 || percentage > 100) {
      throw new InvalidDiscountException("Discount must be between 0 and 100");
    }
    const discounted = Math.round(this._amount * (1 - percentage / 100));
    return new Money(discounted, this._currency);
  }

  // Structural Equality: two Value Objects are equal if all their fields match
  public equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  public isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount > other._amount;
  }

  private assertSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new CurrencyMismatchException(
        `Cannot operate on different currencies: ${this._currency} and ${other._currency}`
      );
    }
  }

  public get cents(): number { return this._amount; }
  public get currency(): string { return this._currency; }
}
```

---

## 3. Ports and Adapters (Dependency Inversion)

### Step 1: The Domain Layer Defines the Port (Interface)
The domain declares what it needs from the outside world using ubiquitous language. It contains **zero** references to SQL, ORMs, HTTP, or cloud SDKs.

```typescript
// ── DOMAIN LAYER: Domain Port (Domain/Ports/OrderRepositoryPort.ts) ──
interface OrderRepositoryPort {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
  nextIdentity(): OrderId;
}
```

### Step 2: The Infrastructure Layer Implements the Adapter
The adapter handles database driver details, SQL queries, and maps database rows to pure domain entities using the Data Mapper pattern.

```typescript
// ── INFRASTRUCTURE LAYER: Outbound Adapter ──
class PostgresOrderRepository implements OrderRepositoryPort {
  constructor(private readonly db: DatabasePool) {}

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.db.query("SELECT * FROM orders WHERE id = $1", [id.value]);
    if (!row) return null;

    const itemRows = await this.db.query("SELECT * FROM order_items WHERE order_id = $1", [id.value]);
    return OrderMapper.toDomain(row, itemRows);
  }

  async save(order: Order): Promise<void> {
    const record = OrderMapper.toPersistence(order);
    await this.db.transaction(async (tx) => {
      await tx.query(
        "INSERT INTO orders (id, customer_id, status, total_cents, currency) VALUES ($1, $2, $3, $4, $5) " +
        "ON CONFLICT (id) DO UPDATE SET status = $3, total_cents = $4",
        [record.id, record.customer_id, record.status, record.total_cents, record.currency]
      );
      // Synchronize order items...
    });
  }

  nextIdentity(): OrderId {
    return new OrderId(crypto.randomUUID());
  }
}
```

### Step 3: The Composition Root Wires Port to Adapter
The Composition Root (application bootstrap or DI container) binds the port interface to the adapter implementation at startup.

```typescript
// ── COMPOSITION ROOT: DI Wiring (main.ts / app.module.ts) ──
const db = new DatabasePool(config.databaseUrl);
const orderRepository: OrderRepositoryPort = new PostgresOrderRepository(db);

// Injected into application service via constructor
const placeOrderService = new PlaceOrderService(orderRepository, customerRepository, pricingService);
```

---

## 4. Application Service (Use Case Orchestration)

An **Application Service** coordinates a specific business use case. It orchestrates the lifecycle: fetch domain aggregates via ports → invoke pure domain methods → persist updated state via ports → dispatch events. It **never contains domain rules**.

```typescript
// ── APPLICATION LAYER: Input Command DTO ──
interface PlaceOrderCommand {
  readonly orderId: string;
  readonly customerId: string;
}

// ── APPLICATION LAYER: Use Case Orchestrator ──
class PlaceOrderService {
  constructor(
    private readonly orderRepo: OrderRepositoryPort,
    private readonly customerRepo: CustomerRepositoryPort,
    private readonly pricingService: PricingService, // Domain Service
    private readonly eventPublisher: EventPublisherPort,
  ) {}

  async execute(command: PlaceOrderCommand): Promise<void> {
    const orderId = new OrderId(command.orderId);
    const customerId = new CustomerId(command.customerId);

    // 1. Fetch domain aggregates via ports
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw new OrderNotFoundException(orderId);

    const customer = await this.customerRepo.findById(customerId);
    if (!customer) throw new CustomerNotFoundException(customerId);

    // 2. Invoke pure domain logic (Domain Service & Entity methods)
    const finalTotal = this.pricingService.calculateTotal(order.items, customer);
    order.finalize(finalTotal); // The entity enforces its own state transition

    // 3. Persist updated domain state
    await this.orderRepo.save(order);

    // 4. Dispatch domain events (buffered in entity)
    const events = order.pullDomainEvents();
    await this.eventPublisher.publishAll(events);
  }
}
```

---

## 5. Domain Service vs Application Service

| Dimension | Domain Service | Application Service |
|---|---|---|
| **Layer** | `Domain/Services/` | `Application/` (or `Application/Handlers/`) |
| **Responsibility** | Complex business calculations/rules spanning **2+ aggregates** | Use-case orchestration (fetch, call, save, notify) |
| **I/O & Side Effects** | **Zero I/O** (pure CPU calculations, no DB, no network) | Handles I/O via ports (DB, payment gateways, Kafka) |
| **Signatures** | Pure domain objects in → Domain values/objects out | Commands/DTOs in → Result DTOs out |

### ✅ GOOD: Domain Service (Pure Cross-Entity Logic)
When a business rule involves multiple aggregates and does not naturally belong to either:

```typescript
// ── DOMAIN LAYER: Domain Service ──
// Pure business calculation spanning Order + Customer. Zero I/O dependencies.
class PricingService {
  public calculateTotal(items: readonly OrderItem[], customer: Customer): Money {
    const baseTotal = items.reduce(
      (sum, item) => sum.plus(item.subtotal),
      Money.zero(items[0]?.subtotal.currency ?? "USD")
    );

    // Business rule: VIP customers get 10% discount; high-volume orders get 5%
    if (customer.isVip()) {
      return baseTotal.applyDiscountPercentage(10);
    }
    if (baseTotal.isGreaterThan(Money.cents(50000, "USD"))) {
      return baseTotal.applyDiscountPercentage(5);
    }

    return baseTotal;
  }
}
```

### ❌ BAD: Placing Cross-Entity Rules into Application Services
When business rules are placed inside an application service, the domain becomes anemic and business rules cannot be tested without mocking application repositories.

```typescript
// ── ANTI-PATTERN: Business rule disguised as orchestration ──
class PlaceOrderService {
  async execute(command: PlaceOrderCommand) {
    const order = await this.orderRepo.findById(command.orderId);
    const customer = await this.customerRepo.findById(order.customerId);

    // ❌ WRONG: Business rule leaked into Application Service!
    let total = order.subtotal;
    if (customer.tier === "VIP") {
      total = total * 0.9; // Leaked domain calculation!
    }

    order.total = total; // Anemic model mutation
    await this.orderRepo.save(order);
  }
}
```

### Summary: When to Use a Domain Service
- **USE A DOMAIN SERVICE WHEN:**
  - The business operation spans **2 or more distinct aggregates** and does not naturally belong to either.
  - The operation is a **pure business calculation** (no database lookups, no network calls).
  - Placing the rule inside Entity A would create an unnatural dependency on Entity B.
- **DO NOT USE (KEEP LOGIC IN THE ENTITY WHEN):**
  - The rule involves only **one entity's own state** (e.g. `order.addItem()` belongs inside `Order`).
  - You are tempted to create a Domain Service for every single operation — this leads back to anemic procedural scripts.

---

## 6. CQRS & Query Optimization (Read Path vs. Write Path)

In Domain-Driven Design and Explicit Architecture, treating Read operations and Write operations symmetrically is a major anti-pattern. 

### ❌ The Anti-Pattern: Routing Reads Through Aggregate Roots
```text
Client ──> Controller ──> Application Service ──> OrderRepository.findAll()
                                                          │ (Hydrates 100 Order Aggregates)
                                                          ▼
Client ◄── OrderSummaryDto[] ◄── Map 100 Entities to DTOs
```
* **Why this fails in production:**
  1. **N+1 Query Explosion:** Hydrating 100 `Order` aggregate roots fetches hundreds of child `OrderItem` rows and executes Data Mapper transformations.
  2. **Memory & CPU Waste:** Aggregates load business methods, event buffers, and invariants that are 100% useless for read-only rendering.
  3. **Inability to Optimize:** You cannot write clean SQL JOINs, pagination offsets, or DB indexes because the repository interface is locked to returning the pure `Order` entity.

### ✅ The Solution: CQRS (Command Query Responsibility Segregation)

Separate the codebase into **Commands (Write Path)** and **Queries (Read Path)**:

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│ WRITE PATH (Commands): Enforces Business Rules & State Mutations                │
│ Controller ──> Command ──> UseCase Handler ──> Aggregate Root ──> Repository.save()│
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│ READ PATH (Queries): High-Performance Projections, Zero Domain Overhead        │
│ Controller ──> Query ──> Query Handler ──> Direct SQL / Projections ──> View DTO│
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### Read Path Code Example: Direct Database Projection

Queries completely bypass Domain Entities and Repositories, executing raw SQL or optimized ORM projections directly into read-only DTOs:

```typescript
// ── APPLICATION LAYER: Read Query & DTO ──
interface GetCustomerOrdersQuery {
  readonly customerId: string;
  readonly page: number;
  readonly limit: number;
}

interface OrderSummaryDto {
  readonly orderId: string;
  readonly status: string;
  readonly totalFormatted: string;
  readonly itemCount: number;
  readonly placedAt: string;
}

// ── INFRASTRUCTURE / APPLICATION: Optimized Query Handler ──
class GetCustomerOrdersHandler {
  constructor(private readonly db: DatabasePool) {}

  async execute(query: GetCustomerOrdersQuery): Promise<OrderSummaryDto[]> {
    const offset = (query.page - 1) * query.limit;

    // Single optimized SQL query with JOIN and COUNT: zero Domain Aggregates hydrated!
    const rows = await this.db.query<OrderSummaryRow>(
      `SELECT 
         o.id AS order_id, 
         o.status, 
         o.total_cents, 
         o.currency, 
         o.created_at,
         COUNT(i.id) AS item_count
       FROM orders o
       LEFT JOIN order_items i ON i.order_id = o.id
       WHERE o.customer_id = $1
       GROUP BY o.id
       ORDER BY o.created_at DESC
       LIMIT $2 OFFSET $3`,
      [query.customerId, query.limit, offset]
    );

    return rows.map(row => ({
      orderId: row.order_id,
      status: row.status,
      totalFormatted: `$${(row.total_cents / 100).toFixed(2)} ${row.currency}`,
      itemCount: Number(row.item_count),
      placedAt: row.created_at.toISOString(),
    }));
  }
}
```

---

## 7. Validation Boundaries: Syntactic (DTO) vs. Semantic (Domain Invariants)

A common source of confusion in DDD is where validation belongs. Validation operates at two strictly separate boundaries:

| Dimension | Syntactic Validation (Input Validation) | Semantic Validation (Domain Invariants) |
|---|---|---|
| **Location** | **Presentation / Application Layer** (DTO, Form, Router) | **Domain Layer** (Entities, Value Objects) |
| **Tooling** | Zod, Pydantic, FluentValidation, FormRequest | Pure language code (classes, methods, exceptions) |
| **Question Asked** | *"Is the incoming payload structurally well-formed?"* | *"Is this business operation permitted in current state?"* |
| **Scope** | Data types, required fields, string length, regex patterns, email formats, numeric ranges | Business rules, state machine transitions, balances, stock availability, credit limits |
| **Failure Response** | HTTP `400 Bad Request` / `422 Unprocessable Entity` | Domain Exception ➔ mapped to HTTP `409 Conflict` / `422` |

### Code Example: Two-Tier Validation

```typescript
// ── 1. PRESENTATION LAYER: Syntactic Validation (e.g. Zod Schema) ──
import { z } from "zod";

export const AddItemToOrderSchema = z.object({
  productId: z.string().uuid("Invalid product ID format"),
  quantity: z.number().int().positive("Quantity must be at least 1"),
});

export type AddItemToOrderPayload = z.infer<typeof AddItemToOrderSchema>;

// Controller parses HTTP input — fails fast before ever reaching the domain:
export async function addItemAction(req: Request, res: Response) {
  const result = AddItemToOrderSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({ errors: result.error.flatten() });
  }
  // Data is syntactically valid; proceed to Use Case
  await addItemHandler.execute(result.data);
}


// ── 2. DOMAIN LAYER: Semantic Invariant Enforcement ──
class Order {
  // Business invariant: Can never mutate items if order is already paid/shipped
  public addItem(product: Product, quantity: number): void {
    if (this._status !== OrderStatus.Draft) {
      throw new DomainValidationException(
        `Cannot add items to an order with status '${this._status}'. Only Draft orders can be modified.`
      );
    }

    if (product.stock < quantity) {
      throw new InsufficientStockException(
        `Requested ${quantity} units of '${product.name}', but only ${product.stock} units are available.`
      );
    }

    // Proceed with encapsulated mutation...
  }
}
```
