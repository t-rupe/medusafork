# Building Saga Pattern with FastEndpoints - Real Implementation

## What FastEndpoints Actually Provides

**What's Built-In:**
- ✅ Job Queues - Async command execution with EF Core/Redis persistence
- ✅ Job Tracking - Progress monitoring via TrackingID
- ✅ Command Bus - Synchronous command execution
- ✅ Event Bus - In-process pub/sub

**What's NOT Built-In:**
- ❌ Saga orchestration
- ❌ Automatic compensation
- ❌ Multi-step workflow coordination
- ❌ Transaction state management
- ❌ Rollback logic

## Building Saga Pattern - The Real Work

We need to BUILD saga orchestration ourselves using FastEndpoints as the foundation.

### Architecture

```
Saga =
  FastEndpoints Job Queues (execution) +
  FastEndpoints Events (failure handling) +
  Custom Saga Coordinator (orchestration) +
  EF Core (state persistence)
```

## Implementation

### 1. Saga State Table

```csharp
public class SagaState
{
    public Guid SagaId { get; set; }
    public string SagaType { get; set; }  // e.g., "CreateOrder"
    public string Status { get; set; }    // Started, Running, Completed, Failed, Compensating, Compensated
    public string CurrentStep { get; set; }
    public int StepNumber { get; set; }
    public int TotalSteps { get; set; }

    // Store which steps completed successfully
    public string CompletedStepsJson { get; set; }  // JSON array of step names
    public List<string> CompletedSteps
    {
        get => JsonSerializer.Deserialize<List<string>>(CompletedStepsJson ?? "[]");
        set => CompletedStepsJson = JsonSerializer.Serialize(value);
    }

    // Store compensation data for each step
    public string CompensationDataJson { get; set; }  // JSON dictionary
    public Dictionary<string, object> CompensationData
    {
        get => JsonSerializer.Deserialize<Dictionary<string, object>>(CompensationDataJson ?? "{}");
        set => CompensationDataJson = JsonSerializer.Serialize(value);
    }

    public DateTime CreatedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
    public string? FailureReason { get; set; }

    // Store the original saga input for retry
    public string InputDataJson { get; set; }
}
```

### 2. Saga Coordinator

```csharp
public class SagaCoordinator
{
    private readonly MedusaDbContext _db;
    private readonly ILogger<SagaCoordinator> _logger;

    public SagaCoordinator(MedusaDbContext db, ILogger<SagaCoordinator> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task<Guid> StartSagaAsync(
        string sagaType,
        int totalSteps,
        object inputData,
        CancellationToken ct)
    {
        var sagaId = Guid.NewGuid();

        var saga = new SagaState
        {
            SagaId = sagaId,
            SagaType = sagaType,
            Status = "Started",
            CurrentStep = "Initializing",
            StepNumber = 0,
            TotalSteps = totalSteps,
            CompletedSteps = new List<string>(),
            CompensationData = new Dictionary<string, object>(),
            CreatedAt = DateTime.UtcNow,
            InputDataJson = JsonSerializer.Serialize(inputData)
        };

        await _db.SagaStates.AddAsync(saga, ct);
        await _db.SaveChangesAsync(ct);

        _logger.LogInformation("Started saga {SagaId} of type {SagaType}", sagaId, sagaType);

        return sagaId;
    }

    public async Task RecordStepCompletionAsync(
        Guid sagaId,
        string stepName,
        int stepNumber,
        object? compensationData,
        CancellationToken ct)
    {
        var saga = await _db.SagaStates.FindAsync(new object[] { sagaId }, ct);
        if (saga == null) throw new InvalidOperationException($"Saga {sagaId} not found");

        var completedSteps = saga.CompletedSteps;
        completedSteps.Add(stepName);
        saga.CompletedSteps = completedSteps;

        if (compensationData != null)
        {
            var compData = saga.CompensationData;
            compData[stepName] = compensationData;
            saga.CompensationData = compData;
        }

        saga.StepNumber = stepNumber;
        saga.CurrentStep = stepName;
        saga.Status = "Running";

        await _db.SaveChangesAsync(ct);

        _logger.LogInformation(
            "Saga {SagaId} completed step {StepNumber}/{TotalSteps}: {StepName}",
            sagaId, stepNumber, saga.TotalSteps, stepName);
    }

    public async Task RecordStepFailureAsync(
        Guid sagaId,
        string stepName,
        string failureReason,
        CancellationToken ct)
    {
        var saga = await _db.SagaStates.FindAsync(new object[] { sagaId }, ct);
        if (saga == null) throw new InvalidOperationException($"Saga {sagaId} not found");

        saga.Status = "Failed";
        saga.CurrentStep = stepName;
        saga.FailureReason = failureReason;

        await _db.SaveChangesAsync(ct);

        _logger.LogError(
            "Saga {SagaId} failed at step {StepName}: {Reason}",
            sagaId, stepName, failureReason);
    }

    public async Task<List<string>> GetCompletedStepsAsync(Guid sagaId, CancellationToken ct)
    {
        var saga = await _db.SagaStates.FindAsync(new object[] { sagaId }, ct);
        return saga?.CompletedSteps ?? new List<string>();
    }

    public async Task<Dictionary<string, object>> GetCompensationDataAsync(Guid sagaId, CancellationToken ct)
    {
        var saga = await _db.SagaStates.FindAsync(new object[] { sagaId }, ct);
        return saga?.CompensationData ?? new Dictionary<string, object>();
    }

    public async Task MarkSagaCompletedAsync(Guid sagaId, CancellationToken ct)
    {
        var saga = await _db.SagaStates.FindAsync(new object[] { sagaId }, ct);
        if (saga == null) return;

        saga.Status = "Completed";
        saga.CompletedAt = DateTime.UtcNow;
        await _db.SaveChangesAsync(ct);

        _logger.LogInformation("Saga {SagaId} completed successfully", sagaId);
    }

    public async Task MarkSagaCompensatingAsync(Guid sagaId, CancellationToken ct)
    {
        var saga = await _db.SagaStates.FindAsync(new object[] { sagaId }, ct);
        if (saga == null) return;

        saga.Status = "Compensating";
        await _db.SaveChangesAsync(ct);
    }

    public async Task MarkSagaCompensatedAsync(Guid sagaId, CancellationToken ct)
    {
        var saga = await _db.SagaStates.FindAsync(new object[] { sagaId }, ct);
        if (saga == null) return;

        saga.Status = "Compensated";
        saga.CompletedAt = DateTime.UtcNow;
        await _db.SaveChangesAsync(ct);

        _logger.LogInformation("Saga {SagaId} compensated successfully", sagaId);
    }
}
```

### 3. Saga Step Pattern

Each saga step is a job with explicit compensation handling:

```csharp
// Step 1: Create Order
public class CreateOrderStep : ICommand
{
    public Guid SagaId { get; set; }
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class CreateOrderStepHandler : ICommandHandler<CreateOrderStep>
{
    private readonly IOrderService _orderService;
    private readonly SagaCoordinator _sagaCoordinator;
    private readonly ILogger<CreateOrderStepHandler> _logger;

    public async Task ExecuteAsync(CreateOrderStep cmd, CancellationToken ct)
    {
        try
        {
            // Execute the business logic
            var order = await _orderService.CreateAsync(new Order
            {
                CustomerId = cmd.CustomerId,
                Items = cmd.Items,
                Status = OrderStatus.Pending
            }, ct);

            // Record success with compensation data
            await _sagaCoordinator.RecordStepCompletionAsync(
                sagaId: cmd.SagaId,
                stepName: "CreateOrder",
                stepNumber: 1,
                compensationData: new { OrderId = order.Id },
                ct: ct
            );

            // Queue the next step
            await new ReserveInventoryStep
            {
                SagaId = cmd.SagaId,
                OrderId = order.Id,
                Items = cmd.Items
            }.QueueJobAsync(ct);
        }
        catch (Exception ex)
        {
            // Record failure
            await _sagaCoordinator.RecordStepFailureAsync(
                cmd.SagaId,
                "CreateOrder",
                ex.Message,
                ct
            );

            // Publish failure event to trigger compensation
            await new SagaFailedEvent
            {
                SagaId = cmd.SagaId,
                FailedStep = "CreateOrder",
                Reason = ex.Message
            }.PublishAsync(ct);

            throw;
        }
    }
}
```

### 4. Saga Orchestrator (Entry Point)

```csharp
public class CreateOrderSagaOrchestrator : ICommandHandler<CreateOrderCommand, SagaResult>
{
    private readonly SagaCoordinator _sagaCoordinator;

    public async Task<SagaResult> ExecuteAsync(CreateOrderCommand cmd, CancellationToken ct)
    {
        // Start the saga
        var sagaId = await _sagaCoordinator.StartSagaAsync(
            sagaType: "CreateOrder",
            totalSteps: 3,  // CreateOrder, ReserveInventory, ChargePayment
            inputData: cmd,
            ct: ct
        );

        // Queue the first step
        await new CreateOrderStep
        {
            SagaId = sagaId,
            CustomerId = cmd.CustomerId,
            Items = cmd.Items
        }.QueueJobAsync(ct);

        // Return immediately with saga ID
        return new SagaResult
        {
            SagaId = sagaId,
            Status = "Started",
            Message = "Order creation saga started. Use the saga ID to track progress."
        };
    }
}
```

### 5. Compensation Handler

```csharp
public class SagaCompensationHandler : IEventHandler<SagaFailedEvent>
{
    private readonly SagaCoordinator _sagaCoordinator;
    private readonly IOrderService _orderService;
    private readonly IInventoryService _inventoryService;
    private readonly IPaymentService _paymentService;
    private readonly ILogger<SagaCompensationHandler> _logger;

    public async Task HandleAsync(SagaFailedEvent evt, CancellationToken ct)
    {
        _logger.LogWarning("Compensating saga {SagaId} after failure at {Step}",
            evt.SagaId, evt.FailedStep);

        await _sagaCoordinator.MarkSagaCompensatingAsync(evt.SagaId, ct);

        // Get completed steps and their compensation data
        var completedSteps = await _sagaCoordinator.GetCompletedStepsAsync(evt.SagaId, ct);
        var compensationData = await _sagaCoordinator.GetCompensationDataAsync(evt.SagaId, ct);

        // Compensate in REVERSE order
        completedSteps.Reverse();

        foreach (var step in completedSteps)
        {
            try
            {
                await CompensateStep(step, compensationData, ct);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to compensate step {Step} for saga {SagaId}",
                    step, evt.SagaId);
                // Continue compensating other steps even if one fails
            }
        }

        await _sagaCoordinator.MarkSagaCompensatedAsync(evt.SagaId, ct);

        // Publish completion event
        await new SagaCompensatedEvent
        {
            SagaId = evt.SagaId,
            OriginalFailureReason = evt.Reason
        }.PublishAsync(ct);
    }

    private async Task CompensateStep(
        string stepName,
        Dictionary<string, object> compensationData,
        CancellationToken ct)
    {
        if (!compensationData.TryGetValue(stepName, out var data))
        {
            _logger.LogWarning("No compensation data found for step {Step}", stepName);
            return;
        }

        var jsonData = JsonSerializer.Serialize(data);

        switch (stepName)
        {
            case "CreateOrder":
                var orderData = JsonSerializer.Deserialize<CreateOrderCompensationData>(jsonData);
                await _orderService.DeleteAsync(orderData.OrderId, ct);
                _logger.LogInformation("Compensated: Deleted order {OrderId}", orderData.OrderId);
                break;

            case "ReserveInventory":
                var inventoryData = JsonSerializer.Deserialize<ReserveInventoryCompensationData>(jsonData);
                await _inventoryService.ReleaseAsync(inventoryData.ReservationId, ct);
                _logger.LogInformation("Compensated: Released inventory reservation {ReservationId}",
                    inventoryData.ReservationId);
                break;

            case "ChargePayment":
                var paymentData = JsonSerializer.Deserialize<ChargePaymentCompensationData>(jsonData);
                await _paymentService.RefundAsync(paymentData.TransactionId, ct);
                _logger.LogInformation("Compensated: Refunded payment {TransactionId}",
                    paymentData.TransactionId);
                break;

            default:
                _logger.LogWarning("Unknown step for compensation: {Step}", stepName);
                break;
        }
    }
}

// Compensation data classes
public class CreateOrderCompensationData
{
    public string OrderId { get; set; }
}

public class ReserveInventoryCompensationData
{
    public string ReservationId { get; set; }
}

public class ChargePaymentCompensationData
{
    public string TransactionId { get; set; }
}
```

### 6. Success Completion Handler

```csharp
public class Step3_ChargePaymentHandler : ICommandHandler<ChargePaymentStep>
{
    public async Task ExecuteAsync(ChargePaymentStep cmd, CancellationToken ct)
    {
        try
        {
            var transaction = await _paymentService.ChargeAsync(cmd.OrderId, cmd.Amount, ct);

            // Record success
            await _sagaCoordinator.RecordStepCompletionAsync(
                cmd.SagaId,
                "ChargePayment",
                3,
                new { TransactionId = transaction.Id },
                ct
            );

            // This was the last step - mark saga as complete
            await _sagaCoordinator.MarkSagaCompletedAsync(cmd.SagaId, ct);

            // Publish success event
            await new SagaCompletedEvent
            {
                SagaId = cmd.SagaId,
                OrderId = cmd.OrderId
            }.PublishAsync(ct);
        }
        catch (Exception ex)
        {
            await _sagaCoordinator.RecordStepFailureAsync(cmd.SagaId, "ChargePayment", ex.Message, ct);
            await new SagaFailedEvent
            {
                SagaId = cmd.SagaId,
                FailedStep = "ChargePayment",
                Reason = ex.Message
            }.PublishAsync(ct);
            throw;
        }
    }
}
```

### 7. Saga Progress Tracking

```csharp
public class GetSagaStatusEndpoint : Endpoint<GetSagaStatusRequest, SagaStatusResponse>
{
    private readonly MedusaDbContext _db;

    public override void Configure()
    {
        Get("/saga/{sagaId:guid}/status");
        AllowAnonymous();
    }

    public override async Task HandleAsync(GetSagaStatusRequest req, CancellationToken ct)
    {
        var saga = await _db.SagaStates.FindAsync(new object[] { req.SagaId }, ct);

        if (saga == null)
        {
            await SendNotFoundAsync(ct);
            return;
        }

        await SendOkAsync(new SagaStatusResponse
        {
            SagaId = saga.SagaId,
            Status = saga.Status,
            CurrentStep = saga.CurrentStep,
            Progress = $"{saga.StepNumber}/{saga.TotalSteps}",
            CompletedSteps = saga.CompletedSteps,
            FailureReason = saga.FailureReason
        }, ct);
    }
}
```

## Key Points

### What We Built (Not in FastEndpoints):
1. ✅ **Saga Coordinator** - Tracks state, steps, compensation data
2. ✅ **Saga State Persistence** - EF Core table for saga state
3. ✅ **Manual Compensation Logic** - Explicit rollback for each step
4. ✅ **Step Chaining** - Each step queues the next one
5. ✅ **Failure Handling** - Events trigger compensation

### What FastEndpoints Provides:
1. ✅ **Job Queue** - Reliable async execution (`.QueueJobAsync()`)
2. ✅ **Job Tracking** - Monitor job progress via TrackingID
3. ✅ **Event Bus** - Decouple failure handling (`.PublishAsync()`)
4. ✅ **Persistence** - EF Core storage provider for jobs

## Comparison to MedusaJS

| Aspect | MedusaJS | Our FastEndpoints Implementation |
|--------|----------|----------------------------------|
| Workflow Definition | `createWorkflow()` DSL | Manual orchestrator command |
| Step Definition | `createStep()` with auto-compensation | Job handlers with explicit try/catch |
| Compensation | Automatic (reverse order) | Manual via event handler |
| State Tracking | Built-in transaction store | Custom `SagaState` table |
| Step Chaining | Implicit (proxied dependencies) | Explicit (queue next job) |
| Failure Handling | Automatic rollback | Manual via `SagaFailedEvent` |

## Honest Assessment

**Pros:**
- ✅ Full control over saga logic
- ✅ Explicit compensation (easier to debug)
- ✅ Uses only free, open-source tools
- ✅ Job persistence handles crashes

**Cons:**
- ❌ More boilerplate than MedusaJS workflows
- ❌ Manual step chaining (error-prone)
- ❌ Compensation logic must be written for each saga
- ❌ No automatic rollback
- ❌ Requires discipline to maintain saga state

**When to Use This:**
- Long-running order processing (hours/days)
- Multi-service coordination
- Need explicit compensation logic
- Want full observability into saga state

**When NOT to Use This:**
- Simple CRUD operations → Use EF Core transactions
- Fast operations (< 5 seconds) → Use command handlers directly
- No failure recovery needed → Use events for fire-and-forget

## References

**FastEndpoints Demo:**
- https://github.com/FastEndpoints/Job-Queue-EF-Core-Demo

**Saga Pattern:**
- https://microservices.io/patterns/data/saga.html
