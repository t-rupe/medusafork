# Workflow Engine Analysis - MedusaJS to FastEndpoints

## Purpose
Distributed transaction orchestration system enabling saga pattern for long-running business processes with automatic compensation (rollback) on failures.

## MedusaJS Workflow Architecture

### Core Concepts

**1. Workflow**
Composable, retryable, distributed transaction with compensation logic:
- Step-based execution (sequential/parallel)
- Automatic compensation (Saga pattern)
- Idempotency via transaction IDs
- Async execution with workflow engine
- Type-safe composition

**2. Step**
Atomic unit of work with invoke + compensate:
```typescript
const createProductStep = createStep(
  "create-product",
  async (input, { container }) => {
    const product = await productService.create(input)
    return new StepResponse(product, { productId: product.id })
  },
  async (compensationInput, { container }) => {
    await productService.delete(compensationInput.productId)
  }
)
```

**3. Workflow Composition**
```typescript
const createProductWorkflow = createWorkflow(
  "create-product",
  (input) => {
    const product = createProductStep(input)
    const inventory = reserveInventoryStep(product.id)
    return new WorkflowResponse({ product, inventory })
  }
)
```

## FastEndpoints Mapping Strategy

FastEndpoints doesn't have a direct "workflow engine" but provides **primitives to build equivalent functionality**:

1. **Command Bus** → Synchronous steps
2. **Job Queues** → Asynchronous steps with persistence
3. **Event Bus** → Compensation triggers and cross-cutting concerns

### Pattern 1: Simple Workflows = Command Chains

For **synchronous, short-lived workflows**, use Command Bus:

```csharp
// Medusa Workflow
const workflow = createWorkflow("create-product", (input) => {
  const product = createProductStep(input)
  const indexed = indexProductStep(product)
  return new WorkflowResponse(indexed)
})

// FastEndpoints Equivalent
public class CreateProductCommand : ICommand<Product>
{
    public string Title { get; set; }
    public decimal Price { get; set; }
}

public class CreateProductHandler : ICommandHandler<CreateProductCommand, Product>
{
    public async Task<Product> ExecuteAsync(
        CreateProductCommand cmd,
        CancellationToken ct)
    {
        // Step 1: Create product
        var product = await _productService.CreateAsync(new Product
        {
            Title = cmd.Title,
            Price = cmd.Price
        }, ct);

        // Step 2: Index product (via event - decoupled)
        await new ProductCreatedEvent
        {
            ProductId = product.Id
        }.PublishAsync(Mode.WaitForNone, ct);

        return product;
    }
}

// Event handler for indexing (automatic, decoupled)
public class IndexProductHandler : IEventHandler<ProductCreatedEvent>
{
    public async Task HandleAsync(ProductCreatedEvent evt, CancellationToken ct)
    {
        await _searchService.IndexAsync(evt.ProductId, ct);
    }
}
```

**Key Points:**
- No explicit workflow definition needed
- Command handler executes steps sequentially
- Events decouple side effects (indexing, notifications)
- No compensation for simple cases (let DB transactions handle it)

### Pattern 2: Saga Workflows = Job Queues + Events

For **long-running, distributed workflows** with compensation:

```csharp
// Medusa Workflow (multi-step with compensation)
const createOrderWorkflow = createWorkflow("create-order", (input) => {
  const order = createOrderStep(input)
  const inventory = reserveInventoryStep(order) // Can fail
  const payment = chargePaymentStep(order)      // Can fail
  return new WorkflowResponse({ order, inventory, payment })
})

// FastEndpoints Equivalent with Saga Pattern
```

#### Step 1: Orchestrator Command

```csharp
public class CreateOrderCommand : ICommand<OrderResult>
{
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal Total { get; set; }
}

public class CreateOrderHandler : ICommandHandler<CreateOrderCommand, OrderResult>
{
    public async Task<OrderResult> ExecuteAsync(
        CreateOrderCommand cmd,
        CancellationToken ct)
    {
        var sagaId = Guid.NewGuid().ToString();

        // Step 1: Create order (synchronous)
        var order = await _orderService.CreateAsync(new Order
        {
            CustomerId = cmd.CustomerId,
            Items = cmd.Items,
            Total = cmd.Total,
            Status = OrderStatus.Pending,
            SagaId = sagaId
        }, ct);

        // Step 2: Queue inventory reservation (async, can fail)
        await new ReserveInventoryJob
        {
            SagaId = sagaId,
            OrderId = order.Id,
            Items = cmd.Items
        }.QueueJobAsync(ct);

        // Don't wait - return immediately
        return new OrderResult
        {
            OrderId = order.Id,
            Status = "Processing",
            SagaId = sagaId
        };
    }
}
```

#### Step 2: Job Handlers with Failure Events

```csharp
// Job 1: Reserve Inventory
public class ReserveInventoryJob : ICommand
{
    public string SagaId { get; set; }
    public string OrderId { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class ReserveInventoryJobHandler : ICommandHandler<ReserveInventoryJob>
{
    public async Task ExecuteAsync(ReserveInventoryJob job, CancellationToken ct)
    {
        try
        {
            // Reserve inventory
            await _inventoryService.ReserveAsync(job.Items, ct);

            // Success - queue next job (payment)
            await new ChargePaymentJob
            {
                SagaId = job.SagaId,
                OrderId = job.OrderId,
                Amount = job.Items.Sum(i => i.Price * i.Quantity)
            }.QueueJobAsync(ct);
        }
        catch (Exception ex)
        {
            // Failure - trigger compensation
            await new SagaFailedEvent
            {
                SagaId = job.SagaId,
                OrderId = job.OrderId,
                FailedStep = "ReserveInventory",
                Reason = ex.Message
            }.PublishAsync(ct);

            throw; // Re-throw to mark job as failed
        }
    }
}

// Job 2: Charge Payment
public class ChargePaymentJob : ICommand
{
    public string SagaId { get; set; }
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
}

public class ChargePaymentJobHandler : ICommandHandler<ChargePaymentJob>
{
    public async Task ExecuteAsync(ChargePaymentJob job, CancellationToken ct)
    {
        try
        {
            await _paymentService.ChargeAsync(job.OrderId, job.Amount, ct);

            // Success - saga complete
            await new SagaCompletedEvent
            {
                SagaId = job.SagaId,
                OrderId = job.OrderId
            }.PublishAsync(ct);
        }
        catch (Exception ex)
        {
            // Failure - trigger compensation
            await new SagaFailedEvent
            {
                SagaId = job.SagaId,
                OrderId = job.OrderId,
                FailedStep = "ChargePayment",
                Reason = ex.Message
            }.PublishAsync(ct);

            throw;
        }
    }
}
```

#### Step 3: Compensation Event Handlers

```csharp
// Compensation Handler
public class SagaCompensationHandler : IEventHandler<SagaFailedEvent>
{
    public async Task HandleAsync(SagaFailedEvent evt, CancellationToken ct)
    {
        _logger.LogWarning(
            "Saga {SagaId} failed at step {Step}. Compensating...",
            evt.SagaId,
            evt.FailedStep);

        // Get saga state
        var order = await _orderService.GetBySagaIdAsync(evt.SagaId, ct);

        // Compensate based on which step failed
        switch (evt.FailedStep)
        {
            case "ReserveInventory":
                // Nothing to compensate yet
                await _orderService.CancelAsync(order.Id, ct);
                break;

            case "ChargePayment":
                // Need to release inventory
                await _inventoryService.ReleaseAsync(order.Id, ct);
                await _orderService.CancelAsync(order.Id, ct);
                break;
        }

        // Notify customer
        await new OrderCancelledEvent
        {
            OrderId = order.Id,
            Reason = evt.Reason
        }.PublishAsync(ct);
    }
}

// Success Handler
public class SagaCompletionHandler : IEventHandler<SagaCompletedEvent>
{
    public async Task HandleAsync(SagaCompletedEvent evt, CancellationToken ct)
    {
        // Mark order as complete
        await _orderService.CompleteAsync(evt.OrderId, ct);

        // Send confirmation email
        await new OrderConfirmedEvent
        {
            OrderId = evt.OrderId
        }.PublishAsync(ct);
    }
}
```

### Pattern 3: Explicit Saga State Machine

For **complex sagas with many states**, track saga state explicitly:

```csharp
// Saga State Entity
public class OrderSaga
{
    public string SagaId { get; set; }
    public string OrderId { get; set; }
    public SagaStatus Status { get; set; }
    public string CurrentStep { get; set; }
    public Dictionary<string, bool> CompletedSteps { get; set; } = new();
    public DateTime CreatedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
}

public enum SagaStatus
{
    Started,
    ReservingInventory,
    InventoryReserved,
    ChargingPayment,
    Completed,
    Failed,
    Compensating,
    Compensated
}

// Saga Coordinator (injected into job handlers)
public class SagaCoordinator
{
    public async Task UpdateSagaStateAsync(
        string sagaId,
        SagaStatus status,
        string step,
        CancellationToken ct)
    {
        var saga = await _db.Sagas.FindAsync(sagaId, ct);
        saga.Status = status;
        saga.CurrentStep = step;
        saga.CompletedSteps[step] = true;
        await _db.SaveChangesAsync(ct);
    }

    public async Task<List<string>> GetCompletedStepsAsync(
        string sagaId,
        CancellationToken ct)
    {
        var saga = await _db.Sagas.FindAsync(sagaId, ct);
        return saga.CompletedSteps.Where(kvp => kvp.Value)
                                   .Select(kvp => kvp.Key)
                                   .ToList();
    }
}

// Updated job handler with state tracking
public class ReserveInventoryJobHandler : ICommandHandler<ReserveInventoryJob>
{
    private readonly SagaCoordinator _sagaCoordinator;

    public async Task ExecuteAsync(ReserveInventoryJob job, CancellationToken ct)
    {
        await _sagaCoordinator.UpdateSagaStateAsync(
            job.SagaId,
            SagaStatus.ReservingInventory,
            "ReserveInventory",
            ct);

        try
        {
            await _inventoryService.ReserveAsync(job.Items, ct);

            await _sagaCoordinator.UpdateSagaStateAsync(
                job.SagaId,
                SagaStatus.InventoryReserved,
                "ReserveInventory",
                ct);

            // Queue next step
            await new ChargePaymentJob { ... }.QueueJobAsync(ct);
        }
        catch (Exception ex)
        {
            await _sagaCoordinator.UpdateSagaStateAsync(
                job.SagaId,
                SagaStatus.Failed,
                "ReserveInventory",
                ct);

            await new SagaFailedEvent { ... }.PublishAsync(ct);
            throw;
        }
    }
}

// Compensation with state awareness
public class SagaCompensationHandler : IEventHandler<SagaFailedEvent>
{
    public async Task HandleAsync(SagaFailedEvent evt, CancellationToken ct)
    {
        var completedSteps = await _sagaCoordinator
            .GetCompletedStepsAsync(evt.SagaId, ct);

        // Compensate in reverse order
        if (completedSteps.Contains("ChargePayment"))
            await _paymentService.RefundAsync(evt.OrderId, ct);

        if (completedSteps.Contains("ReserveInventory"))
            await _inventoryService.ReleaseAsync(evt.OrderId, ct);

        await _orderService.CancelAsync(evt.OrderId, ct);

        await _sagaCoordinator.UpdateSagaStateAsync(
            evt.SagaId,
            SagaStatus.Compensated,
            "Compensation",
            ct);
    }
}
```

## Job Queue Setup with EF Core

### 1. Job Record Entity

```csharp
public class JobRecord : IJobStorageRecord
{
    public Guid ID { get; set; }
    public DateTime ExecuteAfter { get; set; }
    public DateTime ExpireOn { get; set; }
    public bool IsComplete { get; set; }
    public string QueueID { get; set; }
    public byte[] CommandBytes { get; set; }
    public string CommandTypeName { get; set; }
    public Guid TrackingID { get; set; }
}
```

### 2. DbContext Configuration

```csharp
public class MedusaDbContext : DbContext
{
    public DbSet<JobRecord> JobRecords { get; set; }
    public DbSet<OrderSaga> Sagas { get; set; }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.Entity<JobRecord>(e =>
        {
            e.HasKey(j => j.ID);
            e.HasIndex(j => new { j.QueueID, j.ExecuteAfter, j.IsComplete });
            e.HasIndex(j => j.ExpireOn);
        });

        builder.Entity<OrderSaga>(e =>
        {
            e.HasKey(s => s.SagaId);
            e.Property(s => s.CompletedSteps).HasColumnType("jsonb");
        });
    }
}
```

### 3. Job Storage Provider

```csharp
public class EfCoreJobStorageProvider : IJobStorageProvider<JobRecord>
{
    private readonly IDbContextFactory<MedusaDbContext> _factory;

    public EfCoreJobStorageProvider(IDbContextFactory<MedusaDbContext> factory)
    {
        _factory = factory;
    }

    public async Task StoreJobAsync(JobRecord job, CancellationToken ct)
    {
        await using var db = await _factory.CreateDbContextAsync(ct);
        await db.JobRecords.AddAsync(job, ct);
        await db.SaveChangesAsync(ct);
    }

    public async Task<IEnumerable<JobRecord>> GetNextBatchAsync(
        PendingSearchParams<JobRecord> p)
    {
        await using var db = await _factory.CreateDbContextAsync(p.CancellationToken);
        return await db.JobRecords
            .Where(p.Match)
            .OrderBy(j => j.ExecuteAfter)
            .Take(p.Limit)
            .ToListAsync(p.CancellationToken);
    }

    public async Task MarkJobAsCompleteAsync(JobRecord job, CancellationToken ct)
    {
        await using var db = await _factory.CreateDbContextAsync(ct);
        job.IsComplete = true;
        db.JobRecords.Update(job);
        await db.SaveChangesAsync(ct);
    }

    public async Task OnHandlerExecutionFailureAsync(
        JobRecord job,
        Exception ex,
        CancellationToken ct)
    {
        await using var db = await _factory.CreateDbContextAsync(ct);

        // Retry after 1 minute
        job.ExecuteAfter = DateTime.UtcNow.AddMinutes(1);
        db.JobRecords.Update(job);
        await db.SaveChangesAsync(ct);
    }

    public async Task PurgeStaleJobsAsync(
        StaleJobSearchParams<JobRecord> p)
    {
        await using var db = await _factory.CreateDbContextAsync(p.CancellationToken);
        var staleJobs = db.JobRecords.Where(p.Match);
        db.JobRecords.RemoveRange(staleJobs);
        await db.SaveChangesAsync(p.CancellationToken);
    }
}
```

## Comparison Table

| Aspect | MedusaJS Workflows | FastEndpoints |
|--------|-------------------|---------------|
| **Simple Workflows** | Workflow + Steps | Command handlers |
| **Long-Running** | Workflow Engine + Redis | Job Queues + EF Core/Redis |
| **Compensation** | Automatic (reverse order) | Manual (event-driven) |
| **State Tracking** | Built-in transaction store | Custom saga state table |
| **Idempotency** | Transaction ID | Job Tracking ID |
| **Parallelization** | `parallelize()` helper | Multiple job queues |
| **Nested Workflows** | `runAsStep()` | Nested commands/jobs |
| **Type Safety** | TypeScript inference | C# strong typing |

## Implementation Recommendations

### For Simple Workflows:
✅ Use **Command Bus** only
- Fast, in-process
- No persistence overhead
- Perfect for < 5 second operations

### For Complex Sagas:
✅ Use **Job Queues + Events + Saga State**
- Persistent, resilient
- Explicit compensation
- Good observability

### For Distributed Systems:
✅ Use **Job Queues + Redis Pub/Sub**
- Scale across multiple instances
- Reliable message delivery
- External event integration

## Example: Complete Create Order Saga

See full example in: `ARCHITECTURE_OVERVIEW.md` section "Implementing Saga Pattern with FastEndpoints"

## Key Takeaways

1. **No need for MassTransit/Hangfire** - FastEndpoints provides all primitives
2. **Sagas are explicit** - You control the compensation logic
3. **State tracking is manual** - But simple with a saga state table
4. **Job queues provide durability** - Failed steps can be retried
5. **Events provide decoupling** - Compensation and cross-cutting concerns
6. **Commands provide synchronicity** - Fast paths don't need jobs

## References

**FastEndpoints Docs:**
- https://fast-endpoints.com/docs/command-bus
- https://fast-endpoints.com/docs/event-bus
- https://fast-endpoints.com/docs/job-queues
- https://github.com/FastEndpoints/Job-Queue-EF-Core-Demo

**MedusaJS Workflow Files:**
- `workflows-sdk/src/utils/composer/create-workflow.ts`
- `workflows-sdk/src/utils/composer/create-step.ts`
- `orchestration/src/transaction/transaction-orchestrator.ts`
