# Idiomatic Stack Blueprints

The pattern folder trees in this knowledge base are language-agnostic architectural abstractions: they define layer roles, cohesion boundaries, and dependency directions — not literal directory names.

When a programming language or framework's idiomatic conventions conflict with generic names, the language conventions win. This document provides concrete, production-ready directory trees for the six major web ecosystems, fully aligned with the unified rules for **`_Shared/`**, **Channels (`Admin/` vs `Public/`)**, and **Action Styles**.

> 💡 **No Hard Naming or Casing Lock-in:**  
> These blueprints show idiomatic adaptations for specific stacks. There is zero hard lock-in to PascalCase or fixed directory names: use `snake_case` in Python, lowercase in Go, `kebab-case` in TypeScript, and feel free to use accepted synonyms (e.g. `UseCases/` for `Application/`, `Interfaces/` for `Ports/`, `_shared/` for `_Shared/`).

---

## 1. Go (Standard & Capability-Driven Layout)

> **Pattern:** Explicit Architecture (Ports & Adapters)  
> **Casing & Idiom:** Short lowercase package names, `internal/` boundary enforced by the Go compiler.

### The Go Anti-Pattern
In Go, naming packages `controllers`, `services`, `models`, or `utils` is considered a major anti-pattern. Such names lead to stuttering imports (`models.User`, `controllers.UserController`), cyclic dependencies, and poor encapsulation.

### Idiomatic Go Architecture (Ports & Adapters)
Go code should be organized by capability or domain in `internal/`, using short lowercase package names. Each capability package contains its domain entity, ports (interfaces), application service, and inbound/outbound adapters:

```text
my-service/
├── cmd/
│   ├── server/
│   │   └── main.go                             # HTTP/gRPC server entry point (wires DI, starts listeners)
│   ├── worker/
│   │   └── main.go                             # Background daemon (Kafka/RabbitMQ consumers, cron jobs)
│   └── cli/
│       └── main.go                             # CLI administrative tool
│
├── internal/                                   # Strictly private application code (enforced by Go compiler)
│   ├── order/                                  # Capability / Domain package
│   │   ├── order.go                            # Pure Domain Entity, Value Objects, business rules
│   │   ├── ports.go                            # Port interfaces: OrderRepository, PaymentGateway
│   │   ├── service.go                          # Application service (orchestration logic)
│   │   ├── handler_http_admin.go               # [Channel] Admin Backoffice HTTP handlers
│   │   ├── handler_http_public.go              # [Channel] Public Customer HTTP handlers
│   │   ├── handler_grpc.go                     # Inbound gRPC server implementation
│   │   ├── handler_consumer.go                 # Inbound async consumer (Kafka/RabbitMQ subscriber)
│   │   ├── job_expire_orders.go                # Background scheduled task
│   │   ├── repository_postgres.go              # Database adapter (implements OrderRepository)
│   │   └── dto.go                              # Request/Response data transfer structs
│   │
│   ├── customer/                               # Isolated customer domain package
│   │   ├── customer.go
│   │   ├── ports.go
│   │   ├── service.go
│   │   └── handler_http.go
│   │
│   └── platform/                               # Cross-cutting non-business platform tooling
│       ├── database/                           # DB connection pool initialization (pgx / gorm)
│       ├── telemetry/                          # OpenTelemetry, structured logging (slog / zap)
│       └── middleware/                         # HTTP authentication, panic recovery, CORS
│
├── api/                                        # Public API contracts (OpenAPI specs, Protobuf definitions)
├── migrations/                                 # SQL migrations (golang-migrate / goose)
├── go.mod
└── go.sum
```

---

## 2. TypeScript / Node.js

### 2.1 Fullstack Next.js (App Router + Vertical Feature Slices)

> **Pattern:** Structured Modules (Vertical Slices)  
> **Channels:** Native App Router Route Groups `app/(public)/` and `app/(admin)/`  
> **Action Style:** Style 1 (Feature Subfolders) with `_Shared/`

Modern Next.js unifies UI rendering, Server Components, and Server Actions. Routing directories in `src/app/` act strictly as the **Delivery Layer** (URL resolution, layouts, route guards), while all business logic, UI components, and state mutations live in cohesive feature modules in `src/modules/`:

```text
src/
├── app/                                        # ── DELIVERY LAYER (App Router routing only) ──
│   ├── (public)/                               # [Channel] Public storefront audience (route group)
│   │   ├── products/
│   │   │   ├── page.tsx                        # Server Component (renders ProductGrid)
│   │   │   └── [id]/page.tsx                   # Product details page
│   │   └── cart/page.tsx
│   ├── (admin)/                                # [Channel] Admin backoffice audience (route group)
│   │   ├── products/
│   │   │   ├── page.tsx                        # Admin product list table
│   │   │   ├── new/page.tsx                    # Uses _Shared/Components/ProductForm.tsx
│   │   │   └── [id]/edit/page.tsx              # Uses _Shared/Components/ProductForm.tsx
│   │   └── dashboard/page.tsx
│   ├── api/                                    # Raw REST / Webhook handlers
│   │   └── webhooks/stripe/route.ts
│   └── layout.tsx                              # Root HTML shell
│
├── modules/                                    # ── FEATURE MODULES (Structured Modules / Slices) ──
│   ├── catalog/
│   │   ├── _Shared/                            # Shared within this module (prevents lateral imports)
│   │   │   ├── Components/
│   │   │   │   ├── ProductForm.tsx             # Shared CRUD form component for Create & Edit
│   │   │   │   └── ProductCard.tsx
│   │   │   ├── Schemas/
│   │   │   │   └── product.schema.ts           # Zod validation schemas
│   │   │   └── Dtos/
│   │   │       └── product.dto.ts
│   │   │
│   │   ├── Slices/                             # Vertical Slices for use cases (Style 1)
│   │   │   ├── CreateProduct/
│   │   │   │   └── create-product.action.ts    # Server Action ("use server")
│   │   │   ├── UpdateProduct/
│   │   │   │   └── update-product.action.ts    # Server Action ("use server")
│   │   │   └── DeleteProduct/
│   │   │       └── delete-product.action.ts
│   │   │
│   │   ├── Models/                             # Rich domain model methods & business rules
│   │   │   └── product.ts
│   │   │
│   │   └── Repositories/                       # Database queries (Drizzle / Prisma)
│   │       └── product.repository.ts
│   │
│   └── ordering/
│       ├── _Shared/
│       ├── Slices/
│       └── Repositories/
│
└── shared/                                     # ── GLOBAL CROSS-CUTTING PRIMITIVES ──
    ├── db/                                     # Database client instance & global schema
    │   ├── client.ts
    │   └── schema.ts
    ├── ui/                                     # Reusable design system (Buttons, Modals, Inputs)
    └── lib/                                    # Pure utility functions (formatting, clsx)
```

---

### 2.2 Backend NestJS (Modular Clean Architecture)

> **Pattern:** Explicit Architecture (Clean / Hexagonal)  
> **Channels:** Distinct Controller folders `controllers/admin/` and `controllers/public/`  
> **Action Style:** CQRS Commands & Queries via `@nestjs/cqrs`

NestJS uses TypeScript decorators, dependency injection, and native module encapsulation. In Explicit Architecture, domain entities are kept 100% pure (zero NestJS decorators or ORM annotations), while TypeORM/Prisma models live strictly in `infrastructure/persistence/`:

```text
src/
├── contexts/                                   # Bounded contexts
│   └── ordering/
│       ├── domain/                             # ── PURE DOMAIN (zero NestJS dependencies) ──
│       │   ├── models/order.entity.ts          # Pure domain entity with invariants
│       │   ├── ports/order-repository.port.ts  # Interface (Port)
│       │   └── events/order-placed.event.ts    # Domain Events
│       │
│       ├── application/                        # ── USE CASES & ORCHESTRATION ──
│       │   ├── commands/
│       │   │   ├── create-order.command.ts
│       │   │   └── create-order.handler.ts     # CommandHandler implements ICommandHandler
│       │   ├── queries/
│       │   │   ├── get-order.query.ts
│       │   │   └── get-order.handler.ts        # QueryHandler implements IQueryHandler
│       │   └── dtos/
│       │       └── create-order.dto.ts
│       │
│       ├── infrastructure/                     # ── ADAPTERS (Outbound) ──
│       │   ├── persistence/
│       │   │   ├── order.orm-entity.ts         # TypeORM / Prisma database table mapping
│       │   │   ├── order.mapper.ts             # Maps between ORM entity and Domain entity
│       │   │   └── order-typeorm.repository.ts # Implements OrderRepositoryPort
│       │   └── messaging/
│       │       └── kafka-order-event.publisher.ts
│       │
│       ├── presentation/                       # ── ADAPTERS (Inbound) ──
│       │   ├── controllers/
│       │   │   ├── admin/                      # [Channel] Admin Backoffice API
│       │   │   │   └── admin-order.controller.ts
│       │   │   └── public/                     # [Channel] Public Customer API
│       │   │       └── order.controller.ts
│       │   ├── consumers/                      # Inbound event listeners (RabbitMQ / Kafka / SQS)
│       │   │   └── payment-completed.consumer.ts # @EventPattern() consumer invoking Application Service
│       │   └── jobs/                           # Scheduled background cron workers
│       │       └── cancel-stale-orders.job.ts  # @Cron() worker invoking Application Service
│       │
│       └── ordering.module.ts                  # NestJS module wiring DI: binds Port to Adapter
│
└── app.module.ts                               # Root NestJS module
```

---

## 3. Python (FastAPI Clean Architecture)

> **Pattern:** Explicit Architecture (Ports & Adapters)  
> **Channels:** Endpoint routers grouped in `admin/` and `public/`  
> **Idiom:** Pure `dataclasses` for domain entities, `typing.Protocol` for domain ports, Pydantic for DTOs.

In idiomatic Python Clean Architecture, domain entities are pure standard library `dataclasses` (NOT Pydantic models with validation decorators, which couple domain to validation libraries). Ports are defined using `typing.Protocol` (structural subtyping), and FastAPI's `Depends()` manages dependency injection in the composition root:

```text
src/
└── app/
    ├── domain/                                 # ── PURE DOMAIN LAYER ──
    │   ├── models/                             # Pure dataclasses with encapsulated domain methods
    │   │   ├── order.py                        # @dataclass Order (pure logic, zero I/O)
    │   │   └── customer.py
    │   ├── ports/                              # Interfaces using typing.Protocol
    │   │   ├── repositories.py                 # class OrderRepository(Protocol): ...
    │   │   └── payment_gateway.py
    │   └── exceptions.py                       # Domain exceptions (OrderFinalizedError)
    │
    ├── application/                            # ── USE CASE ORCHESTRATION ──
    │   ├── use_cases/                          # Interactors
    │   │   ├── place_order.py                  # PlaceOrderUseCase
    │   │   └── cancel_order.py
    │   └── dtos/                               # Pydantic schemas for request/response validation
    │       └── order_dtos.py
    │
    ├── infrastructure/                         # ── ADAPTERS (Outbound) ──
    │   ├── db/
    │   │   ├── session.py                      # SQLAlchemy async engine & sessionmaker
    │   │   ├── models.py                       # SQLAlchemy ORM Table mappings (Base)
    │   │   ├── mappers.py                      # Maps SQLAlchemy models <-> Pure Domain Dataclasses
    │   │   └── repositories/
    │   │       └── order_repository_sql.py     # Implements OrderRepository Protocol
    │   └── clients/                            # External HTTP / Stripe clients
    │       └── stripe_payment_gateway.py
    │
    └── presentation/                           # ── ADAPTERS (Inbound) ──
        ├── api/
        │   ├── v1/
        │   │   ├── admin/                      # [Channel] Admin Backoffice Endpoints
        │   │   │   └── orders.py               # APIRouter(prefix="/admin/orders")
        │   │   ├── public/                     # [Channel] Public Customer Endpoints
        │   │   │   └── orders.py               # APIRouter(prefix="/orders")
        │   │   └── dependencies.py             # FastAPI Depends() providers (wires Port to Adapter)
        │   └── error_handlers.py               # Maps domain exceptions to HTTP 4xx/5xx responses
        ├── consumers/                          # Inbound event listeners (FastStream / AIOKafka / RabbitMQ)
        │   └── order_events_consumer.py        # Dispatches consumed messages to Application Use Cases
        ├── jobs/                               # Celery / ARQ / Cron background tasks
        │   └── cancel_expired_orders.py
        └── main.py                             # FastAPI app composition root
```

---

## 4. Java / Kotlin (Multi-Module Gradle / Maven)

> **Pattern:** Explicit Architecture (Compile-Time Ring Isolation)  
> **Channels:** Distinct controller packages `admin/` and `public/` in `:order-web`  
> **Idiom:** Separate Gradle subprojects enforce the Inward Dependency Rule at compile time.

In enterprise Java/Kotlin projects, compile-time layer isolation is enforced through separate Gradle subprojects. The `:order-domain` module has zero external dependencies, making inward dependency leaks impossible to compile:

```text
my-application/
├── order-domain/                               # :order-domain (Zero framework dependencies)
│   └── src/main/java/com/app/order/domain/
│       ├── model/
│       │   ├── Order.java                      # Pure domain entity with invariants
│       │   └── OrderId.java                    # Value Object
│       ├── port/
│       │   └── OrderRepository.java            # Pure interface
│       └── exception/
│           └── OrderFinalizedException.java
│
├── order-application/                          # :order-application (depends on :order-domain)
│   └── src/main/java/com/app/order/application/
│       ├── service/PlaceOrderService.java      # Application orchestrator
│       └── dto/PlaceOrderCommand.java
│
├── order-infrastructure/                       # :order-infrastructure (depends on :order-domain)
│   └── src/main/java/com/app/order/infrastructure/
│       ├── persistence/
│       │   ├── OrderJpaEntity.java             # @Entity annotated DB table
│       │   ├── OrderMapper.java                # Maps between JpaEntity and Domain model
│       │   └── JpaOrderRepository.java         # Implements OrderRepository port
│       └── messaging/
│           └── KafkaOrderEventPublisher.java
│
├── order-web/                                  # :order-web (depends on :order-application)
│   └── src/main/java/com/app/order/web/
│       ├── admin/                              # [Channel] Admin Backoffice REST Controllers
│       │   └── AdminOrderRestController.java
│       └── public/                             # [Channel] Public Customer REST Controllers
│           └── OrderRestController.java
│
└── app-bootstrap/                              # :app-bootstrap (depends on all, wires Spring context)
    └── src/main/java/com/app/
        └── Application.java                    # @SpringBootApplication entry point
```

---

## 5. C# / .NET (Solution-Based Clean Architecture)

> **Pattern:** Explicit Architecture (Clean Architecture with MediatR)  
> **Channels:** Separate API Controllers for `Admin/` and `Public/`  
> **Action Style:** Style 1 (Feature Subfolders via Commands & Queries)

In .NET, architectural layers are compiled as separate `.csproj` projects within a solution (`.sln`). Features are organized in **Style 1: Feature Subfolders** with dedicated `Command`, `Handler`, and `Validator` classes:

```text
MyApplication.sln
├── src/
│   ├── MyApplication.Domain/                   # Class library (Zero NuGet packages except core abstractions)
│   │   ├── Entities/Order.cs                   # Pure domain entity with encapsulation
│   │   ├── ValueObjects/Money.cs               # Immutable Value Object
│   │   ├── Interfaces/IOrderRepository.cs      # Domain Repository Port
│   │   └── Exceptions/DomainException.cs
│   │
│   ├── MyApplication.Application/              # Class library (references Domain)
│   │   ├── Orders/
│   │   │   ├── Commands/
│   │   │   │   ├── CreateOrder/                # Style 1: Feature Subfolder
│   │   │   │   │   ├── CreateOrderCommand.cs   # IRequest<OrderDto>
│   │   │   │   │   ├── CreateOrderHandler.cs   # IRequestHandler<CreateOrderCommand, OrderDto>
│   │   │   │   │   └── CreateOrderValidator.cs # FluentValidation AbstractValidator
│   │   │   │   └── UpdateOrder/
│   │   │   └── Queries/
│   │   │       └── GetOrderById/
│   │   └── Common/Interfaces/IDateTimeProvider.cs
│   │
│   ├── MyApplication.Infrastructure/           # Class library (references Domain & Application)
│   │   ├── Persistence/
│   │   │   ├── OrderDbContext.cs               # Entity Framework Core DbContext
│   │   │   ├── Configurations/OrderConfig.cs   # IEntityTypeConfiguration<Order>
│   │   │   └── Repositories/OrderRepository.cs # Implements IOrderRepository
│   │   └── Services/EmailSender.cs
│   │
│   └── MyApplication.WebApi/                   # ASP.NET Core Web API (references Application & Infrastructure)
│       ├── Controllers/
│       │   ├── Admin/                          # [Channel] Admin Backoffice API
│       │   │   └── AdminOrdersController.cs
│       │   └── Public/                         # [Channel] Public Storefront API
│       │       └── OrdersController.cs
│       └── Program.cs                          # Composition root: builder.Services.AddScoped<...>()
```

---

## 6. PHP / Laravel (Modular Monolith & Action Pattern)

> **Pattern:** Structured Modules (Vertical Slices via Single-Action Controllers)  
> **Channels:** Controllers separated into `Admin/` and `Public/`  
> **Action Style:** Style 1 (Feature Subfolders / Single-Action Invokable Classes) with `_Shared/`

Standard Laravel projects start with Layered Architecture (`app/Http/Controllers/`, `app/Models/`). For large-scale web platforms, idiomatic Laravel adopts a **Modular Monolith** using Single-Action Classes (Spatie / Lorisleiva Actions) and entity-level `_Shared/` folders for forms:

```text
app/
├── Modules/                                    # ── FEATURE MODULES (Structured Modules) ──
│   ├── Ordering/
│   │   ├── _Shared/                            # Shared within Ordering module (prevents lateral imports)
│   │   │   ├── Forms/                          # Reusable Blade / Livewire form components
│   │   │   │   └── OrderForm.php
│   │   │   └── Dtos/                           # Shared Data Transfer Objects (Spatie Data)
│   │   │       └── OrderBaseData.php
│   │   │
│   │   ├── Actions/                            # Single-Action Use Cases (Style 1)
│   │   │   ├── CreateOrderAction.php           # Encapsulates business execution (__invoke)
│   │   │   ├── UpdateOrderAction.php           # Uses _Shared/Forms/OrderForm.php
│   │   │   └── CancelOrderAction.php
│   │   │
│   │   ├── Models/                             # Eloquent models with domain methods
│   │   │   └── Order.php                       # Invariants, state transitions & business calculations
│   │   │
│   │   ├── Http/                               # Delivery Layer
│   │   │   ├── Controllers/
│   │   │   │   ├── Admin/                      # [Channel] Admin Backoffice
│   │   │   │   │   └── AdminOrderController.php
│   │   │   │   └── Public/                     # [Channel] Public Customer Storefront
│   │   │   │       └── OrderController.php
│   │   │   └── Requests/                       # Form Requests (validation)
│   │   │       ├── CreateOrderRequest.php
│   │   │       └── UpdateOrderRequest.php
│   │   │
│   │   └── Providers/
│   │       └── OrderingServiceProvider.php     # Registers module routes, policies, and events
│   │
│   └── Identity/
│       └── ...
│
└── Support/                                    # ── GLOBAL INFRASTRUCTURE & CROSS-CUTTING ──
    ├── Helpers/
    └── Middleware/
```
