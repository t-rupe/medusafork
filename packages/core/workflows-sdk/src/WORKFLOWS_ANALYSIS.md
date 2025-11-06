# Workflow Engine Analysis - MedusaJS

## Purpose
Distributed transaction orchestration system enabling saga pattern for long-running business processes with automatic compensation (rollback) on failures.

## Core Concepts

### 1. Workflow
Composable, retryable, distributed transaction with compensation logic:
- **Step-based execution** - Sequential/parallel task execution
- **Automatic compensation** - Rollback on failures (Saga pattern)
- **Idempotency** - Safe retries via transaction IDs
- **Async execution** - Background processing with workflow engine
- **Type-safe composition** - Full TypeScript inference

### 2. Step
Atomic unit of work with invoke + compensate functions:
- **Invoke**: Forward execution (e.g., create product)
- **Compensate**: Rollback (e.g., delete product)
- **StepResponse**: Output + compensation input
- **Async/Sync modes**: Immediate or background execution

### 3. Orchestration Engine
Manages workflow execution state:
- **Transaction tracking** - Stores step results and state
- **Compensation orchestration** - Auto-executes rollbacks
- **Event emission** - Progress tracking
- **Storage backends** - In-memory or Redis

## Architecture

### **1. Workflow Definition** (`create-workflow.ts`)

```typescript
const createProductWorkflow = createWorkflow(
  "create-product",
  (input: WorkflowData<ProductInput>) => {
    const product = createProductStep(input)
    const inventory = reserveInventoryStep(product.id)
    const notification = sendEmailStep({
      product: product,
      inventory: inventory
    })

    return new WorkflowResponse({
      product,
      inventory
    })
  }
)
```

**Execution:**
```typescript
const { result, transaction } = await createProductWorkflow(container).run({
  input: { title: "Shirt", price: 1000 }
})
```

**Key Mechanisms:**
- **Proxify pattern**: Input/outputs are proxies tracking dependencies
- **Composer context**: Global state during composition
- **WorkflowManager**: Registry of all workflows
- **Handler Map**: Step name → invoke/compensate functions

See: `workflows-sdk/src/utils/composer/create-workflow.ts`

### **2. Step Definition** (`create-step.ts`)

```typescript
const createProductStep = createStep(
  "create-product",
  async (input: ProductInput, { container }) => {
    const productService = container.resolve("productService")
    const product = await productService.create(input)

    return new StepResponse(
      product,                    // Output (passed to next steps)
      { productId: product.id }   // Compensation input
    )
  },
  async (compensationInput, { container }) => {
    if (!compensationInput) return

    const productService = container.resolve("productService")
    await productService.delete(compensationInput.productId)
  }
)
```

**Features:**
- **Conditional execution**: `step.if(condition, fn)`
- **Config override**: `step.config({ async: true })`
- **Dependency injection**: Via `container` in context
- **Automatic compensation**: Executed in reverse order on error

See: `workflows-sdk/src/utils/composer/create-step.ts`

### **3. StepResponse** (`helpers/step-response.ts`)

Wrapper for step output + compensation data:

```typescript
class StepResponse<TOutput, TCompensateInput> {
  constructor(
    public output: TOutput,
    public compensateInput?: TCompensateInput
  ) {}

  static skip() {
    // Skip step execution (conditional)
  }
}
```

If `compensateInput` not provided, `output` is used for compensation.

### **4. Workflow Execution Flow**

**Registration Phase (Build Time):**
```
createWorkflow("name", composer)
→ Creates composer context
→ Executes composer with proxied input
→ Builds transaction flow definition
→ Registers handlers in WorkflowManager
→ Returns executable workflow function
```

**Execution Phase (Runtime):**
```
workflow(container).run({ input })
→ WorkflowManager.run(name, input, context)
→ DistributedTransaction.begin()
→ For each step in order:
  → Resolve input dependencies from previous steps
  → Execute step.invoke(resolvedInput, context)
  → Store result in transaction state
  → Emit step.completed event
→ If error occurs:
  → Execute compensation steps in reverse
  → Emit workflow.failed event
→ Else:
  → Emit workflow.completed event
→ Return { result, transaction }
```

### **5. Transaction Orchestrator** (`orchestration/transaction-orchestrator.ts`)

Manages step execution and compensation:

```typescript
class TransactionOrchestrator {
  async run(transactionId, flow, context) {
    const transaction = new DistributedTransaction(
      transactionId,
      flow,
      this.storage
    )

    for (const step of flow.steps) {
      try {
        const input = this.resolveStepInput(step, transaction)
        const result = await this.handlers[step.name].invoke({
          input,
          container: context.container,
          transaction
        })

        transaction.addStepSuccess(step, result)
      } catch (error) {
        await this.compensate(transaction, error)
        throw error
      }
    }

    return transaction.getResult()
  }

  async compensate(transaction, error) {
    const completedSteps = transaction.getCompletedSteps().reverse()

    for (const step of completedSteps) {
      if (step.noCompensation) continue

      const compensateInput = transaction.getCompensateInput(step)
      await this.handlers[step.name].compensate({
        input: compensateInput,
        container: context.container,
        transaction
      })
    }
  }
}
```

See: `orchestration/src/transaction/transaction-orchestrator.ts`

### **6. Distributed Transaction** (`orchestration/distributed-transaction.ts`)

Tracks execution state:

```typescript
class DistributedTransaction {
  private state: {
    steps: Map<string, StepResult>
    status: "idle" | "executing" | "completed" | "failed"
    errors: Error[]
  }

  addStepSuccess(step, result) {
    this.state.steps.set(step.name, {
      status: "success",
      output: result.output,
      compensateInput: result.compensateInput
    })
  }

  async persist() {
    await this.storage.save(this.transactionId, this.state)
  }

  async resume() {
    const state = await this.storage.load(this.transactionId)
    this.state = state
    // Continue from last completed step
  }
}
```

### **7. Workflow Engine Service** (Modules)

For async/background workflows:

```typescript
// Workflow engine module
interface IWorkflowEngineService {
  async run(workflowId, { input, transactionId, context }) {
    // Queue workflow for background execution
    await this.queue.add({
      workflowId,
      input,
      transactionId,
      context
    })

    return { transactionId, status: "pending" }
  }

  async getStatus(transactionId) {
    return this.storage.getTransaction(transactionId)
  }

  async cancel(workflowId, { transactionId }) {
    // Execute compensation for transaction
    await this.orchestrator.compensate(transactionId)
  }
}
```

Implementations:
- **workflow-engine-inmemory**: In-process execution
- **workflow-engine-redis**: Redis-backed queue

See: `modules/workflow-engine-*/src/services/workflow-engine.ts`

### **8. Parallel Execution** (`parallelize.ts`)

Execute steps concurrently:

```typescript
const { productA, productB } = parallelize(
  createProductStep({ title: "A" }),
  createProductStep({ title: "B" })
)

// Both execute in parallel, results are tupled
```

### **9. Transform Utility** (`transform.ts`)

Data manipulation between steps:

```typescript
const transformedData = transform({ product, price }, (data) => ({
  ...data.product,
  finalPrice: data.price * 1.1
}))

// Required because composer function can't manipulate data directly
```

### **10. Hooks** (`create-hook.ts`)

Extension points for workflows:

```typescript
const workflow = createWorkflow("my-workflow", (input) => {
  const hook = createHook("afterProduct", () => {
    // Extension point
  })

  const product = createProductStep(input)
  hook.invoke(product)  // Trigger hook

  return new WorkflowResponse(product)
})

// External code can register handlers
workflow.hooks.afterProduct((productData) => {
  // Custom logic
})
```

## Patterns

### **1. Saga Pattern Implementation**

```
Step 1: Create Product ✓
Step 2: Reserve Inventory ✓
Step 3: Charge Payment ✗ (FAILS)

Compensation Flow (Reverse):
Step 2 Compensate: Release Inventory ✓
Step 1 Compensate: Delete Product ✓
```

### **2. Idempotency**

```typescript
const transactionId = `create-product-${orderId}`

// First call
await workflow.run({
  input,
  transactionId  // Custom ID for idempotency
})

// Retry (same transactionId) - resumes from last successful step
await workflow.run({
  input,
  transactionId
})
```

### **3. Nested Workflows**

```typescript
const childWorkflow = createWorkflow("child", (input) => {
  return new WorkflowResponse(doSomething(input))
})

const parentWorkflow = createWorkflow("parent", (input) => {
  const result = childWorkflow.runAsStep({ input })
  return new WorkflowResponse(result)
})

// Nested compensation automatically handled
```

### **4. Conditional Steps**

```typescript
const step = createProductStep(input).if(
  input,
  (data) => data.shouldCreate
)

// Step only executes if condition returns true
```

### **5. Async Steps**

```typescript
const asyncStep = createStep(
  { name: "send-email", async: true },
  async (input) => {
    // Queued for background execution
    return new StepResponse({ emailId: "..." })
  }
)

// Workflow continues, step executes asynchronously
```

## .NET Mapping with MassTransit Sagas

### 1. Replace Workflow with MassTransit State Machine

**Medusa Workflow:**
```typescript
const createProductWorkflow = createWorkflow("create-product", (input) => {
  const product = createProductStep(input)
  const inventory = reserveInventoryStep(product.id)
  return new WorkflowResponse({ product, inventory })
})
```

**.NET MassTransit Saga:**
```csharp
public class CreateProductSaga : MassTransitStateMachine<CreateProductState>
{
    public CreateProductSaga()
    {
        InstanceState(x => x.CurrentState);

        Event(() => CreateProductRequested, x => x.CorrelateById(m => m.Message.OrderId));

        Initially(
            When(CreateProductRequested)
                .Then(context => {
                    context.Instance.ProductData = context.Data;
                })
                .PublishAsync(context => context.Init<CreateProduct>(new
                {
                    context.Instance.ProductData
                }))
                .TransitionTo(CreatingProduct)
        );

        During(CreatingProduct,
            When(ProductCreated)
                .PublishAsync(context => context.Init<ReserveInventory>(new
                {
                    ProductId = context.Data.ProductId
                }))
                .TransitionTo(ReservingInventory),

            When(ProductCreationFailed)
                .TransitionTo(Failed)
        );

        During(ReservingInventory,
            When(InventoryReserved)
                .TransitionTo(Completed),

            When(InventoryReservationFailed)
                .PublishAsync(context => context.Init<DeleteProduct>(new
                {
                    context.Instance.ProductId
                }))
                .TransitionTo(CompensatingProduct)
        );
    }

    public State CreatingProduct { get; private set; }
    public State ReservingInventory { get; private set; }
    public State CompensatingProduct { get; private set; }
    public State Completed { get; private set; }
    public State Failed { get; private set; }

    public Event<CreateProductRequest> CreateProductRequested { get; private set; }
    public Event<ProductCreatedEvent> ProductCreated { get; private set; }
    public Event<InventoryReservedEvent> InventoryReserved { get; private set; }
}

public class CreateProductState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; }
    public ProductData ProductData { get; set; }
    public string ProductId { get; set; }
}
```

### 2. Step Pattern with Consumers

**Medusa Step:**
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

**.NET Consumer with Compensation:**
```csharp
public class CreateProductConsumer : IConsumer<CreateProduct>
{
    private readonly IProductService _productService;

    public async Task Consume(ConsumeContext<CreateProduct> context)
    {
        try
        {
            var product = await _productService.Create(context.Message);

            await context.Publish(new ProductCreated
            {
                ProductId = product.Id,
                Product = product
            });
        }
        catch (Exception ex)
        {
            await context.Publish(new ProductCreationFailed
            {
                Reason = ex.Message
            });
        }
    }
}

// Compensation consumer
public class DeleteProductConsumer : IConsumer<DeleteProduct>
{
    private readonly IProductService _productService;

    public async Task Consume(ConsumeContext<DeleteProduct> context)
    {
        await _productService.Delete(context.Message.ProductId);
    }
}
```

### 3. Workflow Execution

**Medusa:**
```typescript
const { result } = await workflow(container).run({ input })
```

**.NET:**
```csharp
// Via message bus
await _publishEndpoint.Publish(new CreateProductRequest
{
    OrderId = orderId,
    Title = "Shirt"
});

// Check status
var saga = await _repository.GetSaga(orderId);
```

### 4. Alternative: Simpler Workflow Library (Elsa/Workflow Core)

If saga pattern too complex, use .NET workflow library:

```csharp
public class CreateProductWorkflow : IWorkflow<ProductInput, ProductOutput>
{
    public string Id => "create-product-workflow";

    public void Build(IWorkflowBuilder<ProductInput, ProductOutput> builder)
    {
        builder
            .StartWith<CreateProductStep>()
                .Input(step => step.Input, data => data)
                .Output(data => data.ProductId, step => step.Output.ProductId)
                .CompensateWith<DeleteProductStep>()
            .Then<ReserveInventoryStep>()
                .Input(step => step.ProductId, data => data.ProductId)
                .CompensateWith<ReleaseInventoryStep>()
            .Then<SendEmailStep>();
    }
}

// Steps
public class CreateProductStep : StepBody
{
    public ProductInput Input { get; set; }
    public ProductOutput Output { get; set; }

    public override async Task<ExecutionResult> RunAsync(IStepExecutionContext context)
    {
        var productService = context.GetService<IProductService>();
        var product = await productService.Create(Input);

        Output = new ProductOutput { ProductId = product.Id };

        return ExecutionResult.Next();
    }
}

public class DeleteProductStep : StepBody
{
    public string ProductId { get; set; }

    public override async Task<ExecutionResult> RunAsync(IStepExecutionContext context)
    {
        var productService = context.GetService<IProductService>();
        await productService.Delete(ProductId);

        return ExecutionResult.Next();
    }
}

// Execute
var workflowHost = serviceProvider.GetRequiredService<IWorkflowHost>();
await workflowHost.StartWorkflow("create-product-workflow", new ProductInput
{
    Title = "Shirt"
});
```

### 5. Idempotency

**Medusa:** Built-in via transactionId
**.NET:** Use MassTransit RequestId or Inbox pattern

```csharp
public class CreateProductConsumer : IConsumer<CreateProduct>
{
    private readonly IProductService _productService;
    private readonly IIdempotencyStore _idempotencyStore;

    public async Task Consume(ConsumeContext<CreateProduct> context)
    {
        var requestId = context.RequestId ?? context.MessageId;

        if (await _idempotencyStore.Exists(requestId))
        {
            // Already processed
            return;
        }

        var product = await _productService.Create(context.Message);
        await _idempotencyStore.Store(requestId, product);

        await context.Publish(new ProductCreated { Product = product });
    }
}
```

## Key Differences

| Aspect | MedusaJS | .NET (MassTransit) | .NET (Workflow Core) |
|--------|----------|-------------------|---------------------|
| Pattern | Saga (coded) | Saga (state machine) | Workflow (coded) |
| Compensation | Auto-reverse | Manual state transitions | Compensate blocks |
| State Storage | Transaction store | Saga repository | Workflow persistence |
| Execution | In-process or async | Message-based | In-process |
| Type Safety | Full TypeScript | Message contracts | Step DTOs |
| Composition | Functional | Declarative | Fluent builder |

## Implementation Recommendations

### NuGet Packages

**Option 1: MassTransit (Enterprise-grade sagas)**
- **MassTransit**
- **MassTransit.RabbitMQ** or **MassTransit.AzureServiceBus**
- **MassTransit.EntityFrameworkCore** - State persistence

**Option 2: WorkflowCore (Simpler workflows)**
- **WorkflowCore**
- **WorkflowCore.Persistence.EntityFramework**

**Option 3: Elsa Workflows (Complex BPM)**
- **Elsa**
- **Elsa.Persistence.EntityFrameworkCore**

### Recommendation

For Medusa-like workflows with compensation:
→ **Use MassTransit Sagas** for distributed, event-driven sagas
→ **Use WorkflowCore** for in-process workflows with simpler compensation

## References

**Core Files:**
- `workflows-sdk/src/utils/composer/create-workflow.ts` - Workflow definition
- `workflows-sdk/src/utils/composer/create-step.ts` - Step definition
- `orchestration/src/transaction/transaction-orchestrator.ts` - Orchestration
- `orchestration/src/transaction/distributed-transaction.ts` - State tracking
- `orchestration/src/workflow/workflow-manager.ts` - Workflow registry
- `modules/workflow-engine-*/src/services/workflow-engine.ts` - Async execution

**Example Workflows:**
- `core-flows/src/product/workflows/create-product.ts`
- `core-flows/src/order/workflows/create-order.ts`
