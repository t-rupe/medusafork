# MedusaJS Architecture Overview - .NET Mapping Guide

## Purpose

This document provides a comprehensive architectural overview of MedusaJS framework and how to map its concepts to .NET 10 with FastEndpoints using VSA/REPR patterns.

## System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        HTTP Layer (Express)                       │
│  ┌────────────┐  ┌────────────┐  ┌──────────────┐               │
│  │ File-based │→ │ Middleware │→ │  Auth (JWT/  │               │
│  │  Routing   │  │  Pipeline  │  │  Session/API)│               │
│  └────────────┘  └────────────┘  └──────────────┘               │
└─────────────────────────┬────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────────┐
│                    DI Container (Awilix)                          │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐       │
│  │ Services │  │ Repositor│  │  Modules  │  │  Config  │       │
│  │ Registry │  │   ies    │  │  Loader   │  │  Logger  │       │
│  └──────────┘  └──────────┘  └───────────┘  └──────────┘       │
└─────────────────────────┬────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────────┐
│                     Workflow Engine (Orchestration)               │
│  ┌───────────┐  ┌────────────┐  ┌──────────────┐               │
│  │  Workflow │  │    Steps   │  │ Compensation │               │
│  │  Composer │→ │  Execution │→ │   (Saga)     │               │
│  └───────────┘  └────────────┘  └──────────────┘               │
└─────────────────────────┬────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────────┐
│                  Module System (Domain Modules)                   │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐   │
│  │Product │  │ Cart   │  │ Order  │  │Payment │  │Inventory│   │
│  │ Module │  │ Module │  │ Module │  │ Module │  │ Module  │   │
│  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘   │
└─────────────────────────┬────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────────┐
│              Database Layer (MikroORM + PostgreSQL)               │
│  ┌──────────┐  ┌────────────┐  ┌──────────────┐                │
│  │   DML    │  │  Reposit   │  │  Migrations  │                │
│  │  Entities│→ │  ories     │→ │  & Indexes   │                │
│  └──────────┘  └────────────┘  └──────────────┘                │
└──────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. HTTP Layer
**Purpose:** API endpoint routing, authentication, validation, request/response handling

**MedusaJS:**
- Express-based with file-based routing (`/api/admin/products/route.ts`)
- Middleware pipeline (auth, CORS, validation, error handling)
- Route discovery and registration
- Typed request/response objects

**→ .NET Mapping:**
- **FastEndpoints** for endpoint definition
- **ASP.NET Core middleware** pipeline
- **FluentValidation** for request validation
- **JWT/Cookie authentication**

**See:** `packages/core/framework/src/http/ANALYSIS.md`

### 2. Dependency Injection
**Purpose:** Service registration, scoping, module loading

**MedusaJS:**
- Awilix container with string-based registration
- Auto-registration of repositories and services
- Request scoping via `container.createScope()`
- Collection registration with `registerAdd`

**→ .NET Mapping:**
- **Built-in ASP.NET Core DI**
- **Scrutor** for assembly scanning
- Automatic per-request scoping
- `IEnumerable<T>` for collections

**See:** `packages/core/framework/src/CONTAINER_ANALYSIS.md`

### 3. Database Layer
**Purpose:** Data access, entity management, migrations, transactions

**MedusaJS:**
- Custom DML (Data Modeling Language) for entity definition
- MikroORM for PostgreSQL
- Repository pattern with generic base
- Soft deletes, upserts, relation management

**→ .NET Mapping:**
- **Entity Framework Core**
- **Fluent API** for entity configuration
- **Generic repository** pattern
- **Query filters** for soft deletes

**See:** `packages/core/utils/src/dal/ANALYSIS.md`

### 4. Workflow Engine
**Purpose:** Distributed transactions, saga pattern, compensation logic

**MedusaJS:**
- Workflow = composition of steps
- Step = invoke + compensate functions
- Transaction orchestrator manages execution
- Automatic rollback on failures
- Async/background execution support

**→ .NET Mapping:**
- **MassTransit Sagas** (distributed)
- **WorkflowCore** (in-process)
- **Outbox pattern** for reliability
- **State machine** for compensation

**See:** `packages/core/workflows-sdk/src/WORKFLOWS_ANALYSIS.md`

### 5. Module System
**Purpose:** Domain-driven modular architecture

**MedusaJS:**
- Self-contained domain modules (Product, Cart, Order, etc.)
- Each module has: entities, repositories, services, workflows
- Module loader auto-registers components
- Inter-module communication via Links

**→ .NET Mapping:**
- **Vertical Slice Architecture** (VSA)
- **Feature folders** per domain
- **Module extension methods** for DI
- **MediatR** for cross-module communication

### 6. Event System
**Purpose:** Asynchronous event handling, pub/sub

**MedusaJS:**
- Event bus module (Redis or Local)
- Subscribers auto-registered
- Event emission from workflows
- Typed event payloads

**→ .NET Mapping:**
- **MassTransit** message bus
- **MediatR** notifications (in-process)
- **RabbitMQ/Azure Service Bus** (distributed)

### 7. Jobs & Scheduling
**Purpose:** Background job processing, scheduled tasks

**MedusaJS:**
- Job definitions with cron schedules
- Workflow engine for async execution
- Redis-backed queue

**→ .NET Mapping:**
- **Hangfire** for background jobs
- **Quartz.NET** for scheduling
- **MassTransit** scheduling

## Request Flow Example

### MedusaJS: Create Product

```
1. HTTP Request
   POST /admin/products
   { "title": "Shirt", "price": 1000 }

2. Express Router
   → Middleware: Authentication (JWT)
   → Middleware: Body validation (Zod)
   → Route handler: POST /admin/products

3. Route Handler
   → Resolve container scope
   → Execute createProductWorkflow

4. Workflow Execution
   Step 1: createProductStep
     → Invoke: productService.create(input)
     → Store result + compensation data
   Step 2: indexProductStep
     → Invoke: indexService.index(product)
     → Store result
   Step 3: emitProductCreatedEvent
     → Invoke: eventBus.emit("product.created")

   If Step 2 fails:
     → Execute Step 1 compensation
     → productService.delete(product.id)

5. Response
   { "product": { "id": "prod_123", "title": "Shirt" } }
```

### .NET Equivalent

```csharp
// 1. FastEndpoints Endpoint
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
        // 3. Execute workflow via MediatR or direct call
        var command = new CreateProductCommand
        {
            Title = req.Title,
            Price = req.Price
        };

        var result = await SendAsync(command, ct);

        // 5. Response
        await SendAsync(new ProductResponse
        {
            Product = result
        }, cancellation: ct);
    }
}

// 4. Workflow (using WorkflowCore)
public class CreateProductWorkflow : IWorkflow<CreateProductInput, Product>
{
    public void Build(IWorkflowBuilder<CreateProductInput, Product> builder)
    {
        builder
            .StartWith<CreateProductStep>()
                .CompensateWith<DeleteProductStep>()
            .Then<IndexProductStep>()
            .Then<EmitProductCreatedEventStep>();
    }
}

// Steps
public class CreateProductStep : StepBody
{
    private readonly IProductService _productService;

    public override async Task<ExecutionResult> RunAsync(IStepExecutionContext context)
    {
        var input = context.Workflow.Data;
        var product = await _productService.Create(new Product
        {
            Title = input.Title,
            Price = input.Price
        });

        context.Workflow.Data.Product = product;
        return ExecutionResult.Next();
    }
}
```

## .NET Project Structure

```
Medusa.NET/
├── src/
│   ├── Medusa.Core/
│   │   ├── Domain/                   # Domain entities
│   │   ├── Abstractions/             # Interfaces
│   │   └── Common/                   # Shared utilities
│   │
│   ├── Medusa.Infrastructure/
│   │   ├── Data/
│   │   │   ├── MedusaDbContext.cs
│   │   │   ├── Repositories/
│   │   │   └── Migrations/
│   │   ├── Messaging/                # Event bus
│   │   └── Jobs/                     # Background jobs
│   │
│   ├── Medusa.Modules/
│   │   ├── Product/
│   │   │   ├── Entities/
│   │   │   ├── Endpoints/            # FastEndpoints
│   │   │   ├── Workflows/            # WorkflowCore workflows
│   │   │   ├── Services/
│   │   │   └── ProductModule.cs      # DI registration
│   │   │
│   │   ├── Cart/
│   │   ├── Order/
│   │   └── Payment/
│   │
│   └── Medusa.API/
│       ├── Program.cs
│       ├── Middleware/
│       └── appsettings.json
│
└── tests/
    ├── Medusa.UnitTests/
    └── Medusa.IntegrationTests/
```

## Key Design Patterns

### 1. REPR (Request-Endpoint-Response)
**MedusaJS:** Implicit via file-based routing
**.NET:** Explicit with FastEndpoints

```csharp
// Request
public class GetProductRequest
{
    public string Id { get; set; }
}

// Endpoint
public class GetProductEndpoint : Endpoint<GetProductRequest, ProductResponse>
{
    public override async Task HandleAsync(GetProductRequest req, CancellationToken ct)
    {
        var product = await SendAsync(new GetProductQuery { Id = req.Id }, ct);
        await SendAsync(new ProductResponse { Product = product }, ct);
    }
}

// Response
public class ProductResponse
{
    public Product Product { get; set; }
}
```

### 2. VSA (Vertical Slice Architecture)
**MedusaJS:** Modules (Product, Cart, Order)
**.NET:** Feature folders

```
/Modules/Product/
  /CreateProduct/
    CreateProductEndpoint.cs
    CreateProductRequest.cs
    CreateProductResponse.cs
    CreateProductValidator.cs
    CreateProductWorkflow.cs
  /GetProduct/
    GetProductEndpoint.cs
    ...
```

### 3. Repository Pattern
**MedusaJS:** Generic `MikroOrmBaseRepository<T>`
**.NET:** Generic `EfCoreRepository<T>`

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(string id, CancellationToken ct);
    Task<List<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct);
    Task<T> CreateAsync(T entity, CancellationToken ct);
    Task UpdateAsync(T entity, CancellationToken ct);
    Task DeleteAsync(string id, CancellationToken ct);
}
```

### 4. Saga Pattern (Workflows)
**MedusaJS:** Workflow + Steps with compensation
**.NET:** MassTransit State Machine or WorkflowCore

```csharp
// WorkflowCore with compensation
builder
    .StartWith<ReserveInventoryStep>()
        .CompensateWith<ReleaseInventoryStep>()
    .Then<ChargePaymentStep>()
        .CompensateWith<RefundPaymentStep>();
```

## Technology Stack Comparison

| Layer | MedusaJS | .NET 10 |
|-------|----------|---------|
| **Runtime** | Node.js + TypeScript | .NET 10 + C# 12 |
| **HTTP Framework** | Express | ASP.NET Core + FastEndpoints |
| **DI Container** | Awilix | Built-in DI |
| **ORM** | MikroORM | Entity Framework Core |
| **Database** | PostgreSQL | PostgreSQL (Npgsql) |
| **Validation** | Zod | FluentValidation |
| **Workflows** | Custom orchestration | MassTransit / WorkflowCore |
| **Event Bus** | Custom + Redis | MassTransit + RabbitMQ |
| **Jobs** | Custom + Redis | Hangfire / Quartz.NET |
| **Logging** | Winston | Serilog / Microsoft.Extensions.Logging |
| **Config** | Custom | Microsoft.Extensions.Configuration |
| **Testing** | Jest | xUnit / NUnit |

## Essential NuGet Packages

```xml
<PackageReference Include="FastEndpoints" Version="5.*" />
<PackageReference Include="FluentValidation" Version="11.*" />
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.*" />
<PackageReference Include="EFCore.NamingConventions" Version="8.*" />
<PackageReference Include="Serilog.AspNetCore" Version="8.*" />
<PackageReference Include="MassTransit" Version="8.*" />
<PackageReference Include="MassTransit.RabbitMQ" Version="8.*" />
<PackageReference Include="WorkflowCore" Version="3.*" />
<PackageReference Include="Hangfire" Version="1.*" />
<PackageReference Include="MediatR" Version="12.*" />
<PackageReference Include="Scrutor" Version="4.*" />
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

// FastEndpoints
builder.Services.AddFastEndpoints();

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { /* config */ });

// Modules
builder.Services.AddProductModule();
builder.Services.AddCartModule();
builder.Services.AddOrderModule();

// Workflows
builder.Services.AddWorkflow<CreateProductWorkflow>();

// MassTransit
builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq://localhost");
        cfg.ConfigureEndpoints(context);
    });
});

// Hangfire
builder.Services.AddHangfire(config =>
    config.UsePostgreSqlStorage(builder.Configuration.GetConnectionString("Medusa")));

var app = builder.Build();

// Middleware
app.UseAuthentication();
app.UseAuthorization();

// FastEndpoints
app.UseFastEndpoints();

// Hangfire
app.UseHangfireDashboard();

app.Run();
```

## Migration Strategy

### Phase 1: Core Infrastructure
1. Set up .NET project structure
2. Configure Entity Framework Core with PostgreSQL
3. Implement FastEndpoints for HTTP layer
4. Set up DI container and module registration

### Phase 2: Data Layer
1. Map MedusaJS entities to EF Core
2. Implement generic repository pattern
3. Create migrations from DML schemas
4. Implement soft delete query filters

### Phase 3: Business Logic
1. Port domain services to .NET
2. Implement workflows with WorkflowCore
3. Set up MassTransit for events
4. Configure Hangfire for background jobs

### Phase 4: API Layer
1. Create FastEndpoints for all routes
2. Implement authentication/authorization
3. Add request validation with FluentValidation
4. Configure CORS and middleware pipeline

### Phase 5: Testing & Deployment
1. Write unit tests (xUnit)
2. Write integration tests
3. Set up CI/CD
4. Deploy to production

## Key Differences to Note

1. **Type Safety**: C# is statically typed vs TypeScript's structural typing
2. **Async/Await**: C# requires explicit `async Task` vs TS implicit promises
3. **DI**: .NET uses constructor injection primarily vs property injection in Awilix
4. **Routing**: FastEndpoints is class-based vs file-based in Medusa
5. **ORM**: EF Core has better LINQ support vs MikroORM's builder pattern
6. **Workflows**: More mature libraries in .NET (MassTransit, WorkflowCore)

## References

**Analysis Documents:**
- `/packages/core/framework/src/http/ANALYSIS.md` - HTTP Layer
- `/packages/core/framework/src/CONTAINER_ANALYSIS.md` - DI Container
- `/packages/core/utils/src/dal/ANALYSIS.md` - Database Layer
- `/packages/core/workflows-sdk/src/WORKFLOWS_ANALYSIS.md` - Workflow Engine

**MedusaJS Documentation:**
- https://docs.medusajs.com

**.NET Resources:**
- https://learn.microsoft.com/en-us/aspnet/core
- https://fast-endpoints.com
- https://masstransit.io
- https://workflowcore.io
