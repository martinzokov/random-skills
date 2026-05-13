---
name: hex-infra
description: Use when implementing a feature or behavior change with hexagonal architecture (commands, queries, ports, adapters), DDD-informed domain craftsmanship, and optional deep DDD reference material
---

# hex-infra

## Overview

When a feature or behavior change is requested, work through four steps in order. Never skip ahead.

1. **Classify** — Command (mutates state) or Query (reads state)?
2. **Write handler** — application logic only, no HTTP/DB imports
3. **Define ports** — interfaces the handler depends on
4. **Write adapters** — concrete implementations for transport, storage, and external systems

Use this as the default frame for feature work: domain and application code stay separate from frameworks, datastores, transport layers, and vendor SDKs. All external actions should be modeled as either a command or a query, with dependencies expressed as ports and implemented by adapters.

**With TDD:** Use `superpowers:test-driven-development` alongside this skill. Tests drive behavior; hex boundaries keep that behavior testable and isolated from infrastructure.

`src/domain/` is read-only when your repository policy marks it frozen. Otherwise, new domain concepts belong under `src/domain/` following the craftsmanship rules below — never leak infrastructure into that tree.

---

## Domain implementation and software craftsmanship

Domain code should express **business behavior and invariants**, not data shapes alone. Apply these principles in order of priority:

1. **Ubiquitous language** — Names in code match how the business speaks. Avoid generic DTO names (`Data`, `Info`) when a domain term exists.
2. **Behavior-first types** — Prefer methods that enforce rules (`order.addLine(item)`) over setter-heavy structures. Reject anemic models where services do all the work and entities are bags of fields (see anti-pattern coverage in the DDD reference).
3. **Single responsibility** — One type, one reason to change: either a cohesive slice of domain behavior or a clearly bounded value. Split when a class answers two unrelated business questions.
4. **Encapsulation** — Expose intent (`cancel()`, `ship()`) not field mutation. Keep invariants inside the aggregate or entity boundary.
5. **Dependency direction** — Domain depends on nothing outside itself. Application handlers orchestrate domain objects and ports; infrastructure implements ports only.
6. **SOLID as guardrails** — **S** small cohesive modules; **O** extend via new types or ports, not giant switches; **L** substitutable port implementations; **I** narrow port interfaces; **D** depend on abstractions (ports), not concrete adapters.

**Handlers vs domain:** Handlers parse input, load aggregates via repositories, invoke domain behavior, and persist outcomes. Complex rules live in domain types; handlers stay thin orchestration.

**When to read more:** For aggregates, bounded contexts, strategic vs tactical DDD, CQRS tradeoffs, and Microsoft-style implementation patterns, consult [`references/ddd-expert-knowledge-base.md`](references/ddd-expert-knowledge-base.md).

---

## Step 1: Classify

| | Command | Query |
|---|---|---|
| Mutates state? | Yes | No |
| Depends on | Repository (load/save aggregate) | ViewStore (read model) |
| Returns | `Result<E, { id } \| void>` | `Result<E, View>` |

Convention, not a rule - commands may return richer data when genuinely needed.

---

## Step 2: Write the Handler

### Shared types

Define once in `src/application/types.ts`:

```typescript
export type Result<E extends string, A> =
  | { ok: true; value: A }
  | { ok: false; error: E }

export const ok  = <A>(value: A): Result<never, A>            => ({ ok: true, value })
export const err = <E extends string>(e: E): Result<E, never> => ({ ok: false, error: e })

export type CommandHandler<I, O> = (input: I, ctx: AppContext) => Promise<O>
export type QueryHandler<I, O>   = (input: I, ctx: AppContext) => Promise<O>

export type AppContext = Record<string, unknown> // extend as you add ports
```

`AppContext` is the seam between application code and infrastructure. It should carry ports only, never framework request objects, database handles, or SDK clients.

### Command handler

`src/application/commands/CreateOrder.ts`:

```typescript
import type { CommandHandler, Result } from '../types'

export type CreateOrderInput = {
  customerId: string
  items: { productId: string; quantity: number }[]
}
export type CreateOrderOutput = Result<
  'CUSTOMER_NOT_FOUND' | 'INVALID_QUANTITY',
  { orderId: string }
>

export const handler: CommandHandler<CreateOrderInput, CreateOrderOutput> =
  async (input, ctx) => {
    // domain logic only - no HTTP, no DB imports
  }
```

### Query handler

`src/application/queries/GetOrder.ts`:

```typescript
import type { QueryHandler, Result } from '../types'

export type GetOrderInput  = { orderId: string }
export type GetOrderOutput = Result<'ORDER_NOT_FOUND', OrderView>

export const handler: QueryHandler<GetOrderInput, GetOrderOutput> =
  async (input, ctx) => ctx.orderViewStore.byId(input.orderId)
```

---

## Step 3: Define Ports

One file per dependency in `src/ports/`. Interface only - no implementation.

Ports describe capabilities in business terms. They should not leak ORM rows, HTTP request/response objects, SDK result shapes, or other infrastructure-specific types.

```typescript
// src/ports/OrderRepository.ts
export interface OrderRepository {
  load(id: OrderId): Promise<Order | null>
  save(order: Order): Promise<void>
}
```

```typescript
// src/ports/OrderViewStore.ts
export interface OrderViewStore {
  byId(id: string): Promise<OrderView | null>
  search(criteria: OrderSearch): Promise<Page<OrderView>>
}
```

After defining ports, update `AppContext` in `src/application/types.ts`:

```typescript
export type AppContext = {
  orderRepository: OrderRepository
  orderViewStore: OrderViewStore
}
```

Handlers receive ports via `ctx` - never import infrastructure into application code.

---

## Step 4a: Transport Adapter

One file per handler in the appropriate infrastructure boundary. Its jobs are: validate input, call the handler, map the result.

```typescript
// src/infrastructure/http/CreateOrderController.ts

// 1. Validate raw request input (use whatever validation library the project uses)
const input = validate<CreateOrderInput>(request.body)

// 2. Call handler
const result = await handler(input, ctx)

// 3. Map result
if (!result.ok) return httpError(reply, result.error)
return httpSuccess(reply, result.value, 201)
```

No business logic in controllers - ever.

The same rule applies outside HTTP. A queue consumer, job runner, event handler, CLI entrypoint, or UI-triggered action is still just an adapter around a command or query handler.

### Error code -> HTTP status

| Error code pattern | Status |
|---|---|
| `*_NOT_FOUND` | 404 |
| `INVALID_*` | 400 |
| `UNAUTHORIZED` | 401 |
| `FORBIDDEN` | 403 |
| `*_CONFLICT` | 409 |
| `RATE_LIMITED` | 429 |
| anything else | 500 |

Response envelope always: `{ "code": "ERROR_CODE" }`

---

## Step 4b: Storage Or External Adapter

One file per port in the appropriate infrastructure boundary. Implement the interface exactly - no extra methods. Keep mapping functions private to the file.

**Repository (command side)** `src/infrastructure/db/OrderRepository.ts`:

```typescript
import type { OrderRepository } from '../../ports/OrderRepository'

export class OrderDbRepository implements OrderRepository {
  async load(id: OrderId): Promise<Order | null> {
    const row = await /* db query by id */
    return row ? toDomain(row) : null
  }

  async save(order: Order): Promise<void> {
    const row = toRow(order)
    await /* upsert row */
  }
}

// Private - never export
function toDomain(row: unknown): Order { /* DB row -> domain object */ }
function toRow(order: Order): unknown  { /* domain object -> DB row  */ }
```

**ViewStore (query side)** `src/infrastructure/db/OrderViewStore.ts`:

```typescript
import type { OrderViewStore } from '../../ports/OrderViewStore'

export class OrderDbViewStore implements OrderViewStore {
  async byId(id: string): Promise<OrderView | null> {
    const row = await /* db query by id */
    return row ? toView(row) : null
  }

  async search(criteria: OrderSearch): Promise<Page<OrderView>> {
    /* filter + paginate rows, return { items, total } */
  }
}

// Private - never export
function toView(row: unknown): OrderView { /* DB row -> view */ }
```

Apply the same pattern to non-database integrations. A mailer, queue publisher, payment gateway client, filesystem writer, or third-party API client is an adapter that implements a port owned by the application.

---

## Hard Rules

- No framework, datastore, transport, SDK, or vendor imports inside `src/domain/` or `src/application/`
- No business logic inside controllers
- External effects go through ports implemented by adapters
- Commands mutate state and may trigger effectful ports
- Queries read state and must not hide writes
- Port interfaces must not expose infrastructure types (e.g. no raw DB rows as return types)
- Adapters implement the port interface exactly - no extra methods

## Non-HTTP Feature Work

Use the same command/query plus ports/adapters framing even when the entrypoint is not HTTP:

- Background jobs
- Event consumers
- UI-triggered actions
- Cron tasks
- Service-to-service calls

If the operation changes state or causes effects, model it as a command. If it only reads, model it as a query.

## File Layout

```
src/
  domain/                           <- READ ONLY
  application/
    commands/CreateOrder.ts         <- handler + input/output types
    queries/GetOrder.ts
    types.ts                        <- Result, CommandHandler, QueryHandler, AppContext
  ports/
    OrderRepository.ts              <- interface only
    OrderViewStore.ts
  infrastructure/
    http/
      CreateOrderController.ts      <- one per handler
    db/
      OrderRepository.ts            <- one per port
      OrderViewStore.ts
```
