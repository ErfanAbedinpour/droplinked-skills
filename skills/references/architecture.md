# Architecture Patterns Reference

## Table of Contents
- [UseCase Pattern Details](#usecase-pattern-details)
- [Repository Pattern](#repository-pattern)
- [Saga Pattern](#saga-pattern)
- [Service Documentation](#service-documentation)
- [Logging and Error Handling](#logging-and-error-handling)

---

## UseCase Pattern Details

### Breaking Down Complex Logic

Each UseCase `doExecute` should read like a table of contents:

```typescript
async doExecute(req: Request): Promise<Response> {
  // 1. Initial Validation
  const context = await this.validateAndFetchContext(req);

  // 2. Perform Calculations or Core Logic
  const calculationResult = this.performCoreLogic(context);

  // 3. Persist Changes to Database
  await this.persistChanges(context.cart.id, calculationResult);

  // 4. Return Final Result
  return this.fetchFinalResult(context.cart.id);
}
```

### Using Internal Interfaces

Define precise interfaces for data flow between private methods:

```typescript
// At the top of the UseCase file
interface ExecutionContext {
  cart: CartV2;
  product: Product;
  shopSettings: ShopSettings;
}

interface CalculationResult {
  finalPrice: number;
  discount: number;
}

// Inside the UseCase class
private async validateAndFetchContext(req: Request): Promise<ExecutionContext> { ... }
private performCoreLogic(ctx: ExecutionContext): CalculationResult { ... }
```

**Benefits:**
- Each small method is testable
- Clear input/output contracts
- Reduces bugs from ambiguous data shapes

### Data Access in UseCases

**Never** call `this.prisma.model.find...` directly in `doExecute`. Wrap in semantic private methods:

```typescript
// WRONG: In doExecute
const cart = await this.prisma.cart.findUnique({
  where: { id },
  include: { items: true },
});

// CORRECT: In doExecute
const cart = await this.fetchCartWithItems(id);

// Private method at bottom of file
private async fetchCartWithItems(id: string) {
  return this.prisma.cart.findUnique({
    where: { id },
    include: { items: true }
  });
}
```

---

## Repository Pattern

### Structure

```
src/modules/<module-name>/repositories/
├── cart.repository.ts
└── cart-item.repository.ts
```

### Guidelines

1. **Semantic method names**: `findCartForCheckout()`, `updateCartStatus()`, not `findOne()`
2. **Select optimization**: Fetch only needed fields
3. **No generic repository**: Create specific methods per use case

```typescript
@Injectable()
export class CartRepository {
  constructor(private readonly prisma: PrismaService) {}

  /**
   * Finds a cart by ID with items for checkout.
   * @param id - Cart UUID
   * @returns Cart with items or null
   */
  async findCartForCheckout(id: string): Promise<CartWithItems | null> {
    return this.prisma.cartV2.findUnique({
      where: { id },
      include: { items: true },
    });
  }

  async updateStatus(id: string, status: CartStatus): Promise<CartV2> {
    return this.prisma.cartV2.update({
      where: { id },
      data: { status },
    });
  }
}
```

### Performance Best Practices

```typescript
// 1. Select only what you need
this.prisma.product.findUnique({
  where: { id },
  select: { id: true, price: true },
});

// 2. Use findUnique for indexed lookups (not findFirst)
this.prisma.order.findUnique({ where: { orderNumber } });

// 3. Parallel independent queries
const [user, cart] = await Promise.all([
  this.fetchUser(userId),
  this.fetchCart(cartId),
]);

// 4. Ensure indexes on frequently queried fields
// In schema.prisma: @@index([shopId, status])
```

---

## Saga Pattern

For multi-step operations with external APIs (Stripe, EasyPost), use Saga pattern with compensation.

### Location

`src/common/saga/saga.base.ts` and `src/modules/order-v2/saga/`

### Structure

Each step has `execute()` + `compensate()` for rollback:

```typescript
interface SagaStep<TContext> {
  name: string;
  execute(context: TContext): Promise<void>;
  compensate(context: TContext): Promise<void>;
}

class OrderCreationSaga extends Saga<OrderContext> {
  steps = [
    {
      name: 'validate-inventory',
      execute: async (ctx) => { /* reserve inventory */ },
      compensate: async (ctx) => { /* release inventory */ },
    },
    {
      name: 'create-payment',
      execute: async (ctx) => { /* charge payment */ },
      compensate: async (ctx) => { /* refund payment */ },
    },
    {
      name: 'create-shipment',
      execute: async (ctx) => { /* book shipment */ },
      compensate: async (ctx) => { /* cancel shipment */ },
    },
  ];
}
```

### When to Use

- Operations involving external APIs (payment, shipping)
- Multi-step transactions requiring atomicity
- Operations where partial completion needs rollback

---

## Service Documentation

Each module needs a `docs.md` documenting exposed services.

### Location

`src/modules/<module-name>/docs.md`

### Format Per Method

```markdown
## createOrder

**Input:**
- `cartId` (string): Cart identifier
- `paymentMethod` (PaymentMethod): Payment details

**Output:**
- `OrderV2`: Created order with items

**Description:**
Creates an order from the given cart, processing payment and reserving inventory.

**Example:**
```typescript
const order = await orderService.createOrder({
  cartId: '507f1f77bcf86cd799439011',
  paymentMethod: { type: 'STRIPE', token: 'tok_xxx' },
});
```
```

---

## Logging and Error Handling

### Contextual Logging

Log at key points with identifiers:

```typescript
// WRONG
console.log('Error happened');

// CORRECT
this.logger.error(`Failed to apply coupon`, {
  cartId: req.cartId,
  coupon: req.code,
  error: e.message,
});
```

### Exception Handling

- Use standard NestJS exceptions: `BadRequestException`, `NotFoundException`, `ForbiddenException`
- Never swallow errors silently
- Messages should tell users "what happened" and developers "why"

```typescript
// Avoid useless try-catch
try {
  await this.doSomething();
} catch (e) {
  // Never do this - swallowing the error
}

// Correct: Let errors bubble or handle meaningfully
if (!cart) {
  throw new NotFoundException(`Cart ${cartId} not found`);
}
```

### Commenting Standards

**JSDoc for public/shared methods:**

```typescript
/**
 * Finds a cart by ID and includes its items.
 * Used primarily during the checkout process.
 *
 * @param id - The UUID of the cart
 * @returns The cart object with items or null if not found
 */
async findCartForCheckout(id: string): Promise<CartWithItems | null> { ... }
```

**Numbered inline comments for UseCase flow:**

```typescript
// 1. Validate input and fetch current state
const context = await this.validate(req);

// 2. Calculate discounts
const result = this.calculate(context);
```
