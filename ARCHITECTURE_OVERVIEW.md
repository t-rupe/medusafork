# MedusaJS Architecture Overview - .NET Mapping with FastEndpoints

## Purpose

This document provides a comprehensive architectural overview of MedusaJS framework and how to map its concepts to .NET 10 with **FastEndpoints native features only** - no commercial 3rd-party dependencies.

## FastEndpoints Native Capabilities

FastEndpoints provides everything needed to replicate MedusaJS functionality:

1. **Job Queues** - Background job processing with Redis/EF Core/MongoDB persistence
2. **Event Bus** - In-process pub/sub pattern for decoupled event handling
3. **Command Bus** - In-process command execution with single handlers
4. **Endpoints** - REPR pattern for API routes
5. **Validation** - Built-in FluentValidation integration
6. **Testing** - Integrated testing framework

**NO NEED FOR:** MassTransit, Hangfire, Quartz.NET, MediatR, or other commercial libraries.

## System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    FastEndpoints HTTP Layer                       │
│  ┌────────────┐  ┌────────────┐  ┌──────────────┐               │
│  │  Endpoint  │→ │ Validation │→ │ Auth/Policies │               │
│  │   REPR     │  │  Pipeline  │  │   Pipeline    │               │
│  └────────────┘  └────────────┘  └──────────────┘               │
└─────────────────────────┬────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────────┐
│                   Built-in DI Container                           │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐       │
│  │ Services │  │ Repositor│  │  Command  │  │   Job    │       │
│  │ Registry │  │   ies    │  │  Handlers │  │  Queues  │       │
│  └──────────┘  └──────────┘  └───────────┘  └──────────┘       │
└─────────────────────────┬────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────────┐
│              FastEndpoints Event Bus + Command Bus                │
│  ┌───────────┐  ┌────────────┐  ┌──────────────┐               │
│  │  Commands │  │   Events   │  │   Job Queue  │               │
│  │  (sync)   │→ │  (pub/sub) │→ │   (async)    │               │
│  └───────────┘  └────────────┘  └──────────────┘               │
└─────────────────────────┬────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────────┐
│              Domain Modules (VSA Features)                        │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐                │
│  │Product │  │ Cart   │  │ Order  │  │Payment │                │
│  │Feature │  │Feature │  │Feature │  │Feature │                │
│  └────────┘  └────────┘  └────────┘  └────────┘                │
└─────────────────────────┬────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────────┐
│              Database Layer (EF Core + PostgreSQL)                │
│  ┌──────────┐  ┌────────────┐  ┌──────────────┐                │
│  │ Entities │  │  DbContext │  │  Repositories│                │
│  │  Config  │→ │  Migrations│→ │   Pattern    │                │
│  └──────────┘  └────────────┘  └──────────────┘                │
└──────────────────────────────────────────────────────────────────┘
```

## Request Flow Example

### MedusaJS: Create Product Workflow

```
1. HTTP Request
   POST /admin/products
   { "title": "Shirt", "price": 1000 }

2. Express Router
   → Middleware: Authentication (JWT)
   → Middleware: Body validation (Zod)
   → Route handler: POST /admin/products

3. Route Handler
   → Execute createProductWorkflow

4. Workflow Execution
   Step 1: createProductStep
     → Invoke: productService.create(input)
   Step 2: indexProductStep
     → Invoke: indexService.index(product)
   Step 3: emitProductCreatedEvent
     → Invoke: eventBus.emit("product.created")

   If Step 2 fails:
     → Compensation: productService.delete(product.id)

5. Response
   { "product": { "id": "prod_123", "title": "Shirt" } }
```

### .NET with FastEndpoints: Create Product

```csharp
// 1. Endpoint (REPR Pattern)
public class CreateProductEndpoint : Endpoint<CreateProductRequest, ProductResponse>
{
    public override void Configure()
    {
        Post("/admin/products");
        Policies("Admin");
    }

    public override async Task HandleAsync(
        CreateProductRequest req,
        CancellationToken ct)
    {
        // 3. Execute Command (replaces workflow)
        var result = await new CreateProductCommand
        {
            Title = req.Title,
            Price = req.Price
        }.ExecuteAsync(ct);

        // 5. Response
        await SendOkAsync(new ProductResponse
        {
            Product = result
        }, ct);
    }
}

// 4. Command Handler (replaces workflow step)
public class CreateProductHandler : ICommandHandler<CreateProductCommand, Product>
{
    private readonly IProductService _productService;
    private readonly ILogger<CreateProductHandler> _logger;

    public CreateProductHandler(
        IProductService productService,
        ILogger<CreateProductHandler> logger)
    {
        _productService = productService;
        _logger = logger;
    }

    public async Task<Product> ExecuteAsync(
        CreateProductCommand cmd,
        CancellationToken ct)
    {
        try
        {
            // Create product
            var product = await _productService.CreateAsync(new Product
            {
                Title = cmd.Title,
                Price = cmd.Price
            }, ct);

            // Publish event (async handlers can index, send notifications, etc.)
            await new ProductCreatedEvent
            {
                ProductId = product.Id,
                Title = product.Title
            }.PublishAsync(Mode.WaitForNone, ct);

            return product;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create product");
            throw;
        }
    }
}

// Event Handler for indexing (decoupled)
public class IndexProductHandler : IEventHandler<ProductCreatedEvent>
{
    private readonly ISearchIndexService _indexService;

    public async Task HandleAsync(ProductCreatedEvent evt, CancellationToken ct)
    {
        await _indexService.IndexProductAsync(evt.ProductId, ct);
    }
}

// Event Handler for notifications (decoupled)
public class NotifyProductCreatedHandler : IEventHandler<ProductCreatedEvent>
{
    private readonly INotificationService _notificationService;

    public async Task HandleAsync(ProductCreatedEvent evt, CancellationToken ct)
    {
        await _notificationService.SendAsync(
            $"New product created: {evt.Title}",
            ct);
    }
}
```

## Implementing Saga Pattern with FastEndpoints

For complex multi-step workflows with compensation (like MedusaJS workflows), use **Commands + Job Queues + Events**:

### Pattern: Saga with Job Queue

```csharp
// 1. Main Command - Orchestrates the saga
public class CreateOrderCommand : ICommand<OrderResult>
{
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class CreateOrderHandler : ICommandHandler<CreateOrderCommand, OrderResult>
{
    public async Task<OrderResult> ExecuteAsync(
        CreateOrderCommand cmd,
        CancellationToken ct)
    {
        var sagaId = Guid.NewGuid();

        // Step 1: Create order (synchronous)
        var order = await _orderService.CreateAsync(cmd, ct);

        // Step 2: Queue job for inventory reservation (async with compensation)
        await new ReserveInventoryJob
        {
            SagaId = sagaId,
            OrderId = order.Id,
            Items = cmd.Items
        }.QueueJobAsync(ct);

        // Step 3: Queue job for payment (async with compensation)
        await new ChargePaymentJob
        {
            SagaId = sagaId,
            OrderId = order.Id,
            Amount = order.Total
        }.QueueJobAsync(ct);

        return new OrderResult { OrderId = order.Id, Status = "Processing" };
    }
}

// 2. Job Handler with Compensation
public class ReserveInventoryJobHandler
    : ICommandHandler<ReserveInventoryJob>
{
    public async Task ExecuteAsync(ReserveInventoryJob job, CancellationToken ct)
    {
        try
        {
            await _inventoryService.ReserveAsync(job.Items, ct);

            // Publish success event
            await new InventoryReservedEvent
            {
                SagaId = job.SagaId,
                OrderId = job.OrderId
            }.PublishAsync(ct);
        }
        catch (Exception ex)
        {
            // Publish failure event to trigger compensation
            await new InventoryReservationFailedEvent
            {
                SagaId = job.SagaId,
                OrderId = job.OrderId,
                Reason = ex.Message
            }.PublishAsync(ct);

            throw;
        }
    }
}

// 3. Compensation Event Handler
public class CompensateOrderHandler : IEventHandler<InventoryReservationFailedEvent>
{
    public async Task HandleAsync(
        InventoryReservationFailedEvent evt,
        CancellationToken ct)
    {
        // Cancel the order
        await _orderService.CancelAsync(evt.OrderId, ct);

        // Release any reserved resources
        await _inventoryService.ReleaseAsync(evt.OrderId, ct);

        // Notify customer
        await new OrderCancelledEvent
        {
            OrderId = evt.OrderId,
            Reason = evt.Reason
        }.PublishAsync(ct);
    }
}
```

### Job Storage with EF Core

```csharp
// Job Record Entity
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

// Job Storage Provider
public class EfCoreJobStorageProvider
    : IJobStorageProvider<JobRecord>
{
    private readonly MedusaDbContext _db;

    public async Task StoreJobAsync(JobRecord job, CancellationToken ct)
        => await _db.JobRecords.AddAsync(job, ct);

    public async Task<IEnumerable<JobRecord>> GetNextBatchAsync(
        PendingSearchParams<JobRecord> p)
    {
        return await _db.JobRecords
            .Where(p.Match)
            .OrderBy(j => j.ExecuteAfter)
            .Take(p.Limit)
            .ToListAsync(p.CancellationToken);
    }

    public async Task MarkJobAsCompleteAsync(JobRecord job, CancellationToken ct)
    {
        job.IsComplete = true;
        await _db.SaveChangesAsync(ct);
    }

    public async Task PurgeStaleJobsAsync(
        StaleJobSearchParams<JobRecord> p)
    {
        var staleJobs = _db.JobRecords.Where(p.Match);
        _db.JobRecords.RemoveRange(staleJobs);
        await _db.SaveChangesAsync(p.CancellationToken);
    }
}
```

## .NET Project Structure (VSA)

```
Medusa.NET/
├── src/
│   ├── Medusa.Core/
│   │   ├── Domain/
│   │   │   ├── Product.cs
│   │   │   ├── Order.cs
│   │   │   └── Cart.cs
│   │   ├── Common/
│   │   │   ├── Result.cs
│   │   │   └── Error.cs
│   │   └── Abstractions/
│   │       └── IRepository.cs
│   │
│   ├── Medusa.Infrastructure/
│   │   ├── Data/
│   │   │   ├── MedusaDbContext.cs
│   │   │   ├── JobRecord.cs
│   │   │   ├── EfCoreJobStorageProvider.cs
│   │   │   └── Repositories/
│   │   │       └── ProductRepository.cs
│   │   └── Migrations/
│   │
│   ├── Medusa.Features/  (VSA - Vertical Slices)
│   │   ├── Products/
│   │   │   ├── Create/
│   │   │   │   ├── CreateProductEndpoint.cs
│   │   │   │   ├── CreateProductCommand.cs
│   │   │   │   ├── CreateProductHandler.cs
│   │   │   │   ├── CreateProductValidator.cs
│   │   │   │   └── ProductCreatedEvent.cs
│   │   │   ├── Get/
│   │   │   │   ├── GetProductEndpoint.cs
│   │   │   │   └── GetProductQuery.cs
│   │   │   ├── Update/
│   │   │   └── Delete/
│   │   │
│   │   ├── Orders/
│   │   │   ├── Create/
│   │   │   │   ├── CreateOrderEndpoint.cs
│   │   │   │   ├── CreateOrderCommand.cs
│   │   │   │   ├── ReserveInventoryJob.cs
│   │   │   │   └── ChargePaymentJob.cs
│   │   │   └── ...
│   │   │
│   │   └── Cart/
│   │
│   └── Medusa.API/
│       ├── Program.cs
│       └── appsettings.json
│
└── tests/
    └── Medusa.Tests/
        ├── Products/
        └── Orders/
```

## Technology Stack

| Layer | MedusaJS | .NET with FastEndpoints |
|-------|----------|------------------------|
| **HTTP** | Express | FastEndpoints (built-in) |
| **DI** | Awilix | ASP.NET Core DI (built-in) |
| **ORM** | MikroORM | Entity Framework Core |
| **Database** | PostgreSQL | PostgreSQL (Npgsql) |
| **Validation** | Zod | FluentValidation (built-in) |
| **Workflows** | Custom orchestration | Commands + Job Queues |
| **Events** | Custom + Redis | Event Bus (built-in) |
| **Jobs** | Custom + Redis | Job Queues (built-in) |
| **Command Bus** | N/A | Command Bus (built-in) |

## Essential NuGet Packages

```xml
<!-- FastEndpoints Core -->
<PackageReference Include="FastEndpoints" Version="5.*" />

<!-- Database -->
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.*" />
<PackageReference Include="EFCore.NamingConventions" Version="8.*" />

<!-- Logging -->
<PackageReference Include="Serilog.AspNetCore" Version="8.*" />

<!-- Optional: Redis for Job Queue persistence -->
<PackageReference Include="StackExchange.Redis" Version="2.*" />

<!-- Testing -->
<PackageReference Include="FastEndpoints.Testing" Version="5.*" />
<PackageReference Include="xunit" Version="2.*" />
```

## Program.cs Setup

```csharp
var builder = WebApplication.CreateBuilder(args);

// Logging
builder.Host.UseSerilog();

// Database
builder.Services.AddDbContext<MedusaDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Medusa"))
        .UseSnakeCaseNamingConvention());

// FastEndpoints with all features
builder.Services
    .AddFastEndpoints()
    .AddJobQueues<JobRecord, EfCoreJobStorageProvider>(); // Job queues with EF Core

// Authentication
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { /* config */ });

// Domain Services
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped(typeof(IRepository<>), typeof(EfCoreRepository<>));

var app = builder.Build();

// FastEndpoints
app.UseAuthentication()
   .UseAuthorization()
   .UseFastEndpoints()
   .UseJobQueues(o =>
   {
       o.MaxConcurrency = 4;  // Limit concurrent job execution
       o.ExecutionTimeLimit = TimeSpan.FromMinutes(5);
   });

app.Run();
```

## Key Design Patterns

### 1. Command Pattern (Replaces Simple Workflows)

```csharp
// Command
public class UpdateInventoryCommand : ICommand<Result>
{
    public string ProductId { get; set; }
    public int Quantity { get; set; }
}

// Handler
public class UpdateInventoryHandler
    : ICommandHandler<UpdateInventoryCommand, Result>
{
    public async Task<Result> ExecuteAsync(
        UpdateInventoryCommand cmd,
        CancellationToken ct)
    {
        // Business logic here
        await _inventoryService.UpdateAsync(cmd.ProductId, cmd.Quantity, ct);
        return Result.Success();
    }
}

// Usage in endpoint
var result = await new UpdateInventoryCommand
{
    ProductId = req.ProductId,
    Quantity = req.Quantity
}.ExecuteAsync(ct);
```

### 2. Event Bus Pattern (Replaces Event Subscribers)

```csharp
// Event
public class OrderPlacedEvent : IEvent
{
    public string OrderId { get; set; }
    public decimal Total { get; set; }
}

// Multiple Handlers (decoupled)
public class SendOrderConfirmationHandler : IEventHandler<OrderPlacedEvent>
{
    public async Task HandleAsync(OrderPlacedEvent evt, CancellationToken ct)
    {
        await _emailService.SendOrderConfirmationAsync(evt.OrderId, ct);
    }
}

public class UpdateAnalyticsHandler : IEventHandler<OrderPlacedEvent>
{
    public async Task HandleAsync(OrderPlacedEvent evt, CancellationToken ct)
    {
        await _analyticsService.TrackOrderAsync(evt.OrderId, evt.Total, ct);
    }
}

// Publish (fire and forget or wait for all)
await new OrderPlacedEvent
{
    OrderId = order.Id,
    Total = order.Total
}.PublishAsync(Mode.WaitForNone, ct);
```

### 3. Job Queue Pattern (Replaces Background Jobs)

```csharp
// Job Command
public class SendEmailJob : ICommand
{
    public string To { get; set; }
    public string Subject { get; set; }
    public string Body { get; set; }
}

// Job Handler
public class SendEmailJobHandler : ICommandHandler<SendEmailJob>
{
    public async Task ExecuteAsync(SendEmailJob job, CancellationToken ct)
    {
        await _emailService.SendAsync(job.To, job.Subject, job.Body, ct);
    }
}

// Queue job (async execution)
await new SendEmailJob
{
    To = "customer@example.com",
    Subject = "Order Confirmation",
    Body = "Your order has been placed"
}.QueueJobAsync(ct);

// Queue with delay
await new SendEmailJob { ... }
    .QueueJobAsync(
        executeAfter: DateTime.UtcNow.AddMinutes(30),
        expireOn: DateTime.UtcNow.AddHours(2),
        ct);
```

## Migration Strategy

### Phase 1: Core Infrastructure
1. Set up .NET project with FastEndpoints
2. Configure EF Core with PostgreSQL
3. Set up job queue with EF Core storage provider
4. Implement base repository pattern

### Phase 2: Data Layer
1. Map entities to EF Core
2. Create migrations
3. Implement soft delete query filters
4. Set up DbContext pooling

### Phase 3: Features (VSA)
1. Port endpoints to FastEndpoints REPR
2. Convert workflows to Commands + Job Queues
3. Convert event subscribers to Event Handlers
4. Implement validation with FluentValidation

### Phase 4: Testing
1. Write integration tests with FastEndpoints.Testing
2. Write unit tests for handlers
3. Test job queue execution
4. Test event pub/sub

## Key Differences from Commercial Libraries

**Why FastEndpoints over MassTransit/Hangfire:**
1. **All-in-one**: Single library, consistent API
2. **No commercial licensing**: 100% free, even for enterprise
3. **Simpler setup**: Less configuration, faster dev
4. **Better DX**: Strongly typed, no magic strings
5. **Built-in testing**: Integrated test framework

**Trade-offs:**
- FastEndpoints Event Bus is **in-process only** (fine for monoliths)
- For distributed systems, use Redis pub/sub or SignalR
- Job Queues need persistence setup (EF Core/Redis/MongoDB)

## References

**Analysis Documents:**
- `/packages/core/framework/src/http/ANALYSIS.md` - HTTP Layer
- `/packages/core/framework/src/CONTAINER_ANALYSIS.md` - DI Container
- `/packages/core/utils/src/dal/ANALYSIS.md` - Database Layer
- `/packages/core/workflows-sdk/src/WORKFLOWS_ANALYSIS.md` - Workflow Engine

**FastEndpoints Documentation:**
- https://fast-endpoints.com/docs/command-bus
- https://fast-endpoints.com/docs/event-bus
- https://fast-endpoints.com/docs/job-queues
- https://github.com/FastEndpoints/Job-Queue-EF-Core-Demo
