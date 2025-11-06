# Medusa .NET Port - FastEndpoints Architecture Guide

## Executive Summary

This document outlines how to build Medusa's modular architecture using **FastEndpoints** and the **REPR pattern** (Request-Endpoint-Response) while maintaining the core decoupling that makes Medusa powerful.

**Key Decision**: Use FastEndpoints for modern, performant endpoints while organizing code into **independent module assemblies** for true decoupling.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                  FastEndpoints Layer                    │
│         (REPR: Request → Endpoint → Response)           │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│               Workflow Orchestrator                     │
│  (Business logic, compensation, state management)       │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│       Module System (Independent Assemblies)            │
│  (Product, Pricing, Inventory, Order modules)           │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│              Core Engine                                │
│  (DI, Events, Remote Query, Workflows)                  │
└─────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
Medusa.NET/
│
├── src/
│   ├── Medusa.Core/                          # Core engine
│   │   ├── Medusa.Core.DependencyInjection/
│   │   ├── Medusa.Core.Workflows/
│   │   ├── Medusa.Core.EventBus/
│   │   ├── Medusa.Core.RemoteQuery/
│   │   └── Medusa.Core.Modules/
│   │
│   ├── Modules/                              # Independent module assemblies
│   │   ├── Medusa.Product.Module/
│   │   │   ├── Features/                     # VSA organization
│   │   │   │   ├── CreateProduct/
│   │   │   │   │   ├── Endpoint.cs           # FastEndpoint
│   │   │   │   │   ├── Request.cs
│   │   │   │   │   ├── Response.cs
│   │   │   │   │   ├── Validator.cs
│   │   │   │   │   └── Mapper.cs
│   │   │   │   ├── GetProduct/
│   │   │   │   ├── ListProducts/
│   │   │   │   └── UpdateProduct/
│   │   │   ├── Domain/
│   │   │   │   ├── Product.cs                # Entity
│   │   │   │   └── ProductVariant.cs
│   │   │   ├── Data/
│   │   │   │   ├── ProductRepository.cs
│   │   │   │   └── ProductDbContext.cs
│   │   │   ├── Services/
│   │   │   │   └── ProductService.cs         # Business logic
│   │   │   ├── Events/
│   │   │   │   ├── ProductCreated.cs
│   │   │   │   └── ProductUpdated.cs
│   │   │   ├── Workflows/
│   │   │   │   └── CreateProductWorkflow.cs
│   │   │   └── ProductModule.cs              # Module definition
│   │   │
│   │   ├── Medusa.Pricing.Module/
│   │   │   ├── Features/
│   │   │   │   ├── CreatePrice/
│   │   │   │   └── UpdatePrice/
│   │   │   ├── Domain/
│   │   │   ├── Services/
│   │   │   ├── Subscribers/                  # Event subscribers
│   │   │   │   └── ProductCreatedSubscriber.cs
│   │   │   └── PricingModule.cs
│   │   │
│   │   ├── Medusa.Inventory.Module/
│   │   ├── Medusa.Cart.Module/
│   │   ├── Medusa.Order.Module/
│   │   └── Medusa.Customer.Module/
│   │
│   └── Medusa.API/                           # API host
│       ├── Program.cs
│       ├── appsettings.json
│       └── GlobalUsings.cs
│
└── tests/
    ├── Medusa.Product.Module.Tests/
    ├── Medusa.Pricing.Module.Tests/
    └── Medusa.Core.Tests/
```

---

## Module Definition

Every module implements `IModule`:

```csharp
// Medusa.Product.Module/ProductModule.cs

using Medusa.Core.Modules;

namespace Medusa.Product.Module;

public class ProductModule : IModule
{
    public ModuleDefinition Definition => new()
    {
        Key = "product",
        Label = "Product",
        IsRequired = true,
        IsQueryable = true,
        Dependencies = new[] { "eventBus", "logger" }
    };

    public ModuleJoinerConfig JoinerConfig => new()
    {
        ServiceName = "product",
        PrimaryKeys = new[] { "id" },
        LinkableKeys = new Dictionary<string, string[]>
        {
            { "product_id", new[] { "Product" } }
        },
        Relationships = new[]
        {
            new ModuleJoinerRelationship
            {
                ServiceName = "pricing",
                PrimaryKey = "product_id",
                ForeignKey = "id",
                HasMany = true,
                DeleteCascade = false
            }
        }
    };

    public void ConfigureServices(IServiceCollection services)
    {
        // Register module services
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<ProductRepository>();

        // Register DbContext
        services.AddDbContext<ProductDbContext>(options =>
            options.UseNpgsql("connection-string"));

        // Auto-discover FastEndpoints in this assembly
        services.AddFastEndpointsFromAssembly(typeof(ProductModule).Assembly);
    }

    public void ConfigureApp(IApplicationBuilder app)
    {
        // Module-specific middleware (if needed)
    }

    public async Task OnApplicationStartAsync(IServiceProvider serviceProvider)
    {
        // Run migrations, seed data, etc.
        using var scope = serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ProductDbContext>();
        await dbContext.Database.MigrateAsync();
    }

    public async Task OnApplicationShutdownAsync(IServiceProvider serviceProvider)
    {
        // Cleanup resources
    }
}
```

---

## FastEndpoint Example (REPR Pattern)

### Create Product Endpoint

```csharp
// Medusa.Product.Module/Features/CreateProduct/Endpoint.cs

using FastEndpoints;
using Medusa.Core.Workflows;

namespace Medusa.Product.Module.Features.CreateProduct;

public class Endpoint : Endpoint<Request, Response>
{
    private readonly IWorkflowEngine _workflowEngine;
    private readonly IServiceContainer _container;

    public Endpoint(IWorkflowEngine workflowEngine, IServiceContainer container)
    {
        _workflowEngine = workflowEngine;
        _container = container;
    }

    public override void Configure()
    {
        Post("/products");
        AllowAnonymous(); // Or use auth: Roles("admin")
        Description(b => b
            .WithName("CreateProduct")
            .WithTags("Products")
            .Produces<Response>(201)
            .ProducesProblemDetails(400));
    }

    public override async Task HandleAsync(Request req, CancellationToken ct)
    {
        // Execute workflow (not direct service call!)
        var result = await _workflowEngine.ExecuteAsync<Request, Response>(
            "create-product-workflow",
            req,
            _container,
            new Context { EventGroupId = Ulid.NewUlid().ToString() }
        );

        if (result.State == TransactionState.DONE)
        {
            await SendCreatedAtAsync<GetProductEndpoint>(
                new { id = result.Data.Id },
                result.Data,
                cancellation: ct);
        }
        else
        {
            await SendErrorsAsync(cancellation: ct);
        }
    }
}
```

### Request

```csharp
// Medusa.Product.Module/Features/CreateProduct/Request.cs

namespace Medusa.Product.Module.Features.CreateProduct;

public class Request
{
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public string Handle { get; set; } = string.Empty;
    public bool IsGiftcard { get; set; }
    public List<CreateVariantRequest> Variants { get; set; } = new();
}

public class CreateVariantRequest
{
    public string Title { get; set; } = string.Empty;
    public string Sku { get; set; } = string.Empty;
    public int Inventory { get; set; }
}
```

### Response

```csharp
// Medusa.Product.Module/Features/CreateProduct/Response.cs

namespace Medusa.Product.Module.Features.CreateProduct;

public class Response
{
    public string Id { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public string Handle { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
    public List<VariantResponse> Variants { get; set; } = new();
    public DateTime CreatedAt { get; set; }
}

public class VariantResponse
{
    public string Id { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public string Sku { get; set; } = string.Empty;
}
```

### Validator

```csharp
// Medusa.Product.Module/Features/CreateProduct/Validator.cs

using FastEndpoints;
using FluentValidation;

namespace Medusa.Product.Module.Features.CreateProduct;

public class Validator : Validator<Request>
{
    public Validator()
    {
        RuleFor(x => x.Title)
            .NotEmpty()
            .MaximumLength(200);

        RuleFor(x => x.Handle)
            .NotEmpty()
            .Matches("^[a-z0-9-]+$")
            .WithMessage("Handle must be lowercase alphanumeric with hyphens");

        RuleForEach(x => x.Variants)
            .SetValidator(new VariantValidator());
    }
}

public class VariantValidator : Validator<CreateVariantRequest>
{
    public VariantValidator()
    {
        RuleFor(x => x.Title).NotEmpty();
        RuleFor(x => x.Sku).NotEmpty();
        RuleFor(x => x.Inventory).GreaterThanOrEqualTo(0);
    }
}
```

### Mapper

```csharp
// Medusa.Product.Module/Features/CreateProduct/Mapper.cs

using FastEndpoints;
using Medusa.Product.Module.Domain;

namespace Medusa.Product.Module.Features.CreateProduct;

public class Mapper : Mapper<Request, Response, Domain.Product>
{
    public override Domain.Product ToEntity(Request r)
    {
        return new Domain.Product
        {
            Title = r.Title,
            Description = r.Description,
            Handle = r.Handle,
            IsGiftcard = r.IsGiftcard,
            Status = ProductStatus.Draft,
            Variants = r.Variants.Select(v => new ProductVariant
            {
                Title = v.Title,
                Sku = v.Sku,
                Inventory = v.Inventory
            }).ToList()
        };
    }

    public override Response FromEntity(Domain.Product e)
    {
        return new Response
        {
            Id = e.Id,
            Title = e.Title,
            Handle = e.Handle,
            Status = e.Status.ToString(),
            Variants = e.Variants.Select(v => new VariantResponse
            {
                Id = v.Id,
                Title = v.Title,
                Sku = v.Sku
            }).ToList(),
            CreatedAt = e.CreatedAt
        };
    }
}
```

---

## Workflow Integration

### Create Product Workflow

```csharp
// Medusa.Product.Module/Workflows/CreateProductWorkflow.cs

using Medusa.Core.Workflows;

namespace Medusa.Product.Module.Workflows;

public static class CreateProductWorkflow
{
    public static void Register(IWorkflowRegistry registry)
    {
        var workflow = WorkflowBuilder.CreateWorkflow<CreateProductRequest, ProductResponse>(
            "create-product-workflow",
            (input) =>
            {
                // Step 1: Create product
                var product = CreateProductStep(input);

                // Step 2: Create default prices (in parallel)
                var prices = CreateDefaultPricesStep(product);

                // Step 3: Reserve product handle
                var handleReservation = ReserveHandleStep(product);

                // Step 4: Emit events
                var events = EmitProductCreatedStep(product);

                return new WorkflowResponse<ProductResponse>(product);
            }
        );

        registry.Register(workflow);
    }

    private static WorkflowStep<CreateProductRequest, Domain.Product> CreateProductStep(
        CreateProductRequest input)
    {
        return WorkflowBuilder.CreateStep<CreateProductRequest, Domain.Product>(
            "create-product",
            async (args) =>
            {
                var productService = args.Container.Resolve<IProductService>();
                var product = await productService.CreateAsync(input, args.Context);
                return product;
            },
            async (args) =>
            {
                // COMPENSATION: Delete the product
                var productService = args.Container.Resolve<IProductService>();
                var product = (Domain.Product)args.Step.Result!;
                await productService.DeleteAsync(product.Id, args.Context);
            }
        );
    }

    private static WorkflowStep<Domain.Product, List<Price>> CreateDefaultPricesStep(
        Domain.Product product)
    {
        return WorkflowBuilder.CreateStep<Domain.Product, List<Price>>(
            "create-default-prices",
            async (args) =>
            {
                // This calls Pricing module via event bus!
                var eventBus = args.Container.Resolve<IEventBus>();
                await eventBus.PublishAsync("product.prices.default-needed", new
                {
                    ProductId = product.Id
                }, args.Context);

                return new List<Price>();
            },
            async (args) =>
            {
                // COMPENSATION: Delete default prices
                var eventBus = args.Container.Resolve<IEventBus>();
                await eventBus.PublishAsync("product.prices.rollback", new
                {
                    ProductId = product.Id
                }, args.Context);
            }
        );
    }
}
```

---

## Service Layer (Business Logic)

```csharp
// Medusa.Product.Module/Services/ProductService.cs

using Medusa.Core.Services;
using Medusa.Core.Types;

namespace Medusa.Product.Module.Services;

public interface IProductService
{
    Task<Product> CreateAsync(CreateProductRequest input, Context? context = null);
    Task<Product?> GetAsync(string id, Context? context = null);
    Task<(List<Product>, int)> ListAsync(FilterQuery<Product>? filters = null, Context? context = null);
    Task<Product> UpdateAsync(string id, UpdateProductRequest input, Context? context = null);
    Task DeleteAsync(string id, Context? context = null);
}

public class ProductService : IProductService
{
    private readonly ProductRepository _repository;
    private readonly IEventBus _eventBus;
    private readonly Context _context;

    public ProductService(
        ProductRepository repository,
        IEventBus eventBus,
        Context context)
    {
        _repository = repository;
        _eventBus = eventBus;
        _context = context;
    }

    public async Task<Product> CreateAsync(CreateProductRequest input, Context? context = null)
    {
        context ??= _context;

        var product = new Product
        {
            Id = Ulid.NewUlid().ToString(),
            Title = input.Title,
            Description = input.Description,
            Handle = input.Handle,
            Status = ProductStatus.Draft,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };

        await _repository.CreateAsync(new[] { product }, context);

        // Events are automatically emitted via decorator
        // But you can also manually emit:
        await _eventBus.PublishAsync("product.created", new
        {
            product.Id,
            product.Title,
            product.Handle
        }, new EventMetadata { EventGroupId = context.EventGroupId }, context);

        return product;
    }

    public async Task<Product?> GetAsync(string id, Context? context = null)
    {
        return await _repository.FindAsync(id, new FindOptions<Product>
        {
            Relations = new[] { "variants" }
        }, context);
    }

    public async Task<(List<Product>, int)> ListAsync(
        FilterQuery<Product>? filters = null,
        Context? context = null)
    {
        var (products, count) = await _repository.FindAndCountAsync(
            new FindOptions<Product>
            {
                Where = filters,
                Relations = new[] { "variants" }
            },
            context);

        return (products.ToList(), count);
    }

    public async Task DeleteAsync(string id, Context? context = null)
    {
        await _repository.SoftDeleteAsync(new[] { id }, context);

        await _eventBus.PublishAsync("product.deleted", new { Id = id }, context);
    }
}
```

---

## Event Subscribers (Cross-Module Communication)

```csharp
// Medusa.Pricing.Module/Subscribers/ProductCreatedSubscriber.cs

using Medusa.Core.EventBus;

namespace Medusa.Pricing.Module.Subscribers;

public class ProductCreatedSubscriber : BackgroundService
{
    private readonly IEventBus _eventBus;
    private readonly IServiceProvider _serviceProvider;

    public ProductCreatedSubscriber(IEventBus eventBus, IServiceProvider serviceProvider)
    {
        _eventBus = eventBus;
        _serviceProvider = serviceProvider;
    }

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _eventBus.Subscribe<ProductCreatedEvent>("product.created", async (evt) =>
        {
            using var scope = _serviceProvider.CreateScope();
            var pricingService = scope.ServiceProvider.GetRequiredService<IPricingService>();

            // Create default prices for the product
            await pricingService.CreateDefaultPricesAsync(new CreateDefaultPricesRequest
            {
                ProductId = evt.Data.Id
            });
        });

        return Task.CompletedTask;
    }
}

public class ProductCreatedEvent
{
    public string Id { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public string Handle { get; set; } = string.Empty;
}
```

---

## Remote Query (Cross-Module Data Access)

```csharp
// Medusa.API/Features/Products/GetProductWithPrices.cs

using FastEndpoints;
using Medusa.Core.RemoteQuery;

namespace Medusa.API.Features.Products;

public class GetProductWithPricesEndpoint : EndpointWithoutRequest<ProductWithPricesResponse>
{
    private readonly IRemoteQuery _remoteQuery;

    public GetProductWithPricesEndpoint(IRemoteQuery remoteQuery)
    {
        _remoteQuery = remoteQuery;
    }

    public override void Configure()
    {
        Get("/products/{id}/with-prices");
        AllowAnonymous();
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var id = Route<string>("id");

        // Query across Product and Pricing modules!
        var result = await _remoteQuery.QueryAsync<ProductWithPricesResponse>(
            new RemoteJoinerQuery
            {
                Service = "product",
                Fields = new[]
                {
                    "id",
                    "title",
                    "handle",
                    "prices.*",                  // Auto-JOIN to pricing module
                    "prices.currency.*",         // Nested relationship
                    "variants.*",
                    "variants.prices.*"
                },
                Args = new RemoteQueryArgs
                {
                    Filters = new { id = id }
                }
            }
        );

        await SendAsync(result, cancellation: ct);
    }
}

public class ProductWithPricesResponse
{
    public string Id { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public string Handle { get; set; } = string.Empty;
    public List<PriceDto> Prices { get; set; } = new();
    public List<VariantDto> Variants { get; set; } = new();
}
```

---

## Application Bootstrap

```csharp
// Medusa.API/Program.cs

using FastEndpoints;
using Medusa.Core.Modules;

var builder = WebApplication.CreateBuilder(args);

// Add Medusa Core Services
builder.Services.AddMedusaCore(options =>
{
    options.DatabaseConnectionString = builder.Configuration.GetConnectionString("Postgres");
    options.RedisConnectionString = builder.Configuration.GetConnectionString("Redis");
});

// Auto-discover and register all modules
builder.Services.AddMedusaModules(options =>
{
    options.ScanAssemblies("Medusa.*.Module.dll");

    // Or explicitly register modules:
    // options.AddModule<ProductModule>();
    // options.AddModule<PricingModule>();
    // options.AddModule<InventoryModule>();
});

// Add FastEndpoints
builder.Services.AddFastEndpoints();

// Add Swagger
builder.Services.AddSwaggerDoc(settings =>
{
    settings.Title = "Medusa Commerce API";
    settings.Version = "v2.0";
});

var app = builder.Build();

// Initialize modules
await app.Services.InitializeMedusaModulesAsync();

// Use FastEndpoints
app.UseFastEndpoints(config =>
{
    config.Endpoints.RoutePrefix = "api";
    config.Endpoints.Configurator = ep =>
    {
        // Global endpoint configuration
        ep.Description(b => b.Produces(401));
    };
});

// Use Swagger
app.UseSwaggerGen();

await app.RunAsync();
```

---

## Testing

### Unit Test (Endpoint)

```csharp
// Medusa.Product.Module.Tests/Features/CreateProduct/EndpointTests.cs

using FastEndpoints.Testing;

namespace Medusa.Product.Module.Tests.Features.CreateProduct;

public class EndpointTests : TestBase<Program>
{
    [Fact]
    public async Task Should_Create_Product_Successfully()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Title = "Test Product",
            Handle = "test-product",
            Variants = new List<CreateVariantRequest>
            {
                new() { Title = "Default", Sku = "TEST-001", Inventory = 100 }
            }
        };

        // Act
        var (response, result) = await Client.POSTAsync<
            CreateProduct.Endpoint,
            CreateProduct.Request,
            CreateProduct.Response>(request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        result.Id.Should().NotBeNullOrEmpty();
        result.Title.Should().Be("Test Product");
        result.Handle.Should().Be("test-product");
    }

    [Fact]
    public async Task Should_Fail_Validation_When_Title_Empty()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Title = "",  // Invalid
            Handle = "test"
        };

        // Act
        var (response, result) = await Client.POSTAsync<
            CreateProduct.Endpoint,
            CreateProduct.Request,
            ErrorResponse>(request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
        result.Errors.Should().ContainKey("Title");
    }
}
```

### Integration Test (Workflow)

```csharp
// Medusa.Product.Module.Tests/Workflows/CreateProductWorkflowTests.cs

using Medusa.Core.Workflows;

namespace Medusa.Product.Module.Tests.Workflows;

public class CreateProductWorkflowTests : IClassFixture<WorkflowTestFixture>
{
    private readonly IWorkflowEngine _workflowEngine;
    private readonly IServiceContainer _container;

    public CreateProductWorkflowTests(WorkflowTestFixture fixture)
    {
        _workflowEngine = fixture.WorkflowEngine;
        _container = fixture.Container;
    }

    [Fact]
    public async Task Should_Create_Product_And_Default_Prices()
    {
        // Arrange
        var input = new CreateProductRequest
        {
            Title = "Test Product",
            Handle = "test-product"
        };

        // Act
        var result = await _workflowEngine.ExecuteAsync<CreateProductRequest, ProductResponse>(
            "create-product-workflow",
            input,
            _container
        );

        // Assert
        result.State.Should().Be(TransactionState.DONE);
        result.Data.Id.Should().NotBeNullOrEmpty();

        // Verify product was created
        var productService = _container.Resolve<IProductService>();
        var product = await productService.GetAsync(result.Data.Id);
        product.Should().NotBeNull();
    }

    [Fact]
    public async Task Should_Rollback_Product_When_Pricing_Fails()
    {
        // Arrange
        var input = new CreateProductRequest
        {
            Title = "Test Product",
            Handle = "test-product"
        };

        // Mock pricing service to throw
        var pricingService = _container.Resolve<IPricingService>();
        Mock.Get(pricingService)
            .Setup(x => x.CreateDefaultPricesAsync(It.IsAny<CreateDefaultPricesRequest>()))
            .ThrowsAsync(new Exception("Pricing service failed"));

        // Act
        var result = await _workflowEngine.ExecuteAsync<CreateProductRequest, ProductResponse>(
            "create-product-workflow",
            input,
            _container
        );

        // Assert
        result.State.Should().Be(TransactionState.REVERTED);

        // Verify product was rolled back
        var productService = _container.Resolve<IProductService>();
        var product = await productService.GetAsync(result.Data.Id);
        product.Should().BeNull(); // Compensation deleted it
    }
}
```

---

## Key FastEndpoints Features for Medusa

### 1. Pre/Post Processors

```csharp
// Medusa.Core/Processors/EventGroupProcessor.cs

using FastEndpoints;
using Medusa.Core.Types;

namespace Medusa.Core.Processors;

// Add event group ID to all requests
public class EventGroupPreProcessor : IGlobalPreProcessor
{
    public async Task PreProcessAsync(IPreProcessorContext ctx, CancellationToken ct)
    {
        var context = ctx.HttpContext.RequestServices.GetRequiredService<Context>();
        context.EventGroupId = Ulid.NewUlid().ToString();

        await Task.CompletedTask;
    }
}

// Release grouped events after successful response
public class EventGroupPostProcessor : IGlobalPostProcessor
{
    public async Task PostProcessAsync(IPostProcessorContext ctx, CancellationToken ct)
    {
        if (ctx.HttpContext.Response.StatusCode >= 200 &&
            ctx.HttpContext.Response.StatusCode < 300)
        {
            var eventBus = ctx.HttpContext.RequestServices.GetRequiredService<IEventBus>();
            var context = ctx.HttpContext.RequestServices.GetRequiredService<Context>();

            if (context.EventGroupId != null)
            {
                await eventBus.ReleaseGroupedEventsAsync(context.EventGroupId);
            }
        }
    }
}
```

### 2. Response Caching

```csharp
// Medusa.Product.Module/Features/GetProduct/Endpoint.cs

public class GetProductEndpoint : Endpoint<GetProductRequest, ProductResponse>
{
    public override void Configure()
    {
        Get("/products/{id}");

        // Cache for 5 minutes
        Options(x => x.CacheOutput(p => p.Expire(TimeSpan.FromMinutes(5))));
    }
}
```

### 3. Rate Limiting

```csharp
// Medusa.API/Program.cs

builder.Services.AddFastEndpoints(options =>
{
    options.Throttle(t =>
    {
        t.HeaderName = "X-Rate-Limit";
        t.Message = "Too many requests";
        t.Limits = new[]
        {
            new Rule(100, TimeSpan.FromMinutes(1)),  // 100 req/min
            new Rule(1000, TimeSpan.FromHours(1))    // 1000 req/hour
        };
    });
});
```

### 4. Command Bus (for CQRS)

```csharp
// Medusa.Product.Module/Commands/CreateProductCommand.cs

using FastEndpoints;

namespace Medusa.Product.Module.Commands;

public class CreateProductCommand : ICommand<ProductResponse>
{
    public string Title { get; set; } = string.Empty;
    public string Handle { get; set; } = string.Empty;
}

public class CreateProductCommandHandler : ICommandHandler<CreateProductCommand, ProductResponse>
{
    private readonly IWorkflowEngine _workflowEngine;
    private readonly IServiceContainer _container;

    public async Task<ProductResponse> ExecuteAsync(
        CreateProductCommand command,
        CancellationToken ct)
    {
        var result = await _workflowEngine.ExecuteAsync<CreateProductCommand, ProductResponse>(
            "create-product-workflow",
            command,
            _container
        );

        return result.Data;
    }
}

// In endpoint:
public class CreateProductEndpoint : Endpoint<CreateProductRequest, ProductResponse>
{
    public override async Task HandleAsync(CreateProductRequest req, CancellationToken ct)
    {
        var command = new CreateProductCommand
        {
            Title = req.Title,
            Handle = req.Handle
        };

        var result = await command.ExecuteAsync(ct);
        await SendAsync(result, cancellation: ct);
    }
}
```

---

## Module Communication Patterns

### 1. Event-Driven (Async)

```csharp
// Product module
await _eventBus.PublishAsync("product.created", new { ProductId = id });

// Pricing module subscribes
_eventBus.Subscribe<ProductCreatedEvent>("product.created", async evt => {
    await _pricingService.CreateDefaultPricesAsync(evt.Data.ProductId);
});
```

### 2. Remote Query (Sync)

```csharp
// Query across modules
var products = await _remoteQuery.QueryAsync(new {
    Service = "product",
    Fields = new[] { "id", "title", "prices.*" }
});
```

### 3. Workflow Orchestration

```csharp
var workflow = WorkflowBuilder.CreateWorkflow("checkout", (input) => {
    var order = CreateOrderStep(input);        // Order module
    var inventory = ReserveInventoryStep(order);  // Inventory module
    var payment = ChargePaymentStep(order);    // Payment module

    return new { order, inventory, payment };
});
```

---

## Advantages of This Architecture

### 1. True Modularity
- Each module is an independent assembly
- Modules don't reference each other
- Can be developed, tested, deployed independently

### 2. FastEndpoints Benefits
- ✅ REPR pattern (Request-Endpoint-Response)
- ✅ No controller bloat
- ✅ Better performance than MVC
- ✅ Built-in validation, mapping, testing
- ✅ Swagger auto-generation
- ✅ Rate limiting, caching, auth built-in

### 3. Medusa-Style Decoupling
- ✅ Event bus for async communication
- ✅ Remote Query for cross-module data access
- ✅ Workflows for orchestration
- ✅ Automatic compensation on failure

### 4. Developer Experience
- ✅ Feature-focused organization (VSA)
- ✅ Less boilerplate than MVC
- ✅ Easy to test
- ✅ Clear separation of concerns

---

## Migration Path (If You're Concerned About Adoption)

You can **support both FastEndpoints AND traditional controllers**:

```csharp
// Product Module can have BOTH:

// FastEndpoints version
public class CreateProductEndpoint : Endpoint<CreateProductRequest, ProductResponse>
{
    // Modern, fast, clean
}

// MVC fallback (for teams that prefer it)
[ApiController]
[Route("api/v1/products")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult<ProductResponse>> Create([FromBody] CreateProductRequest req)
    {
        // Same workflow, different adapter
        var result = await _workflowEngine.ExecuteAsync(...);
        return Ok(result.Data);
    }
}
```

**Business logic stays in workflows/services** - endpoints/controllers are just thin adapters.

---

## Summary

1. **Use FastEndpoints** - Modern, performant, less boilerplate
2. **Organize as Modules** - Independent assemblies for true decoupling
3. **Communicate via Events** - Async, decoupled communication
4. **Query via Remote Query** - Cross-module data access
5. **Orchestrate via Workflows** - Business logic with automatic compensation

This architecture gives you:
- ✅ Medusa's modular decoupling
- ✅ FastEndpoints' modern developer experience
- ✅ Production-ready, scalable architecture
- ✅ Easy migration path if needed

**You get the best of both worlds!**
