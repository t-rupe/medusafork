# Workflow Engine Analysis - Building on FastEndpoints

## Purpose
Show how MedusaJS implements their workflow engine and how to build an equivalent reusable workflow library using FastEndpoints primitives.

## MedusaJS Workflow Architecture

### What MedusaJS Built

MedusaJS didn't use a 3rd party workflow engine - they **built their own** in these packages:

**1. `workflows-sdk/`** - DSL for defining workflows
```typescript
export function createWorkflow(name, composerFn) { /* ... */ }
export function createStep(name, invokeFn, compensateFn) { /* ... */ }
```

**2. `orchestration/`** - Transaction orchestrator
```typescript
class TransactionOrchestrator {
  async run(transactionId, flow, context) { /* ... */ }
  async compensate(transaction, error) { /* ... */ }
}
```

**3. `workflow-engine-*/`** - Async execution engines
```typescript
class WorkflowEngineService {
  async run(workflowId, { input, transactionId }) { /* ... */ }
  async getStatus(transactionId) { /* ... */ }
}
```

### How It Works

**1. Define Steps:**
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

**2. Compose Workflow:**
```typescript
const workflow = createWorkflow("create-product", (input) => {
  const product = createProductStep(input)
  const indexed = indexProductStep(product)
  return new WorkflowResponse(indexed)
})
```

**3. Execute:**
```typescript
const { result } = await workflow(container).run({ input })
```

**Under the Hood:**
- `createStep()` registers invoke + compensate handlers in a Map
- `createWorkflow()` builds a transaction flow definition
- `.run()` executes steps sequentially via `TransactionOrchestrator`
- On failure, executes compensate functions in reverse order
- State stored in Redis/in-memory for idempotency

See: `workflows-sdk/src/utils/composer/create-workflow.ts`, `orchestration/src/transaction/transaction-orchestrator.ts`

## Building Equivalent in .NET with FastEndpoints

### What We Need to Build

A reusable workflow library providing the same developer experience:

```csharp
// Usage (after we build the library)
var workflow = new WorkflowBuilder<CreateProductInput, Product>()
    .Step("create-product", async (input, ctx) =>
    {
        var product = await ctx.Resolve<IProductService>()
            .CreateAsync(input, ctx.CancellationToken);
        return StepResult.Success(product,
            compensate: async () => await ctx.Resolve<IProductService>()
                .DeleteAsync(product.Id, ctx.CancellationToken));
    })
    .Step("index-product", async (product, ctx) =>
    {
        await ctx.Resolve<ISearchService>()
            .IndexAsync(product.Id, ctx.CancellationToken);
        return StepResult.Success(product);
    })
    .Build("create-product-workflow");

// Execute
var result = await workflow.ExecuteAsync(input, serviceProvider);
```

### Architecture

```
WorkflowBuilder<TInput, TOutput>  - DSL for defining workflows
  ↓
WorkflowDefinition                - Compiled workflow (steps + handlers)
  ↓
WorkflowExecutor                  - Executes workflow with compensation
  ↓
FastEndpoints Job Queues          - Async execution infrastructure
```

## Implementation

### 1. Step Result

```csharp
public class StepResult<T>
{
    public T Output { get; }
    public Func<Task>? CompensateFn { get; }
    public bool IsSuccess { get; }
    public string? ErrorMessage { get; }

    private StepResult(T output, Func<Task>? compensate, bool isSuccess, string? error)
    {
        Output = output;
        CompensateFn = compensate;
        IsSuccess = isSuccess;
        ErrorMessage = error;
    }

    public static StepResult<T> Success(T output, Func<Task>? compensate = null)
        => new(output, compensate, true, null);

    public static StepResult<T> Failure(string error)
        => new(default!, null, false, error);
}
```

### 2. Step Context

```csharp
public class StepContext
{
    private readonly IServiceProvider _serviceProvider;
    public CancellationToken CancellationToken { get; }
    public Guid WorkflowInstanceId { get; }

    public StepContext(
        IServiceProvider serviceProvider,
        Guid workflowInstanceId,
        CancellationToken ct)
    {
        _serviceProvider = serviceProvider;
        WorkflowInstanceId = workflowInstanceId;
        CancellationToken = ct;
    }

    public T Resolve<T>() where T : notnull
        => _serviceProvider.GetRequiredService<T>();
}
```

### 3. Workflow Step

```csharp
public class WorkflowStep<TInput, TOutput>
{
    public string Name { get; }
    public Func<TInput, StepContext, Task<StepResult<TOutput>>> ExecuteFn { get; }

    public WorkflowStep(
        string name,
        Func<TInput, StepContext, Task<StepResult<TOutput>>> executeFn)
    {
        Name = name;
        ExecuteFn = executeFn;
    }
}
```

### 4. Workflow Definition

```csharp
public class WorkflowDefinition<TInput, TOutput>
{
    public string Name { get; }
    private readonly List<object> _steps = new();

    internal WorkflowDefinition(string name)
    {
        Name = name;
    }

    internal void AddStep<TStepInput, TStepOutput>(
        WorkflowStep<TStepInput, TStepOutput> step)
    {
        _steps.Add(step);
    }

    internal IReadOnlyList<object> Steps => _steps;
}
```

### 5. Workflow Builder

```csharp
public class WorkflowBuilder<TInput, TOutput>
{
    private readonly List<object> _steps = new();
    private object? _lastStepOutput;

    public WorkflowBuilder<TInput, TNextOutput> Step<TNextOutput>(
        string name,
        Func<TInput, StepContext, Task<StepResult<TNextOutput>>> executeFn)
        where TInput : TInput  // First step
    {
        var step = new WorkflowStep<TInput, TNextOutput>(name, executeFn);
        _steps.Add(step);
        _lastStepOutput = typeof(TNextOutput);

        return new WorkflowBuilder<TInput, TNextOutput>(_steps);
    }

    public WorkflowBuilder<TInput, TNextOutput> Step<TPrevOutput, TNextOutput>(
        string name,
        Func<TPrevOutput, StepContext, Task<StepResult<TNextOutput>>> executeFn)
    {
        var step = new WorkflowStep<TPrevOutput, TNextOutput>(name, executeFn);
        _steps.Add(step);
        _lastStepOutput = typeof(TNextOutput);

        return new WorkflowBuilder<TInput, TNextOutput>(_steps);
    }

    public WorkflowDefinition<TInput, TOutput> Build(string name)
    {
        var workflow = new WorkflowDefinition<TInput, TOutput>(name);
        foreach (var step in _steps)
        {
            // Add steps dynamically
            var addMethod = typeof(WorkflowDefinition<TInput, TOutput>)
                .GetMethod("AddStep", BindingFlags.NonPublic | BindingFlags.Instance);

            var stepType = step.GetType();
            var genericArgs = stepType.GetGenericArguments();
            var addGenericMethod = addMethod!.MakeGenericMethod(genericArgs);
            addGenericMethod.Invoke(workflow, new[] { step });
        }
        return workflow;
    }

    private WorkflowBuilder(List<object> existingSteps)
    {
        _steps = existingSteps;
    }

    public WorkflowBuilder() { }
}
```

### 6. Workflow Executor

```csharp
public class WorkflowExecutor
{
    private readonly ILogger<WorkflowExecutor> _logger;
    private readonly WorkflowStateStore _stateStore;

    public WorkflowExecutor(
        ILogger<WorkflowExecutor> logger,
        WorkflowStateStore stateStore)
    {
        _logger = logger;
        _stateStore = stateStore;
    }

    public async Task<WorkflowResult<TOutput>> ExecuteAsync<TInput, TOutput>(
        WorkflowDefinition<TInput, TOutput> workflow,
        TInput input,
        IServiceProvider serviceProvider,
        CancellationToken ct = default)
    {
        var instanceId = Guid.NewGuid();
        var context = new StepContext(serviceProvider, instanceId, ct);
        var compensations = new Stack<(string stepName, Func<Task> compensate)>();

        _logger.LogInformation(
            "Starting workflow {WorkflowName} with instance {InstanceId}",
            workflow.Name, instanceId);

        await _stateStore.SaveStateAsync(instanceId, new WorkflowState
        {
            WorkflowName = workflow.Name,
            Status = "Running",
            CurrentStep = 0,
            CreatedAt = DateTime.UtcNow
        }, ct);

        object? currentInput = input;
        int stepNumber = 0;

        try
        {
            foreach (var stepObj in workflow.Steps)
            {
                stepNumber++;

                // Execute step via reflection (or use dynamic)
                var stepType = stepObj.GetType();
                var executeFn = stepType.GetProperty("ExecuteFn")!.GetValue(stepObj);
                var stepName = (string)stepType.GetProperty("Name")!.GetValue(stepObj)!;

                _logger.LogInformation(
                    "Executing step {StepNumber}: {StepName}",
                    stepNumber, stepName);

                await _stateStore.UpdateStepAsync(instanceId, stepNumber, stepName, ct);

                // Invoke the execute function
                var executeMethod = executeFn!.GetType().GetMethod("Invoke");
                var resultTask = (Task)executeMethod!.Invoke(executeFn, new[] { currentInput, context })!;
                await resultTask;

                // Get result
                var resultProperty = resultTask.GetType().GetProperty("Result");
                var stepResult = resultProperty!.GetValue(resultTask);

                var isSuccess = (bool)stepResult!.GetType()
                    .GetProperty("IsSuccess")!.GetValue(stepResult)!;

                if (!isSuccess)
                {
                    var errorMsg = (string?)stepResult.GetType()
                        .GetProperty("ErrorMessage")!.GetValue(stepResult);
                    throw new WorkflowException($"Step '{stepName}' failed: {errorMsg}");
                }

                // Get output for next step
                currentInput = stepResult.GetType().GetProperty("Output")!.GetValue(stepResult);

                // Save compensation function
                var compensateFn = stepResult.GetType()
                    .GetProperty("CompensateFn")!.GetValue(stepResult) as Func<Task>;

                if (compensateFn != null)
                {
                    compensations.Push((stepName, compensateFn));
                }

                _logger.LogInformation("Step {StepName} completed successfully", stepName);
            }

            await _stateStore.CompleteWorkflowAsync(instanceId, ct);

            _logger.LogInformation(
                "Workflow {WorkflowName} completed successfully",
                workflow.Name);

            return WorkflowResult<TOutput>.Success((TOutput)currentInput!);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Workflow {WorkflowName} failed at step {StepNumber}. Starting compensation...",
                workflow.Name, stepNumber);

            await _stateStore.MarkFailedAsync(instanceId, ex.Message, ct);

            // Execute compensations in reverse order
            await CompensateAsync(compensations, instanceId, ct);

            return WorkflowResult<TOutput>.Failure(ex.Message);
        }
    }

    private async Task CompensateAsync(
        Stack<(string stepName, Func<Task> compensate)> compensations,
        Guid instanceId,
        CancellationToken ct)
    {
        await _stateStore.UpdateStatusAsync(instanceId, "Compensating", ct);

        while (compensations.Count > 0)
        {
            var (stepName, compensate) = compensations.Pop();

            try
            {
                _logger.LogWarning("Compensating step: {StepName}", stepName);
                await compensate();
                _logger.LogInformation("Successfully compensated step: {StepName}", stepName);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to compensate step: {StepName}", stepName);
                // Continue compensating other steps
            }
        }

        await _stateStore.UpdateStatusAsync(instanceId, "Compensated", ct);
    }
}
```

### 7. Workflow State Store (using EF Core)

```csharp
public class WorkflowState
{
    public Guid InstanceId { get; set; }
    public string WorkflowName { get; set; } = null!;
    public string Status { get; set; } = null!; // Running, Completed, Failed, Compensated
    public int CurrentStep { get; set; }
    public string? CurrentStepName { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
    public string? ErrorMessage { get; set; }
}

public class WorkflowStateStore
{
    private readonly MedusaDbContext _db;

    public async Task SaveStateAsync(Guid instanceId, WorkflowState state, CancellationToken ct)
    {
        state.InstanceId = instanceId;
        await _db.WorkflowStates.AddAsync(state, ct);
        await _db.SaveChangesAsync(ct);
    }

    public async Task UpdateStepAsync(Guid instanceId, int stepNumber, string stepName, CancellationToken ct)
    {
        var state = await _db.WorkflowStates.FindAsync(new object[] { instanceId }, ct);
        if (state != null)
        {
            state.CurrentStep = stepNumber;
            state.CurrentStepName = stepName;
            await _db.SaveChangesAsync(ct);
        }
    }

    public async Task CompleteWorkflowAsync(Guid instanceId, CancellationToken ct)
    {
        var state = await _db.WorkflowStates.FindAsync(new object[] { instanceId }, ct);
        if (state != null)
        {
            state.Status = "Completed";
            state.CompletedAt = DateTime.UtcNow;
            await _db.SaveChangesAsync(ct);
        }
    }

    public async Task MarkFailedAsync(Guid instanceId, string error, CancellationToken ct)
    {
        var state = await _db.WorkflowStates.FindAsync(new object[] { instanceId }, ct);
        if (state != null)
        {
            state.Status = "Failed";
            state.ErrorMessage = error;
            await _db.SaveChangesAsync(ct);
        }
    }

    public async Task UpdateStatusAsync(Guid instanceId, string status, CancellationToken ct)
    {
        var state = await _db.WorkflowStates.FindAsync(new object[] { instanceId }, ct);
        if (state != null)
        {
            state.Status = status;
            await _db.SaveChangesAsync(ct);
        }
    }
}
```

### 8. Workflow Result

```csharp
public class WorkflowResult<T>
{
    public bool IsSuccess { get; }
    public T? Output { get; }
    public string? ErrorMessage { get; }

    private WorkflowResult(bool isSuccess, T? output, string? error)
    {
        IsSuccess = isSuccess;
        Output = output;
        ErrorMessage = error;
    }

    public static WorkflowResult<T> Success(T output)
        => new(true, output, null);

    public static WorkflowResult<T> Failure(string error)
        => new(false, default, error);
}
```

## Usage Examples

### Simple Workflow

```csharp
public class CreateProductWorkflow
{
    public static WorkflowDefinition<CreateProductInput, Product> Definition { get; }

    static CreateProductWorkflow()
    {
        Definition = new WorkflowBuilder<CreateProductInput, Product>()
            .Step("create-product", async (input, ctx) =>
            {
                var service = ctx.Resolve<IProductService>();
                var product = await service.CreateAsync(new Product
                {
                    Title = input.Title,
                    Price = input.Price
                }, ctx.CancellationToken);

                return StepResult.Success(product,
                    compensate: async () =>
                        await service.DeleteAsync(product.Id, ctx.CancellationToken));
            })
            .Step("publish-event", async (product, ctx) =>
            {
                await new ProductCreatedEvent
                {
                    ProductId = product.Id
                }.PublishAsync(ctx.CancellationToken);

                return StepResult.Success(product);
            })
            .Build("create-product-workflow");
    }
}

// Endpoint
public class CreateProductEndpoint : Endpoint<CreateProductRequest, Product>
{
    private readonly WorkflowExecutor _executor;

    public override async Task HandleAsync(CreateProductRequest req, CancellationToken ct)
    {
        var result = await _executor.ExecuteAsync(
            CreateProductWorkflow.Definition,
            new CreateProductInput { Title = req.Title, Price = req.Price },
            Resolve<IServiceProvider>(),
            ct
        );

        if (result.IsSuccess)
            await SendOkAsync(result.Output!, ct);
        else
            await SendErrorsAsync(400, ct);
    }
}
```

### Async Workflow with Job Queues

For long-running workflows, integrate with FastEndpoints job queues:

```csharp
public class ExecuteWorkflowJob<TInput, TOutput> : ICommand<WorkflowResult<TOutput>>
{
    public Guid TrackingId { get; set; }
    public string WorkflowName { get; set; } = null!;
    public TInput Input { get; set; } = default!;
}

public class ExecuteWorkflowJobHandler<TInput, TOutput>
    : ICommandHandler<ExecuteWorkflowJob<TInput, TOutput>, WorkflowResult<TOutput>>
{
    private readonly WorkflowExecutor _executor;
    private readonly IServiceProvider _serviceProvider;
    private readonly WorkflowRegistry _registry;

    public async Task<WorkflowResult<TOutput>> ExecuteAsync(
        ExecuteWorkflowJob<TInput, TOutput> job,
        CancellationToken ct)
    {
        var workflow = _registry.Get<TInput, TOutput>(job.WorkflowName);

        return await _executor.ExecuteAsync(
            workflow,
            job.Input,
            _serviceProvider,
            ct
        );
    }
}

// Usage
var trackingId = await new ExecuteWorkflowJob<CreateProductInput, Product>
{
    WorkflowName = "create-product-workflow",
    Input = input
}.QueueJobAsync(ct);

// Check progress
var result = await JobTracker<ExecuteWorkflowJob<CreateProductInput, Product>>
    .GetJobResultAsync<WorkflowResult<Product>>(trackingId, ct);
```

## Comparison

| Aspect | MedusaJS | Our .NET Library |
|--------|----------|------------------|
| **DSL** | `createWorkflow()` | `WorkflowBuilder<,>` |
| **Steps** | `createStep()` | `.Step()` fluent API |
| **Compensation** | Automatic reverse | Automatic reverse |
| **State** | Redis/In-memory | EF Core table |
| **DI** | Awilix container | ASP.NET Core DI |
| **Async** | Workflow Engine module | FastEndpoints Job Queues |
| **Type Safety** | TypeScript inference | C# generics |

## What We Built

**Reusable Library Components:**
1. ✅ `WorkflowBuilder<TInput, TOutput>` - DSL for defining workflows
2. ✅ `WorkflowDefinition<TInput, TOutput>` - Compiled workflow
3. ✅ `WorkflowExecutor` - Executes with automatic compensation
4. ✅ `StepContext` - DI and context for steps
5. ✅ `WorkflowStateStore` - Persist workflow state
6. ✅ Integration with FastEndpoints Job Queues for async execution

**Developers Use Like This:**
```csharp
var workflow = new WorkflowBuilder<Input, Output>()
    .Step("step1", async (input, ctx) => /* ... */)
    .Step("step2", async (output1, ctx) => /* ... */)
    .Build("my-workflow");

var result = await executor.ExecuteAsync(workflow, input, serviceProvider);
```

**No 3rd Party Libraries Needed:**
- ✅ FastEndpoints (job queues, events, commands)
- ✅ EF Core (state persistence)
- ✅ ASP.NET Core DI (service resolution)

## Project Structure

```
Medusa.Workflows/
├── StepResult.cs
├── StepContext.cs
├── WorkflowStep.cs
├── WorkflowBuilder.cs
├── WorkflowDefinition.cs
├── WorkflowExecutor.cs
├── WorkflowStateStore.cs
├── WorkflowResult.cs
└── Extensions/
    └── ServiceCollectionExtensions.cs
```

## Registration

```csharp
// Startup
builder.Services.AddWorkflows(options =>
{
    options.UseEfCoreStateStore<MedusaDbContext>();
});

// Register workflows
builder.Services.AddSingleton(CreateProductWorkflow.Definition);
```

## References

**MedusaJS Source:**
- `workflows-sdk/src/utils/composer/create-workflow.ts` - Workflow DSL
- `workflows-sdk/src/utils/composer/create-step.ts` - Step definition
- `orchestration/src/transaction/transaction-orchestrator.ts` - Execution engine
- `workflow-engine-*/src/services/workflow-engine.ts` - Async execution

**FastEndpoints:**
- https://fast-endpoints.com/docs/command-bus
- https://fast-endpoints.com/docs/event-bus
- https://fast-endpoints.com/docs/job-queues

**Our Implementation:**
- `/SAGA_IMPLEMENTATION_GUIDE.md` - Manual saga pattern (alternative approach)
