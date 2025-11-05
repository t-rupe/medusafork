# Medusa Core Engine Architecture - .NET Porting Guide

## Executive Summary

This document provides a comprehensive breakdown of Medusa's core engine architecture, separate from its commerce modules, to guide a .NET port. The analysis reveals that Medusa is fundamentally a **modular workflow orchestration framework** with built-in dependency injection, cross-module querying, and automatic compensation for distributed transactions.

**Key Insight**: Before building commerce modules (Product, Order, etc.), you must port the **core engine** that powers them.

---

## Table of Contents

1. [Core Engine Components](#1-core-engine-components)
2. [Dependency Injection System](#2-dependency-injection-system)
3. [Module System Architecture](#3-module-system-architecture)
4. [Workflow Engine](#4-workflow-engine)
5. [Core Abstractions](#5-core-abstractions)
6. [Module Communication Patterns](#6-module-communication-patterns)
7. [Porting Order for .NET](#7-porting-order-for-net)
8. [.NET Implementation Interfaces](#8-net-implementation-interfaces)
9. [File Structure Reference](#9-file-structure-reference)
10. [Critical Design Patterns](#10-critical-design-patterns)

---

## 1. Core Engine Components

Medusa's "engine" consists of these foundational layers:

```
┌─────────────────────────────────────────────────────────┐
│                  Application Layer                      │
│  (API Routes, Admin Dashboard, Storefront)             │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│               Workflow Orchestrator                     │
│  (Business logic, compensation, state management)       │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│            Module System + Remote Query                 │
│  (Cross-module data access, relationships, events)      │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│              DI Container (Awilix)                      │
│  (Service registration, resolution, scoping)            │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│           Infrastructure (DB, Cache, Events)            │
└─────────────────────────────────────────────────────────┘
```

**The core engine provides:**
- Dependency injection and service resolution
- Module discovery, loading, and lifecycle management
- Cross-module querying with automatic JOINs
- Workflow orchestration with automatic rollback on failure
- Event-driven communication between modules
- Transaction management across modules

---

## 2. Dependency Injection System

### Location
- `/packages/core/utils/src/common/medusa-container.ts`
- `/packages/core/framework/src/container.ts`

### Implementation

Medusa uses **Awilix** as its DI container with a custom wrapper:

```typescript
// Core container creation
const container = createMedusaContainer()

// Standard registration
container.register("productService", asClass(ProductService))
container.register("logger", asValue(winstonLogger))

// Plugin-style registration (unique to Medusa)
container.registerAdd("subscriber", asClass(OrderSubscriber))
container.registerAdd("subscriber", asClass(ProductSubscriber))
// Later: container.resolve("subscriber") returns array of all subscribers

// Scoped resolution (per-request)
const requestScope = container.createScope()
requestScope.register("user", asValue(currentUser))
```

### Key Methods

| Method | Purpose |
|--------|---------|
| `register(name, resolver)` | Register a single service |
| `registerAdd(name, resolver)` | Add to a list of services (multi-instance pattern) |
| `resolve(name)` | Retrieve registered service |
| `createScope()` | Create request-scoped container |
| `build(resolver)` | Build complex dependencies with full graph resolution |

### .NET Equivalent

**Option 1: Microsoft.Extensions.DependencyInjection**
```csharp
services.AddScoped<IProductService, ProductService>();
services.AddSingleton<ILogger>(logger);

// For registerAdd pattern:
services.AddScoped<ISubscriber, OrderSubscriber>();
services.AddScoped<ISubscriber, ProductSubscriber>();
// Resolve: IEnumerable<ISubscriber> subscribers
```

**Option 2: Autofac** (closer to Awilix)
```csharp
var builder = new ContainerBuilder();
builder.RegisterType<ProductService>().As<IProductService>();
builder.Register(c => logger).As<ILogger>();

// Multi-instance pattern
builder.RegisterType<OrderSubscriber>().As<ISubscriber>();
builder.RegisterType<ProductSubscriber>().As<ISubscriber>();
```

### Porting Priority
**CRITICAL - Port First** (Phase 1.1)

---

## 3. Module System Architecture

### Location
- `/packages/core/modules-sdk/src/medusa-module.ts` - Static registry
- `/packages/core/modules-sdk/src/loaders/module-loader.ts` - Dynamic loader
- `/packages/core/modules-sdk/src/definitions.ts` - Module metadata

### Architecture Overview

```typescript
// Module Definition (metadata)
const ProductModule: ModuleDefinition = {
  key: "product",
  label: "Product",
  defaultPackage: "@medusajs/medusa/product",
  isRequired: true,
  isQueryable: true,
  dependencies: ["eventBus", "logger"]
}

// Module Resolution (concrete implementation)
const resolution: ModuleResolution = {
  resolutionPath: "/path/to/product-module",
  definition: ProductModule,
  dependencies: ["eventBus", "logger"],
  moduleDeclaration: {
    scope: "internal",
    options: { database_url: "..." }
  }
}

// Module Loader
async function loadModules(container, moduleResolutions) {
  for (const resolution of moduleResolutions) {
    // 1. Resolve dependencies from container
    const deps = resolution.dependencies.map(d => container.resolve(d))

    // 2. Load module class
    const ModuleClass = await import(resolution.resolutionPath)

    // 3. Instantiate with dependencies
    const instance = new ModuleClass(deps, resolution.moduleDeclaration)

    // 4. Register in container
    container.register(resolution.definition.key, asValue(instance))

    // 5. Store joiner config for cross-module queries
    MedusaModule.setJoinerConfig(
      resolution.definition.key,
      await instance.__joinerConfig()
    )
  }
}
```

### Module Lifecycle

```
1. Definition Phase
   ↓
   Register module metadata (key, dependencies, isQueryable)

2. Resolution Phase
   ↓
   Merge user config with defaults
   ↓
   Resolve path to module implementation

3. Loading Phase
   ↓
   Resolve dependencies from container
   ↓
   Import module class
   ↓
   Call constructor with dependencies
   ↓
   Run database migrations (if needed)

4. Registration Phase
   ↓
   Register module service in container
   ↓
   Extract __joinerConfig() for relationships
   ↓
   Store in MedusaModule static registry

5. Ready
   ↓
   Module available for workflows and API routes
```

### Module Interface

Every module must implement:

```typescript
interface IModuleService {
  // Metadata
  __definition: ModuleDefinition
  __joinerConfig(): ModuleJoinerConfig

  // Lifecycle hooks
  __hooks?: {
    onApplicationStart?: () => Promise<void>
    onApplicationShutdown?: () => Promise<void>
    onApplicationPrepareShutdown?: () => Promise<void>
  }

  // Soft delete (for cascade operations)
  softDelete(
    ids: { [entityName: string]: string[] },
    context?: Context
  ): Promise<{
    data: any,
    ids: { [entityName: string]: string[] }
  }>

  // Restore soft-deleted
  restore(
    ids: { [entityName: string]: string[] },
    context?: Context
  ): Promise<{
    data: any,
    ids: { [entityName: string]: string[] }
  }>
}
```

### .NET Implementation

```csharp
// Module definition
public class ModuleDefinition
{
    public string Key { get; set; }
    public string Label { get; set; }
    public string DefaultPackage { get; set; }
    public bool IsRequired { get; set; }
    public bool IsQueryable { get; set; }
    public string[] Dependencies { get; set; }
}

// Module service interface
public interface IModuleService
{
    ModuleDefinition Definition { get; }
    ModuleJoinerConfig JoinerConfig { get; }

    Task<(object Data, Dictionary<string, string[]> Ids)>
        SoftDeleteAsync(Dictionary<string, string[]> ids, Context? context = null);

    Task<(object Data, Dictionary<string, string[]> Ids)>
        RestoreAsync(Dictionary<string, string[]> ids, Context? context = null);
}

// Module loader
public class ModuleLoader
{
    public async Task LoadModulesAsync(
        IServiceCollection services,
        IEnumerable<ModuleResolution> resolutions)
    {
        foreach (var resolution in resolutions)
        {
            // Resolve dependencies
            // Load assembly
            // Instantiate module
            // Register in container
        }
    }
}
```

### Porting Priority
**HIGH - Port Second** (Phase 2.1-2.3)

---

## 4. Workflow Engine

### Location
- `/packages/core/workflows-sdk/src/utils/composer/create-workflow.ts` - DSL
- `/packages/core/orchestration/src/transaction/transaction-orchestrator.ts` - Execution engine
- `/packages/core/orchestration/src/transaction/distributed-transaction.ts` - State machine

### Architecture

The workflow engine is the **most critical** part of Medusa. It provides:
- Multi-step operation orchestration
- **Automatic compensation/rollback on failure**
- Parallel step execution
- Retry logic and timeouts
- Cross-module transaction coordination

### Workflow Definition (User Code)

```typescript
import { createWorkflow, createStep } from "@medusajs/workflows-sdk"

// Define individual steps
const createOrderStep = createStep(
  "create-order",
  async ({ input }, { container }) => {
    const orderService = container.resolve("orderService")
    const order = await orderService.create(input)

    return { order }
  },
  async ({ order }, { container }) => {
    // COMPENSATION: Called on failure of later steps
    const orderService = container.resolve("orderService")
    await orderService.delete(order.id)
  }
)

const reserveInventoryStep = createStep(
  "reserve-inventory",
  async ({ orderId }, { container }) => {
    const inventoryService = container.resolve("inventoryService")
    const reservation = await inventoryService.reserve(orderId)
    return { reservation }
  },
  async ({ reservation }, { container }) => {
    // COMPENSATION: Release reservation
    const inventoryService = container.resolve("inventoryService")
    await inventoryService.release(reservation.id)
  }
)

const chargeCustomerStep = createStep(
  "charge-customer",
  async ({ orderId, amount }, { container }) => {
    const paymentService = container.resolve("paymentService")
    const charge = await paymentService.charge(orderId, amount)
    return { charge }
  },
  async ({ charge }, { container }) => {
    // COMPENSATION: Refund payment
    const paymentService = container.resolve("paymentService")
    await paymentService.refund(charge.id)
  }
)

// Compose workflow
export const createOrderWorkflow = createWorkflow(
  "create-order-workflow",
  (input: OrderInput) => {
    const { order } = createOrderStep(input)
    const { reservation } = reserveInventoryStep({ orderId: order.id })
    const { charge } = chargeCustomerStep({
      orderId: order.id,
      amount: order.total
    })

    return new WorkflowResponse({ order, reservation, charge })
  }
)
```

### Workflow Execution Flow

```
User Request
    ↓
API Route calls: createOrderWorkflow.run(input)
    ↓
TransactionOrchestrator.execute()
    ↓
┌──────────────────────────────────────────────┐
│         INVOKE PHASE (Forward)               │
├──────────────────────────────────────────────┤
│ 1. createOrderStep.invoke()        ✓         │
│ 2. reserveInventoryStep.invoke()   ✓         │
│ 3. chargeCustomerStep.invoke()     ✗ FAILED  │
└──────────────────────────────────────────────┘
    ↓ (Error detected)
┌──────────────────────────────────────────────┐
│      COMPENSATION PHASE (Reverse)            │
├──────────────────────────────────────────────┤
│ 3. chargeCustomerStep.compensate() (skipped) │
│ 2. reserveInventoryStep.compensate() ✓       │
│    → Releases inventory reservation          │
│ 1. createOrderStep.compensate() ✓            │
│    → Deletes the order                       │
└──────────────────────────────────────────────┘
    ↓
Throw error to API route
```

### Transaction State Machine

```typescript
enum TransactionState {
  DORMANT,      // Not started
  PENDING,      // Waiting to start
  PROCESSING,   // Executing steps
  DONE,         // Completed successfully
  FAILED,       // One or more steps failed
  REVERTED,     // Compensation completed
  TIMEOUT,      // Step exceeded timeout
  SKIPPED       // Step was skipped
}

enum TransactionStepStatus {
  IDLE,         // Not executed
  OK,           // Completed successfully
  INVOKING,     // Currently executing
  WAITING,      // Waiting for dependencies
  COMPENSATING, // Running compensation
  REVERTED,     // Compensation completed
  FAILED,       // Execution failed
  SKIPPED,      // Skipped by workflow logic
  TIMEOUT,      // Exceeded timeout
  DORMANT       // Not yet scheduled
}
```

### Core Classes

#### TransactionOrchestrator

```typescript
class TransactionOrchestrator extends EventEmitter {
  id: string                              // Workflow ID
  definition: TransactionStepsDefinition  // Step graph
  options?: TransactionModelOptions       // Retry, timeout

  async execute(
    container: IServiceContainer,
    payload: TransactionPayload,
    context?: Context
  ): Promise<TransactionResult> {
    // 1. Create DistributedTransaction
    const transaction = new DistributedTransaction(this.definition, payload)

    // 2. Execute steps in dependency order
    for (const step of this.getExecutionOrder()) {
      try {
        const handler = container.resolve(`step.${step.id}`)
        const result = await handler.invoke({
          step,
          payload,
          previousResults: transaction.getPreviousResults(),
          container,
          context
        })

        transaction.markStepCompleted(step.id, result)
      } catch (error) {
        // 3. Trigger compensation on failure
        await this.compensate(transaction, container, context)
        throw error
      }
    }

    return transaction.getResult()
  }

  async compensate(
    transaction: DistributedTransaction,
    container: IServiceContainer,
    context?: Context
  ) {
    // Execute steps in REVERSE order
    const executedSteps = transaction.getExecutedSteps().reverse()

    for (const step of executedSteps) {
      const handler = container.resolve(`step.${step.id}`)

      if (handler.compensate) {
        await handler.compensate({
          step,
          payload: transaction.payload,
          container,
          context
        })

        transaction.markStepReverted(step.id)
      }
    }

    transaction.state = TransactionState.REVERTED
  }
}
```

#### TransactionStep

```typescript
class TransactionStep {
  id: string                          // "create-order"
  action: string                      // Step handler ID
  depth: number                       // For ordering
  uuid: string                        // Unique per execution
  invoked: boolean                    // Has invoke run?
  result?: unknown                    // Step output
  error?: Error                       // Step failure
  status: TransactionStepStatus       // Current state

  // Dependencies
  next: TransactionStep[]             // Steps that depend on this
  previous: TransactionStep[]         // Steps this depends on
}
```

#### Step Handler

```typescript
interface WorkflowStepHandler<TInput, TOutput> {
  invoke: async (args: {
    step: TransactionStep,
    payload: TransactionPayload,
    previousResults: Record<string, unknown>,
    container: IServiceContainer,
    context?: Context
  }): Promise<TOutput>

  compensate?: async (args: {
    step: TransactionStep,
    payload: TransactionPayload,
    container: IServiceContainer,
    context?: Context
  }): Promise<void>
}
```

### Advanced Features

#### Parallel Execution

```typescript
import { parallelize } from "@medusajs/workflows-sdk"

const workflow = createWorkflow("parallel-example", () => {
  // These steps run in parallel
  const [result1, result2, result3] = parallelize(
    updateInventoryStep(),
    sendEmailStep(),
    logEventStep()
  )

  return { result1, result2, result3 }
})
```

#### Conditional Steps

```typescript
import { when } from "@medusajs/workflows-sdk"

const workflow = createWorkflow("conditional-example", (input) => {
  const order = createOrderStep(input)

  when({ order }, ({ order }) => {
    return order.total > 100
  }, () => {
    // Only runs if condition is true
    applyDiscountStep(order)
  })

  return { order }
})
```

#### Transform Data

```typescript
import { transform } from "@medusajs/workflows-sdk"

const workflow = createWorkflow("transform-example", () => {
  const order = createOrderStep()

  const orderData = transform({ order }, ({ order }) => ({
    id: order.id,
    total: order.total,
    formatted: `Order #${order.id}`
  }))

  sendEmailStep(orderData)
})
```

### .NET Implementation

```csharp
// Step definition
public delegate Task<TOutput> StepInvokeHandler<TInput, TOutput>(
    TransactionStep step,
    TInput payload,
    Dictionary<string, object> previousResults,
    IServiceContainer container,
    Context? context);

public delegate Task StepCompensateHandler<TInput>(
    TransactionStep step,
    TInput payload,
    IServiceContainer container,
    Context? context);

public class WorkflowStepHandler<TInput, TOutput>
{
    public StepInvokeHandler<TInput, TOutput> Invoke { get; set; }
    public StepCompensateHandler<TInput>? Compensate { get; set; }
}

// Workflow DSL
public static class WorkflowBuilder
{
    public static Workflow<TInput, TOutput> CreateWorkflow<TInput, TOutput>(
        string name,
        Func<TInput, WorkflowResponse<TOutput>> definition)
    {
        // Build workflow from definition
    }

    public static WorkflowStep<TInput, TOutput> CreateStep<TInput, TOutput>(
        string name,
        StepInvokeHandler<TInput, TOutput> invoke,
        StepCompensateHandler<TInput>? compensate = null)
    {
        // Register step handler
    }
}

// Transaction orchestrator
public class TransactionOrchestrator
{
    public async Task<TransactionResult<TOutput>> ExecuteAsync<TInput, TOutput>(
        TransactionStepsDefinition definition,
        TInput payload,
        IServiceContainer container,
        Context? context = null)
    {
        var transaction = new DistributedTransaction(definition, payload);

        try
        {
            // Execute steps in dependency order
            foreach (var step in GetExecutionOrder(definition))
            {
                var handler = container.Resolve<WorkflowStepHandler>(step.Action);
                var result = await handler.Invoke(step, payload, transaction.GetPreviousResults(), container, context);
                transaction.MarkStepCompleted(step.Id, result);
            }

            return transaction.GetResult<TOutput>();
        }
        catch (Exception ex)
        {
            // Compensate in reverse order
            await CompensateAsync(transaction, container, context);
            throw;
        }
    }
}
```

### Porting Priority
**HIGH - Port Third** (Phase 4.1-4.5)

---

## 5. Core Abstractions

### Location
- `/packages/core/utils/src/modules-sdk/medusa-service.ts` - Base service factory
- `/packages/core/types/src/dal/repository-service.ts` - Repository interface
- `/packages/core/types/src/shared-context.ts` - Context pattern

### A. Base Service Factory (MedusaService)

Medusa **auto-generates** CRUD methods for each entity model:

```typescript
// User defines models
const models = {
  Product: ProductModel,
  ProductVariant: ProductVariantModel
}

// MedusaService generates methods automatically
const service = MedusaService(models)

// Generated methods:
// - retrieveProduct(id, config?, context?)
// - listProducts(filters?, config?, context?)
// - listAndCountProducts(filters?, config?, context?)
// - createProducts(data, context?)
// - updateProducts(data, context?)
// - deleteProducts(ids, context?)
// - softDeleteProducts(ids, context?)
// - restoreProducts(ids, context?)
// - upsertProducts(data, context?)
//
// - retrieveProductVariant(id, config?, context?)
// - listProductVariants(...)
// ... and so on for each model
```

**Implementation:**

```typescript
export function MedusaService<TModels>(
  models: Record<string, Model>
): IMedusaInternalService {
  const service = {}

  for (const [modelName, model] of Object.entries(models)) {
    const singularName = inflection.singularize(modelName)
    const pluralName = inflection.pluralize(modelName)

    // Generate retrieve method
    service[`retrieve${singularName}`] = async function(
      id: string,
      config?: FindConfig,
      context?: Context
    ) {
      const repository = this.container.resolve(`${modelName}Repository`)
      return await repository.find({ where: { id }, ...config }, context)
    }

    // Generate list method
    service[`list${pluralName}`] = async function(
      filters?: FilterQuery,
      config?: FindConfig,
      context?: Context
    ) {
      const repository = this.container.resolve(`${modelName}Repository`)
      return await repository.find({ where: filters, ...config }, context)
    }

    // ... generate create, update, delete, etc.
  }

  // Apply decorators
  service = applyDecorators(service)

  return service
}
```

### B. Method Decorators

Three critical decorators are applied to **every** generated method:

#### 1. @MedusaContext() - Context Extraction

```typescript
function MedusaContext() {
  return function(target, propertyName, descriptor) {
    const originalMethod = descriptor.value

    descriptor.value = async function(...args) {
      // Extract context from last parameter
      let context = args[args.length - 1]

      if (!isContext(context)) {
        context = { transactionManager: null }
        args.push(context)
      }

      // Inject transaction manager if available
      if (!context.transactionManager && this.__container__) {
        const manager = this.__container__.resolve("manager")
        context.transactionManager = manager
      }

      return await originalMethod.apply(this, args)
    }
  }
}
```

#### 2. @EmitEvents() - Event Emission

```typescript
function EmitEvents() {
  return function(target, propertyName, descriptor) {
    const originalMethod = descriptor.value

    descriptor.value = async function(...args) {
      const context = args[args.length - 1]
      const entityName = extractEntityName(propertyName) // "product" from "createProduct"
      const operation = extractOperation(propertyName)   // "created"

      const result = await originalMethod.apply(this, args)

      // Emit event
      const eventBus = this.__container__.resolve("eventBus")
      const eventName = `${entityName}.${operation}` // "product.created"

      if (context.eventGroupId) {
        // Defer emission until transaction completes
        await eventBus.stageEvent(eventName, result, {
          eventGroupId: context.eventGroupId
        })
      } else {
        // Emit immediately
        await eventBus.emit(eventName, result)
      }

      return result
    }
  }
}
```

#### 3. @InjectManager() - Transaction Manager Injection

```typescript
function InjectManager() {
  return function(target, propertyName, descriptor) {
    const originalMethod = descriptor.value

    descriptor.value = async function(...args) {
      const context = args[args.length - 1]

      // If already in a transaction, use existing manager
      if (context.transactionManager) {
        return await originalMethod.apply(this, args)
      }

      // Otherwise, create new transaction
      const manager = this.__container__.resolve("manager")

      return await manager.transaction(async (transactionManager) => {
        context.transactionManager = transactionManager
        return await originalMethod.apply(this, args)
      })
    }
  }
}
```

### C. Context Pattern

Every method in Medusa accepts `Context` as the last parameter:

```typescript
type Context = {
  transactionManager?: any           // ORM transaction handle
  manager?: any                      // ORM manager/session
  eventGroupId?: string              // Group events for atomic emission
  withDeleted?: boolean              // Include soft-deleted records
  [key: string]: any                 // Custom properties
}

// Usage
await productService.create({
  title: "T-Shirt"
}, {
  eventGroupId: "wf_123",           // Hold events until workflow completes
  transactionManager: txManager      // Share transaction across services
})
```

### D. Repository Interface

```typescript
interface RepositoryService<T> {
  // Read
  find(options?: FindOptions<T>, context?: Context): Promise<T[]>
  findAndCount(options?, context?): Promise<[T[], number]>

  // Write
  create(data: any[], context?): Promise<T[]>
  update(data: {entity: T, update: Partial<T>}[], context?): Promise<T[]>
  delete(ids: string[], context?): Promise<string[]>

  // Soft delete/restore
  softDelete(
    idsOrFilter: string[] | FilterQuery<T>,
    context?
  ): Promise<[T[], Record<string, unknown[]>]>

  restore(
    idsOrFilter: string[] | FilterQuery<T>,
    context?
  ): Promise<[T[], Record<string, unknown[]>]>

  // Upsert
  upsert(data: any[], context?): Promise<T[]>
  upsertWithReplace(
    data: any[],
    config?: UpsertWithReplaceConfig,
    context?
  ): Promise<{
    entities: T[],
    performedActions: {
      created: T[],
      updated: T[],
      deleted: T[]
    }
  }>

  // Transaction
  transaction<TResult>(
    task: (transactionManager) => Promise<TResult>,
    context?
  ): Promise<TResult>

  // Serialization
  serialize<TOutput>(data: any, options?): Promise<TOutput>
}
```

### .NET Implementation

```csharp
// Context pattern
public class Context : Dictionary<string, object?>
{
    public object? TransactionManager { get; set; }
    public string? EventGroupId { get; set; }
    public bool WithDeleted { get; set; }
}

// Repository interface
public interface IRepository<T> where T : class
{
    Task<T?> FindAsync(string id, FindOptions? options = null, Context? context = null);
    Task<IEnumerable<T>> FindAsync(FindOptions? options = null, Context? context = null);
    Task<(IEnumerable<T>, int)> FindAndCountAsync(FindOptions? options = null, Context? context = null);

    Task<IEnumerable<T>> CreateAsync(T[] data, Context? context = null);
    Task<IEnumerable<T>> UpdateAsync((T entity, T update)[] data, Context? context = null);
    Task<string[]> DeleteAsync(string[] ids, Context? context = null);

    Task<(IEnumerable<T>, Dictionary<string, string[]>)> SoftDeleteAsync(
        string[] ids, Context? context = null);
    Task<(IEnumerable<T>, Dictionary<string, string[]>)> RestoreAsync(
        string[] ids, Context? context = null);

    Task<IEnumerable<T>> UpsertAsync(T[] data, Context? context = null);
}

// Base service factory (using reflection or source generators)
public class MedusaService<TModels>
{
    public static IMedusaInternalService Create(Dictionary<string, Type> models)
    {
        var serviceType = typeof(MedusaInternalService<>).MakeGenericType(models.Values.ToArray());
        var instance = Activator.CreateInstance(serviceType);

        // Generate methods dynamically
        foreach (var (modelName, modelType) in models)
        {
            GenerateRetrieveMethod(instance, modelName, modelType);
            GenerateListMethod(instance, modelName, modelType);
            GenerateCreateMethod(instance, modelName, modelType);
            // ... etc
        }

        return (IMedusaInternalService)instance;
    }
}

// Decorators via AOP (use Castle.DynamicProxy or similar)
public class MedusaContextInterceptor : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        var args = invocation.Arguments;
        var context = args.LastOrDefault() as Context;

        if (context == null)
        {
            context = new Context();
            invocation.Arguments = args.Concat(new object[] { context }).ToArray();
        }

        // Inject transaction manager
        if (context.TransactionManager == null)
        {
            var manager = container.Resolve<ITransactionManager>();
            context.TransactionManager = manager;
        }

        invocation.Proceed();
    }
}
```

### Porting Priority
**CRITICAL - Port First** (Phase 1.2-1.3)

---

## 6. Module Communication Patterns

Modules in Medusa are **independent packages** that communicate through:
1. **Remote Query** - Cross-module data access
2. **Link Module** - Define and manage relationships
3. **Event Bus** - Async event-driven communication

### A. Remote Query System

**Purpose:** Query data across multiple modules with automatic JOINs

**Location:**
- `/packages/core/modules-sdk/src/remote-query/remote-query.ts`
- `/packages/core/orchestration/src/joiner/remote-joiner.ts`

**Example:**

```typescript
// Query products with their prices (different modules)
const products = await remoteQuery.query({
  service: "product",
  fields: [
    "id",
    "title",
    "prices.*",              // Auto-join to pricing module
    "prices.currency.*"      // Nested relationship
  ],
  args: {
    filters: { status: "active" },
    take: 10
  }
})

// Result:
// [
//   {
//     id: "prod_123",
//     title: "T-Shirt",
//     prices: [
//       { id: "price_1", amount: 1999, currency: { code: "USD" } },
//       { id: "price_2", amount: 1799, currency: { code: "EUR" } }
//     ]
//   }
// ]
```

**How it works:**

1. Each module declares `ModuleJoinerConfig`:

```typescript
// Product module
export const joinerConfig: ModuleJoinerConfig = {
  serviceName: "product",
  primaryKeys: ["id"],
  linkableKeys: {
    product_id: "Product"
  },
  relationships: [
    {
      serviceName: "pricing",
      primaryKey: "product_id",
      foreignKey: "id",
      hasMany: true
    }
  ]
}

// Pricing module
export const joinerConfig: ModuleJoinerConfig = {
  serviceName: "pricing",
  primaryKeys: ["id"],
  linkableKeys: {
    pricing_id: "Price"
  },
  relationships: [
    {
      serviceName: "product",
      primaryKey: "id",
      foreignKey: "product_id",
      hasMany: false
    }
  ]
}
```

2. RemoteJoiner builds entity graph:

```
Product (product module)
  ↓ (product.id = pricing.product_id)
Price (pricing module)
  ↓ (price.currency_id = currency.id)
Currency (currency module)
```

3. Query optimizer batches fetches:

```typescript
// Instead of:
// SELECT * FROM products WHERE id IN (...)
// For each product:
//   SELECT * FROM prices WHERE product_id = ?

// Does:
// SELECT * FROM products WHERE id IN (...)
// SELECT * FROM prices WHERE product_id IN (...) -- Batched!
```

### B. Link Module System

**Purpose:** Define many-to-many relationships between modules

**Location:** `/packages/core/modules-sdk/src/link.ts`

**Example:**

```typescript
// Define link: Product <-> Pricing
import { defineLink } from "@medusajs/modules-sdk"

export default defineLink({
  linkable: "product",
  linkable2: "pricing",
  database: {
    table: "product_pricing_link",
    idPrefix: "prodprice"
  }
})

// Create link
await link.create({
  product: { id: "prod_123" },
  pricing: { id: "price_456" }
})

// Query using link (via RemoteQuery)
const products = await remoteQuery.query({
  service: "product",
  fields: ["id", "title", "prices.*"]
})

// Delete with cascade
const [errors, deletedIds] = await link.delete({
  product: { id: ["prod_123"] }
})
// Automatically soft-deletes:
// - product: ["prod_123"]
// - pricing: ["price_456"] (if deleteCascade: true)
```

### C. Event-Driven Communication

**Example:**

```typescript
// Order module emits event
class OrderService {
  async create(data, context) {
    const order = await this.repository.create(data, context)

    // Event emitted automatically by @EmitEvents decorator
    // Event name: "order.created"
    // Event data: order

    return order
  }
}

// Fulfillment module subscribes
class FulfillmentSubscriber {
  constructor({ eventBus, fulfillmentService }) {
    eventBus.subscribe("order.created", async (event) => {
      await fulfillmentService.createFulfillment({
        orderId: event.data.id
      })
    })
  }
}

// Inventory module also subscribes
class InventorySubscriber {
  constructor({ eventBus, inventoryService }) {
    eventBus.subscribe("order.created", async (event) => {
      for (const item of event.data.items) {
        await inventoryService.reserve({
          sku: item.sku,
          quantity: item.quantity
        })
      }
    })
  }
}
```

**Grouped Events (Workflow Integration):**

```typescript
const workflow = createWorkflow("order-flow", () => {
  const context = { eventGroupId: ulid() }

  // All events are held (not emitted)
  const order = createOrderStep({ context })
  const payment = processPaymentStep({ context })
  const shipment = createShipmentStep({ context })

  // After workflow completes successfully:
  // eventBus.releaseGroupedEvents(context.eventGroupId)
  // → Emits: order.created, payment.processed, shipment.created

  // If workflow fails:
  // → Events are discarded (not emitted)
})
```

### .NET Implementation

```csharp
// Remote Query
public interface IRemoteQuery
{
    Task<T> QueryAsync<T>(
        string queryString,
        Dictionary<string, object>? variables = null
    );

    Task<T> QueryAsync<T>(RemoteJoinerQuery query);
}

public class RemoteJoinerQuery
{
    public string Service { get; set; }
    public string[] Fields { get; set; }
    public RemoteQueryArgs Args { get; set; }
}

// Link
public interface ILink
{
    Task CreateAsync(LinkDefinition link, Context? context = null);
    Task DismissAsync(LinkDefinition link, Context? context = null);
    Task<(CascadeError[]?, Dictionary<string, string[][]>)> DeleteAsync(
        Dictionary<string, Dictionary<string, object[]>> entities,
        Context? context = null
    );
}

// Event Bus
public interface IEventBus
{
    Task PublishAsync<T>(string eventName, T data, EventMetadata? metadata = null);
    void Subscribe<T>(string eventName, Func<Event<T>, Task> handler);
    Task ReleaseGroupedEventsAsync(string eventGroupId);
}
```

### Porting Priority
**MEDIUM-HIGH - Port Third** (Phase 3.1-3.3)

---

## 7. Porting Order for .NET

### Phase 1: Foundation (Weeks 1-3)

**Goal:** Core abstractions and DI container

```
├── 1.1 DI Container (Week 1)
│   ├── Port Awilix-equivalent (use Autofac or Microsoft.Extensions.DependencyInjection)
│   ├── Implement registerAdd() pattern for multi-instance registration
│   ├── Scope management (per-request scoping)
│   └── Service resolution with dependency graph
│
├── 1.2 Core Types (Week 2)
│   ├── Context class
│   ├── FindConfig<T> and FilterQuery<T>
│   ├── Event<T> and EventMetadata
│   ├── Repository interfaces (IRepository<T>)
│   └── Module interfaces (IModuleService)
│
└── 1.3 Base Service Class (Week 3)
    ├── Generate CRUD methods dynamically (reflection or source generators)
    ├── Implement decorator pattern (@MedusaContext, @EmitEvents, @InjectManager)
    │   └── Use Castle.DynamicProxy or similar AOP library
    ├── Transaction management
    └── Event emission on entity changes
```

**Deliverable:** A working DI container with basic service registration and method decoration

---

### Phase 2: Module System (Weeks 4-6)

**Goal:** Module discovery, loading, and dependency injection

```
├── 2.1 Module Definition & Registry (Week 4)
│   ├── ModuleDefinition class
│   ├── ModuleResolution class
│   ├── MedusaModule static registry
│   └── Module metadata (key, dependencies, isQueryable)
│
├── 2.2 Module Service Pattern (Week 5)
│   ├── Base module initialization
│   ├── Joiner config generation
│   ├── Lifecycle hooks (OnApplicationStart, OnApplicationShutdown)
│   └── Dependency injection into module constructors
│
└── 2.3 Module Examples (Week 6)
    ├── Implement Logger module (infrastructure)
    ├── Implement Cache module (infrastructure)
    ├── Implement Event Bus module (infrastructure)
    └── Test module loading with dependencies
```

**Deliverable:** 3 working infrastructure modules with automatic loading

---

### Phase 3: Query System (Weeks 7-9)

**Goal:** Cross-module data access and relationships

```
├── 3.1 Remote Joiner (Week 7)
│   ├── Parse ModuleJoinerConfig from modules
│   ├── Build entity relationship graph
│   ├── Query optimization (batch fetching, N+1 prevention)
│   └── Field projection (select specific fields)
│
├── 3.2 Remote Query (Week 8)
│   ├── Query language parsing (JSON-based or string-based DSL)
│   ├── Relationship traversal (dot notation: "product.prices.currency")
│   ├── Filter support (eq, ne, gt, lt, in, like, etc.)
│   └── Pagination and sorting
│
└── 3.3 Link System (Week 9)
    ├── Link definition and storage
    ├── Many-to-many relationship management
    ├── Cascade operations (soft delete with cascade)
    └── Restore operations
```

**Deliverable:** Cross-module querying with automatic JOINs

---

### Phase 4: Workflow Engine (Weeks 10-14)

**Goal:** Orchestrate multi-step operations with compensation

```
├── 4.1 Transaction Step Model (Week 10)
│   ├── TransactionStep state machine
│   ├── TransactionStepStatus enum
│   ├── Step execution model (invoke/compensate)
│   └── Step handler pattern
│
├── 4.2 Transaction Orchestrator (Week 11)
│   ├── Step graph building from workflow definition
│   ├── Dependency ordering (topological sort)
│   ├── Parallel step execution (Task.WhenAll for independent steps)
│   └── Step execution with context and container
│
├── 4.3 Compensation/Rollback (Week 12)
│   ├── Reverse-order compensation on failure
│   ├── Error handling and recovery
│   ├── Idempotency tracking (prevent duplicate compensation)
│   └── Transaction state management
│
├── 4.4 Workflow DSL (Week 13)
│   ├── CreateWorkflow<TInput, TOutput>()
│   ├── CreateStep<TInput, TOutput>(invoke, compensate)
│   ├── Composition helpers:
│   │   ├── Parallelize() - Execute steps in parallel
│   │   ├── When() - Conditional execution
│   │   └── Transform() - Data transformation
│   └── Hook system (onStepSuccess, onStepFailure, onWorkflowComplete)
│
└── 4.5 Workflow Integration (Week 14)
    ├── Container access in step handlers
    ├── Event emission from workflows (grouped events)
    ├── Timeout support per step
    ├── Retry logic with exponential backoff
    └── Checkpoint/resume for long-running workflows
```

**Deliverable:** Working workflow engine with compensation

---

### Phase 5: Framework Integration (Weeks 15-16)

**Goal:** Application loader and testing infrastructure

```
├── 5.1 App Loader (Week 15)
│   ├── Module bootstrap from configuration
│   ├── Module merging (default config + user overrides)
│   ├── Initialization coordination
│   └── Graceful shutdown
│
├── 5.2 Database Integration (Week 15)
│   ├── ORM integration (EF Core or Dapper)
│   ├── Migration system
│   ├── Transaction management (DbContext per request)
│   └── Soft delete support (global query filters)
│
└── 5.3 Testing Infrastructure (Week 16)
    ├── Mock container for unit tests
    ├── Test workflow execution
    ├── Module isolation in tests
    └── In-memory database for integration tests
```

**Deliverable:** Complete framework ready for building commerce modules

---

### Phase 6: Commerce Modules (Week 17+)

**Only after completing Phases 1-5**, build commerce modules:

```
├── Region Module (simple, no dependencies)
├── Currency Module (simple)
├── Product Module (depends on: EventBus, Logger)
├── Pricing Module (depends on: Product, Currency)
├── Inventory Module (depends on: Product)
├── Cart Module (depends on: Product, Pricing)
├── Order Module (depends on: Product, Pricing, Cart)
└── ... (other modules)
```

---

## 8. .NET Implementation Interfaces

### Minimum Viable Set

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace Medusa.Core
{
    // ============================================
    // DI CONTAINER
    // ============================================

    public interface IServiceContainer
    {
        T Resolve<T>(string? name = null);
        object Resolve(Type type, string? name = null);
        void Register<T>(string name, Func<IServiceContainer, T> factory);
        void Register<T>(string name, T instance);
        void RegisterAdd<T>(string name, Func<IServiceContainer, T> factory);
        IServiceContainer CreateScope();
    }

    // ============================================
    // CONTEXT
    // ============================================

    public class Context : Dictionary<string, object?>
    {
        public string? EventGroupId { get; set; }
        public object? TransactionManager { get; set; }
        public object? Manager { get; set; }
        public bool WithDeleted { get; set; }
    }

    // ============================================
    // QUERY/FILTER
    // ============================================

    public class FindOptions<T>
    {
        public FilterQuery<T>? Where { get; set; }
        public string[]? Select { get; set; }
        public string[]? Relations { get; set; }
        public int? Take { get; set; }
        public int? Skip { get; set; }
        public Dictionary<string, SortDirection>? Sort { get; set; }
    }

    public class FilterQuery<T>
    {
        public Dictionary<string, object>? Fields { get; set; }
        public FilterQuery<T>[]? And { get; set; }
        public FilterQuery<T>[]? Or { get; set; }
        public FilterQuery<T>? Not { get; set; }
    }

    public enum SortDirection { ASC, DESC }

    // ============================================
    // REPOSITORY
    // ============================================

    public interface IRepository<T> where T : class
    {
        Task<T?> FindAsync(string id, FindOptions<T>? options = null, Context? context = null);
        Task<IEnumerable<T>> FindAsync(FindOptions<T>? options = null, Context? context = null);
        Task<(IEnumerable<T> Items, int Count)> FindAndCountAsync(FindOptions<T>? options = null, Context? context = null);

        Task<IEnumerable<T>> CreateAsync(T[] data, Context? context = null);
        Task<IEnumerable<T>> UpdateAsync((T Entity, T Update)[] data, Context? context = null);
        Task<string[]> DeleteAsync(string[] ids, Context? context = null);

        Task<(IEnumerable<T> Items, Dictionary<string, string[]> Ids)> SoftDeleteAsync(
            string[] ids, Context? context = null);
        Task<(IEnumerable<T> Items, Dictionary<string, string[]> Ids)> RestoreAsync(
            string[] ids, Context? context = null);

        Task<IEnumerable<T>> UpsertAsync(T[] data, Context? context = null);
        Task<TResult> TransactionAsync<TResult>(Func<object, Task<TResult>> task, Context? context = null);
    }

    // ============================================
    // MODULE
    // ============================================

    public class ModuleDefinition
    {
        public string Key { get; set; }
        public string Label { get; set; }
        public string DefaultPackage { get; set; }
        public bool IsRequired { get; set; }
        public bool IsQueryable { get; set; }
        public string[] Dependencies { get; set; }
    }

    public class ModuleJoinerConfig
    {
        public string ServiceName { get; set; }
        public string[] PrimaryKeys { get; set; }
        public Dictionary<string, string[]> LinkableKeys { get; set; }
        public ModuleJoinerRelationship[] Relationships { get; set; }
    }

    public class ModuleJoinerRelationship
    {
        public string ServiceName { get; set; }
        public string PrimaryKey { get; set; }
        public string ForeignKey { get; set; }
        public bool HasMany { get; set; }
        public bool DeleteCascade { get; set; }
    }

    public interface IModuleService
    {
        ModuleDefinition Definition { get; }
        ModuleJoinerConfig JoinerConfig { get; }

        Task<(object Data, Dictionary<string, string[]> Ids)> SoftDeleteAsync(
            Dictionary<string, object[]> ids, Context? context = null);
        Task<(object Data, Dictionary<string, string[]> Ids)> RestoreAsync(
            Dictionary<string, object[]> ids, Context? context = null);
    }

    // ============================================
    // WORKFLOW
    // ============================================

    public class TransactionStep
    {
        public string Id { get; set; }
        public string Action { get; set; }
        public int Depth { get; set; }
        public string Uuid { get; set; }
        public bool Invoked { get; set; }
        public object? Result { get; set; }
        public Exception? Error { get; set; }
        public TransactionStepStatus Status { get; set; }
        public TransactionStep[] Next { get; set; }
        public TransactionStep[] Previous { get; set; }
    }

    public enum TransactionStepStatus
    {
        IDLE,
        OK,
        INVOKING,
        WAITING,
        COMPENSATING,
        REVERTED,
        FAILED,
        SKIPPED,
        TIMEOUT,
        DORMANT
    }

    public enum TransactionState
    {
        DORMANT,
        PENDING,
        PROCESSING,
        DONE,
        FAILED,
        REVERTED,
        TIMEOUT,
        SKIPPED
    }

    public class TransactionResult<T>
    {
        public T Data { get; set; }
        public TransactionState State { get; set; }
        public TransactionStep[] Steps { get; set; }
    }

    public delegate Task<TOutput> StepInvokeHandler<TInput, TOutput>(
        StepHandlerArguments<TInput> args);

    public delegate Task StepCompensateHandler<TInput>(
        StepHandlerArguments<TInput> args);

    public class StepHandlerArguments<TInput>
    {
        public TransactionStep Step { get; set; }
        public TInput Payload { get; set; }
        public Dictionary<string, object> PreviousResults { get; set; }
        public IServiceContainer Container { get; set; }
        public Context? Context { get; set; }
    }

    public class WorkflowStepHandler<TInput, TOutput>
    {
        public StepInvokeHandler<TInput, TOutput> Invoke { get; set; }
        public StepCompensateHandler<TInput>? Compensate { get; set; }
    }

    public interface IWorkflowEngine
    {
        Task<TransactionResult<TOutput>> ExecuteAsync<TInput, TOutput>(
            string workflowId,
            TInput input,
            IServiceContainer container,
            Context? context = null
        );

        Task CompensateAsync(string transactionId);
    }

    // ============================================
    // REMOTE QUERY
    // ============================================

    public class RemoteJoinerQuery
    {
        public string Service { get; set; }
        public string[] Fields { get; set; }
        public RemoteQueryArgs Args { get; set; }
    }

    public class RemoteQueryArgs
    {
        public object? Filters { get; set; }
        public int? Take { get; set; }
        public int? Skip { get; set; }
        public Dictionary<string, SortDirection>? OrderBy { get; set; }
    }

    public interface IRemoteQuery
    {
        Task<T> QueryAsync<T>(
            string queryString,
            Dictionary<string, object>? variables = null
        );

        Task<T> QueryAsync<T>(RemoteJoinerQuery query);
    }

    // ============================================
    // LINK
    // ============================================

    public class LinkDefinition
    {
        public Dictionary<string, Dictionary<string, object>> Links { get; set; }
        public Dictionary<string, object>? Data { get; set; }
    }

    public interface ILink
    {
        Task CreateAsync(LinkDefinition link, Context? context = null);
        Task DismissAsync(LinkDefinition link, Context? context = null);
        Task<(CascadeError[]? Errors, Dictionary<string, string[][]> DeletedIds)> DeleteAsync(
            Dictionary<string, Dictionary<string, object[]>> entities,
            Context? context = null
        );
    }

    public class CascadeError
    {
        public string ServiceName { get; set; }
        public string EntityName { get; set; }
        public string[] Ids { get; set; }
        public string Message { get; set; }
    }

    // ============================================
    // EVENT BUS
    // ============================================

    public class Event<T>
    {
        public string Name { get; set; }
        public EventMetadata? Metadata { get; set; }
        public T Data { get; set; }
    }

    public class EventMetadata
    {
        public string? EventGroupId { get; set; }
        public Dictionary<string, object>? Additional { get; set; }
    }

    public interface IEventBus
    {
        Task PublishAsync<T>(string eventName, T data, EventMetadata? metadata = null, Context? context = null);
        void Subscribe<T>(string eventName, Func<Event<T>, Task> handler);
        Task ReleaseGroupedEventsAsync(string eventGroupId);
    }
}
```

---

## 9. File Structure Reference

### Core Engine Files in Medusa

```
packages/core/
├── framework/src/
│   ├── container.ts                    [DI Container setup]
│   ├── medusa-app-loader.ts            [App initialization]
│   ├── database/                       [DB connection]
│   ├── migrations/                     [Migration system]
│   ├── workflows/                      [Workflow discovery]
│   └── jobs/                           [Job scheduling]
│
├── modules-sdk/src/
│   ├── medusa-module.ts                [Module registry & loader]
│   ├── definitions.ts                  [Module definitions]
│   ├── link.ts                         [Link/relationship system]
│   ├── remote-query/
│   │   ├── remote-query.ts             [Query engine]
│   │   ├── query.ts                    [Query builder]
│   │   ├── parse-filters.ts            [Filter parsing]
│   │   └── to-remote-query.ts          [Object to query conversion]
│   ├── loaders/
│   │   ├── module-loader.ts            [Load modules into container]
│   │   ├── register-modules.ts         [Register modules]
│   │   └── module-provider-loader.ts   [Load providers]
│   └── medusa-app.ts                   [App bootstrap]
│
├── workflows-sdk/src/
│   └── utils/composer/
│       ├── create-workflow.ts          [DSL entry point]
│       ├── create-step.ts              [Step creation]
│       ├── type.ts                     [Type definitions]
│       ├── parallelize.ts              [Parallel execution]
│       ├── when.ts                     [Conditional steps]
│       ├── transform.ts                [Data transformation]
│       └── helpers/                    [Response objects]
│
├── orchestration/src/
│   ├── transaction/
│   │   ├── transaction-orchestrator.ts [Step execution engine]
│   │   ├── distributed-transaction.ts  [Transaction state]
│   │   ├── transaction-step.ts         [Individual steps]
│   │   ├── orchestrator-builder.ts     [Builder pattern]
│   │   └── types.ts                    [Type definitions]
│   ├── workflow/
│   │   ├── workflow-manager.ts         [Workflow registry]
│   │   ├── local-workflow.ts           [Single-process execution]
│   │   ├── global-workflow.ts          [Distributed execution]
│   │   └── scheduler.ts                [Scheduled workflows]
│   └── joiner/
│       ├── remote-joiner.ts            [Entity join logic]
│       └── helpers.ts                  [Utility functions]
│
├── utils/src/
│   ├── common/
│   │   ├── medusa-container.ts         [Container implementation]
│   │   ├── container.ts                [Container types]
│   │   └── create-container-like.ts    [Container builder]
│   ├── modules-sdk/
│   │   ├── module.ts                   [Module wrapper]
│   │   ├── module-provider.ts          [Provider pattern]
│   │   ├── medusa-service.ts           [Base service factory]
│   │   ├── medusa-internal-service.ts  [Service interface]
│   │   ├── joiner-config-builder.ts    [Config generation]
│   │   └── definition.ts               [Module definitions]
│   ├── event-bus/
│   │   └── common.ts                   [Event types]
│   └── decorators/
│       ├── decorators.ts               [Method decorators]
│       └── medusa-context.ts           [Context decorator]
│
└── types/src/
    ├── index.ts                        [Type exports]
    ├── shared-context.ts               [Context type]
    ├── common/
    │   ├── common.ts                   [Common types]
    │   └── config-module.ts            [Configuration]
    ├── dal/
    │   ├── repository-service.ts       [Repository interface]
    │   └── index.ts                    [DAL types]
    ├── modules-sdk/
    │   ├── medusa-internal-service.ts  [Service interface]
    │   └── remote-query.ts             [Query types]
    ├── event-bus/
    │   └── common.ts                   [Event types]
    ├── joiner/
    │   └── index.ts                    [Relationship types]
    └── workflows-sdk/
        └── service.ts                  [Workflow types]
```

### Suggested .NET Project Structure

```
Medusa.Core/
├── Medusa.Core.DependencyInjection/
│   ├── IServiceContainer.cs
│   ├── ServiceContainer.cs
│   └── ServiceRegistration.cs
│
├── Medusa.Core.Types/
│   ├── Context.cs
│   ├── FindOptions.cs
│   ├── FilterQuery.cs
│   ├── Event.cs
│   └── EventMetadata.cs
│
├── Medusa.Core.Dal/
│   ├── IRepository.cs
│   ├── RepositoryBase.cs
│   └── UnitOfWork.cs
│
├── Medusa.Core.ModulesSDK/
│   ├── ModuleDefinition.cs
│   ├── ModuleResolution.cs
│   ├── MedusaModule.cs
│   ├── ModuleLoader.cs
│   ├── IModuleService.cs
│   └── ModuleJoinerConfig.cs
│
├── Medusa.Core.Workflows/
│   ├── WorkflowBuilder.cs
│   ├── TransactionOrchestrator.cs
│   ├── DistributedTransaction.cs
│   ├── TransactionStep.cs
│   └── IWorkflowEngine.cs
│
├── Medusa.Core.Query/
│   ├── IRemoteQuery.cs
│   ├── RemoteQuery.cs
│   ├── RemoteJoiner.cs
│   └── QueryParser.cs
│
├── Medusa.Core.Link/
│   ├── ILink.cs
│   ├── LinkService.cs
│   └── LinkDefinition.cs
│
├── Medusa.Core.EventBus/
│   ├── IEventBus.cs
│   ├── EventBus.cs
│   └── EventSubscriber.cs
│
├── Medusa.Core.Services/
│   ├── MedusaService.cs
│   ├── IMedusaInternalService.cs
│   └── Decorators/
│       ├── MedusaContextAttribute.cs
│       ├── EmitEventsAttribute.cs
│       └── InjectManagerAttribute.cs
│
└── Medusa.Core.Framework/
    ├── MedusaAppLoader.cs
    ├── ConfigurationLoader.cs
    └── ApplicationLifecycle.cs
```

---

## 10. Critical Design Patterns

Medusa uses these design patterns extensively:

| Pattern | Usage | Why |
|---------|-------|-----|
| **Module Pattern** | Static registry with lazy loading | Independent module deployment |
| **Factory Pattern** | Service class generation via `MedusaService` | Auto-generate CRUD methods |
| **Decorator Pattern** | Method decoration (@MedusaContext, @EmitEvents) | Cross-cutting concerns (events, transactions) |
| **Strategy Pattern** | Multiple module implementations (internal/external) | Flexibility in module types |
| **Repository Pattern** | Data access abstraction | Decouple from ORM |
| **Dependency Injection** | Constructor-based with container resolution | Loose coupling, testability |
| **Observer Pattern** | Event bus for inter-module communication | Decouple modules |
| **State Machine** | Transaction/workflow step state transitions | Track execution state |
| **Builder Pattern** | OrchestratorBuilder for workflow composition | Fluent API for workflows |
| **Chain of Responsibility** | Workflow step execution and compensation | Sequential processing with rollback |

---

## Summary

### What is Medusa's Core Engine?

1. **DI Container (Awilix wrapper)** - Maps modules/services to DI instances
2. **Module System** - Loads independent module packages with declared dependencies
3. **Service Base Classes** - Auto-generates CRUD methods from model definitions
4. **Repository Layer** - Abstracts ORM operations
5. **Remote Query** - Cross-module data access with automatic joins
6. **Link System** - Defines and manages module relationships
7. **Workflow Engine** - Orchestrates multi-step operations with automatic compensation
8. **Event Bus** - Decouples modules via async events
9. **Transaction Manager** - Ensures consistency during workflows

### Why This Architecture?

- **Modularity**: Modules can be developed, tested, and deployed independently
- **Scalability**: Services can be scaled horizontally without core changes
- **Resilience**: Workflow compensation handles failures automatically
- **Extensibility**: New modules plug in by declaring dependencies
- **Consistency**: Cross-module transactions via orchestrator

### What to Port First (Critical Path)

1. **DI Container** → Foundation for everything else
2. **Core Types** (Context, Repository, Event) → Required by all modules
3. **Module Loader** → Enables dynamic module discovery
4. **Base Service Factory** → Auto-generates CRUD operations
5. **Workflow Engine** → Orchestrates business logic
6. **Then** build commerce modules incrementally

---

## Key Insights for .NET Developers

1. **Async-First Architecture**: Medusa is fundamentally async. Every method returns `Promise<T>` (Task<T> in C#).

2. **Event-Driven**: Modules communicate primarily through events, not direct calls.

3. **Context Propagation**: The `Context` object is passed through every method to maintain transaction state, event grouping, and custom metadata.

4. **Compensation Over Rollback**: Instead of database transactions, Medusa uses compensation functions to undo side effects.

5. **Auto-Generated Services**: Most CRUD code is generated dynamically from model definitions, reducing boilerplate.

6. **Cross-Module Queries**: Remote Query allows querying across module boundaries with automatic relationship resolution.

7. **Workflow-Centric**: Business logic lives in workflows, not in services. Services are thin CRUD layers.

---

## Next Steps

1. **Study these key files first:**
   - `/packages/core/utils/src/common/medusa-container.ts` - DI Container
   - `/packages/core/modules-sdk/src/medusa-module.ts` - Module System
   - `/packages/core/utils/src/modules-sdk/medusa-service.ts` - Service Factory
   - `/packages/core/orchestration/src/transaction/transaction-orchestrator.ts` - Workflow Engine

2. **Create proof-of-concept in .NET:**
   - Implement basic DI container with `registerAdd()`
   - Create simple module with auto-generated CRUD
   - Build minimal workflow with 2-3 steps and compensation

3. **Iterate on core abstractions** before building commerce modules

4. **Test cross-module scenarios** early (querying across modules, event-driven communication)

---

**This architecture is production-ready for high-scale commerce applications. Port it carefully and you'll have a powerful framework for .NET.**
