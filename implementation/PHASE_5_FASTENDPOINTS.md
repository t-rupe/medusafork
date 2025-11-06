# Phase 5: FastEndpoints Integration (2-3 weeks)

## Goal
Integrate FastEndpoints as the API layer with Redis output caching, Command Bus for CQRS, and proper event grouping for workflows.

## Prerequisites
- ✅ Phase 1: Foundation complete
- ✅ Phase 2: Module System complete
- ✅ Phase 3: Event Bus complete
- ✅ Phase 4: Workflows complete
- Redis installed and running

## Deliverables
- ✅ FastEndpoints configured with REPR pattern
- ✅ Redis output caching working
- ✅ Command Bus integrated
- ✅ Pre/Post processors for event grouping
- ✅ Rate limiting and validation
- ✅ Swagger documentation

---

## Step 1: Install FastEndpoints (30 min)

### 1.1 Create API Project

```bash
dotnet new web -n Medusa.API -o src/Medusa.API
dotnet sln add src/Medusa.API
```

### 1.2 Install Packages

```bash
cd src/Medusa.API

# FastEndpoints
dotnet add package FastEndpoints --version 5.25.0
dotnet add package FastEndpoints.Swagger --version 5.25.0

# Redis Output Caching
dotnet add package Microsoft.AspNetCore.OutputCaching.StackExchangeRedis --version 8.0.0

# Add core project references
dotnet add reference ../Medusa.Core/Medusa.Core.Types
dotnet add reference ../Medusa.Core/Medusa.Core.DependencyInjection
dotnet add reference ../Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Abstractions
dotnet add reference ../Medusa.Core/Medusa.Core.Workflows
```

### 1.3 Update appsettings.json

```json
{
  "ConnectionStrings": {
    "Postgres": "Host=localhost;Database=medusa;Username=medusa;Password=medusa",
    "Redis": "localhost:6379"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Redis": {
    "InstanceName": "medusa:",
    "CacheDuration": 300
  },
  "RateLimit": {
    "PermitLimit": 100,
    "Window": 60
  }
}
```

---

## Step 2: Configure Program.cs (1-2 hours)

### 2.1 Basic Setup

```csharp
// src/Medusa.API/Program.cs

using FastEndpoints;
using FastEndpoints.Swagger;
using Medusa.Core.DependencyInjection;
using Medusa.Core.EventBus;
using Medusa.Core.Workflows;

var builder = WebApplication.CreateBuilder(args);

// ============================================
// 1. REDIS OUTPUT CACHE
// ============================================
builder.Services.AddStackExchangeRedisOutputCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = builder.Configuration["Redis:InstanceName"];
});

builder.Services.AddOutputCache(options =>
{
    // Default policy: cache for 5 minutes
    options.AddBasePolicy(builder => builder
        .Expire(TimeSpan.FromMinutes(5))
        .Tag("default"));

    // Product-specific policy
    options.AddPolicy("products", builder => builder
        .Expire(TimeSpan.FromMinutes(10))
        .Tag("products"));

    // Short-lived for dynamic data
    options.AddPolicy("dynamic", builder => builder
        .Expire(TimeSpan.FromSeconds(30))
        .Tag("dynamic"));
});

// ============================================
// 2. MEDUSA CORE SERVICES
// ============================================
builder.Services.AddMedusaCore(options =>
{
    options.DatabaseConnectionString = builder.Configuration.GetConnectionString("Postgres");
    options.RedisConnectionString = builder.Configuration.GetConnectionString("Redis");
});

// ============================================
// 3. FASTENDPOINTS
// ============================================
builder.Services.AddFastEndpoints(options =>
{
    // Global configuration
    options.Assemblies = new[]
    {
        typeof(Program).Assembly,
        // Add module assemblies here as you create them
    };
});

// ============================================
// 4. SWAGGER
// ============================================
builder.Services.SwaggerDocument(options =>
{
    options.DocumentSettings = s =>
    {
        s.Title = "Medusa Commerce API";
        s.Version = "v2.0";
        s.Description = "Headless commerce platform built with .NET";
    };
});

// ============================================
// 5. CORS (for frontend apps)
// ============================================
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins(
                builder.Configuration.GetSection("AllowedOrigins").Get<string[]>()
                ?? new[] { "http://localhost:3000" })
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});

var app = builder.Build();

// ============================================
// MIDDLEWARE PIPELINE
// ============================================

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseCors();

// Output caching BEFORE FastEndpoints
app.UseOutputCache();

// FastEndpoints
app.UseFastEndpoints(config =>
{
    config.Endpoints.RoutePrefix = "api";

    config.Endpoints.Configurator = ep =>
    {
        // Global error responses
        ep.Description(b => b
            .Produces(401)
            .Produces(403)
            .Produces(500));
    };

    config.Serializer.Options.PropertyNamingPolicy = System.Text.Json.JsonNamingPolicy.CamelCase;
});

// Swagger
app.UseSwaggerGen();

await app.RunAsync();
```

---

## Step 3: Pre/Post Processors for Event Grouping (2-3 hours)

### 3.1 Event Group Pre-Processor

```csharp
// src/Medusa.API/Processors/EventGroupPreProcessor.cs

using FastEndpoints;
using Medusa.Core.Types;

namespace Medusa.API.Processors;

/// <summary>
/// Creates an event group ID for every request.
/// Events emitted during the request are staged, not emitted immediately.
/// </summary>
public class EventGroupPreProcessor : IGlobalPreProcessor
{
    public async Task PreProcessAsync(IPreProcessorContext ctx, CancellationToken ct)
    {
        // Create a scoped Context for this request
        var context = ctx.HttpContext.RequestServices.GetRequiredService<Context>();

        // Generate unique event group ID using Ulid
        context.EventGroupId = Ulid.NewUlid().ToString();

        // Store request start time for logging
        context["requestStartTime"] = DateTime.UtcNow;

        await Task.CompletedTask;
    }
}
```

### 3.2 Event Group Post-Processor

```csharp
// src/Medusa.API/Processors/EventGroupPostProcessor.cs

using FastEndpoints;
using Medusa.Core.EventBus;
using Medusa.Core.Types;

namespace Medusa.API.Processors;

/// <summary>
/// Releases grouped events if request was successful.
/// Clears them if request failed.
/// </summary>
public class EventGroupPostProcessor : IGlobalPostProcessor
{
    private readonly ILogger<EventGroupPostProcessor> _logger;

    public EventGroupPostProcessor(ILogger<EventGroupPostProcessor> logger)
    {
        _logger = logger;
    }

    public async Task PostProcessAsync(IPostProcessorContext ctx, CancellationToken ct)
    {
        var eventBus = ctx.HttpContext.RequestServices.GetRequiredService<IEventBus>();
        var context = ctx.HttpContext.RequestServices.GetRequiredService<Context>();

        if (context.EventGroupId == null)
            return;

        var statusCode = ctx.HttpContext.Response.StatusCode;
        var isSuccess = statusCode >= 200 && statusCode < 300;

        if (isSuccess)
        {
            // Release all staged events atomically
            _logger.LogInformation("Releasing event group {EventGroupId}", context.EventGroupId);
            await eventBus.ReleaseGroupedEventsAsync(context.EventGroupId);
        }
        else
        {
            // Discard all staged events (request failed)
            _logger.LogWarning("Clearing event group {EventGroupId} due to failed request (status {StatusCode})",
                context.EventGroupId, statusCode);
            await eventBus.ClearGroupedEventsAsync(context.EventGroupId);
        }

        // Log request duration
        if (context.TryGetValue("requestStartTime", out var startTimeObj) && startTimeObj is DateTime startTime)
        {
            var duration = DateTime.UtcNow - startTime;
            _logger.LogInformation("Request completed in {DurationMs}ms", duration.TotalMilliseconds);
        }
    }
}
```

### 3.3 Register Processors

```csharp
// Add to Program.cs in the FastEndpoints configuration

builder.Services.AddFastEndpoints(options =>
{
    options.Assemblies = new[] { typeof(Program).Assembly };
});

// Register processors
app.Services.AddScoped<Context>(); // Per-request context
```

Update `UseFastEndpoints`:

```csharp
app.UseFastEndpoints(config =>
{
    config.Endpoints.RoutePrefix = "api";

    // Register global processors
    config.GlobalPreProcessors.Add<EventGroupPreProcessor>();
    config.GlobalPostProcessors.Add<EventGroupPostProcessor>();
});
```

---

## Step 4: Example Endpoints with REPR Pattern (3-4 hours)

### 4.1 Health Check Endpoint

```csharp
// src/Medusa.API/Features/Health/Endpoint.cs

using FastEndpoints;

namespace Medusa.API.Features.Health;

public class Endpoint : EndpointWithoutRequest<Response>
{
    public override void Configure()
    {
        Get("/health");
        AllowAnonymous();

        // Cache for 30 seconds
        Options(o => o.CacheOutput(p => p
            .Expire(TimeSpan.FromSeconds(30))
            .Tag("health")));

        Description(b => b
            .WithName("HealthCheck")
            .WithTags("System")
            .Produces<Response>(200));
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        await SendAsync(new Response
        {
            Status = "healthy",
            Version = "1.0.0",
            Timestamp = DateTime.UtcNow
        }, cancellation: ct);
    }
}

public class Response
{
    public string Status { get; set; } = string.Empty;
    public string Version { get; set; } = string.Empty;
    public DateTime Timestamp { get; set; }
}
```

### 4.2 Product Endpoint (With Workflow)

```csharp
// src/Medusa.API/Features/Products/Create/Endpoint.cs

using FastEndpoints;
using Medusa.Core.Workflows;
using Medusa.Core.Types;

namespace Medusa.API.Features.Products.Create;

public class Endpoint : Endpoint<Request, Response>
{
    private readonly IWorkflowEngine _workflowEngine;
    private readonly IServiceContainer _container;
    private readonly Context _context;

    public Endpoint(
        IWorkflowEngine workflowEngine,
        IServiceContainer container,
        Context context)
    {
        _workflowEngine = workflowEngine;
        _container = container;
        _context = context;
    }

    public override void Configure()
    {
        Post("/products");

        // Require authentication (configure your auth scheme)
        // Policies("admin");
        AllowAnonymous(); // For now

        Description(b => b
            .WithName("CreateProduct")
            .WithTags("Products")
            .Produces<Response>(201)
            .ProducesProblemDetails(400)
            .ProducesProblemDetails(401));
    }

    public override async Task HandleAsync(Request req, CancellationToken ct)
    {
        // Execute workflow (not direct service call!)
        var result = await _workflowEngine.ExecuteAsync<Request, Response>(
            "create-product-workflow",
            req,
            _container,
            _context // This has EventGroupId from pre-processor
        );

        if (result.State == TransactionState.DONE)
        {
            // Success - events will be released by post-processor
            await SendCreatedAtAsync<GetProductEndpoint>(
                new { id = result.Data.Id },
                result.Data,
                cancellation: ct);
        }
        else
        {
            // Failure - events will be cleared by post-processor
            AddError("Product creation failed");
            await SendErrorsAsync(cancellation: ct);
        }
    }
}
```

```csharp
// src/Medusa.API/Features/Products/Create/Request.cs

using FluentValidation;

namespace Medusa.API.Features.Products.Create;

public class Request
{
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public string Handle { get; set; } = string.Empty;
    public bool IsGiftcard { get; set; }
    public List<VariantInput> Variants { get; set; } = new();
}

public class VariantInput
{
    public string Title { get; set; } = string.Empty;
    public string Sku { get; set; } = string.Empty;
    public int Inventory { get; set; }
}

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

public class VariantValidator : Validator<VariantInput>
{
    public VariantValidator()
    {
        RuleFor(x => x.Title).NotEmpty();
        RuleFor(x => x.Sku).NotEmpty();
        RuleFor(x => x.Inventory).GreaterThanOrEqualTo(0);
    }
}
```

```csharp
// src/Medusa.API/Features/Products/Create/Response.cs

namespace Medusa.API.Features.Products.Create;

public class Response
{
    public string Id { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public string Handle { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
}
```

---

## Step 5: Command Bus Integration (2-3 hours)

### 5.1 Define Command

```csharp
// src/Medusa.API/Commands/Products/CreateProductCommand.cs

using FastEndpoints;

namespace Medusa.API.Commands.Products;

public class CreateProductCommand : ICommand<CreateProductResult>
{
    public string Title { get; set; } = string.Empty;
    public string Handle { get; set; } = string.Empty;
    public string? Description { get; set; }
}

public class CreateProductResult
{
    public string Id { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
}
```

### 5.2 Define Command Handler

```csharp
// src/Medusa.API/Commands/Products/CreateProductCommandHandler.cs

using FastEndpoints;
using Medusa.Core.Workflows;
using Medusa.Core.Types;

namespace Medusa.API.Commands.Products;

public class CreateProductCommandHandler : ICommandHandler<CreateProductCommand, CreateProductResult>
{
    private readonly IWorkflowEngine _workflowEngine;
    private readonly IServiceContainer _container;
    private readonly Context _context;

    public CreateProductCommandHandler(
        IWorkflowEngine workflowEngine,
        IServiceContainer container,
        Context context)
    {
        _workflowEngine = workflowEngine;
        _container = container;
        _context = context;
    }

    public async Task<CreateProductResult> ExecuteAsync(
        CreateProductCommand command,
        CancellationToken ct)
    {
        var result = await _workflowEngine.ExecuteAsync<CreateProductCommand, CreateProductResult>(
            "create-product-workflow",
            command,
            _container,
            _context
        );

        if (result.State != TransactionState.DONE)
        {
            throw new Exception("Product creation failed");
        }

        return result.Data;
    }
}
```

### 5.3 Use Command in Endpoint

```csharp
// Alternative endpoint using Command Bus

public class Endpoint : Endpoint<Request, Response>
{
    public override async Task HandleAsync(Request req, CancellationToken ct)
    {
        var command = new CreateProductCommand
        {
            Title = req.Title,
            Handle = req.Handle,
            Description = req.Description
        };

        // Execute command
        var result = await command.ExecuteAsync(ct);

        await SendAsync(new Response
        {
            Id = result.Id,
            Title = result.Title
        }, cancellation: ct);
    }
}
```

---

## Step 6: Rate Limiting (1 hour)

### 6.1 Configure Rate Limiting

```csharp
// Add to Program.cs

using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    // Global rate limit: 100 requests per minute
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        var userId = context.User.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString() ?? "anonymous";

        return RateLimitPartition.GetFixedWindowLimiter(userId, _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 100,
            Window = TimeSpan.FromMinutes(1),
            QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
            QueueLimit = 10
        });
    });

    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "Too many requests. Please try again later.",
            retryAfter = context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter)
                ? retryAfter.TotalSeconds
                : null
        }, cancellationToken: ct);
    };
});

// In middleware pipeline
app.UseRateLimiter();
```

### 6.2 Apply to Endpoints

```csharp
public override void Configure()
{
    Post("/products");

    // Apply rate limiting
    Options(o => o.RequireRateLimiting("fixed"));
}
```

---

## Step 7: Redis Cache Invalidation (1-2 hours)

### 7.1 Cache Invalidation Service

```csharp
// src/Medusa.API/Services/CacheInvalidationService.cs

using Microsoft.Extensions.Caching.Distributed;

namespace Medusa.API.Services;

public interface ICacheInvalidationService
{
    Task InvalidateAsync(string tag, CancellationToken ct = default);
    Task InvalidatePatternAsync(string pattern, CancellationToken ct = default);
}

public class CacheInvalidationService : ICacheInvalidationService
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<CacheInvalidationService> _logger;

    public CacheInvalidationService(
        IDistributedCache cache,
        ILogger<CacheInvalidationService> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task InvalidateAsync(string tag, CancellationToken ct = default)
    {
        _logger.LogInformation("Invalidating cache for tag: {Tag}", tag);

        // FastEndpoints output cache uses tags
        // This requires access to the output cache store
        // For now, we'll use a simple approach with distributed cache
        await _cache.RemoveAsync($"medusa:cache:tag:{tag}", ct);
    }

    public async Task InvalidatePatternAsync(string pattern, CancellationToken ct = default)
    {
        _logger.LogInformation("Invalidating cache for pattern: {Pattern}", pattern);

        // Redis-specific: use SCAN to find keys matching pattern
        // This requires StackExchange.Redis directly
        // Implementation depends on your caching setup
    }
}
```

### 7.2 Invalidate Cache on Events

```csharp
// src/Medusa.API/Subscribers/ProductCacheInvalidationSubscriber.cs

using Medusa.Core.EventBus;
using Medusa.API.Services;

namespace Medusa.API.Subscribers;

public class ProductCacheInvalidationSubscriber : BackgroundService
{
    private readonly IEventBus _eventBus;
    private readonly IServiceProvider _serviceProvider;

    public ProductCacheInvalidationSubscriber(
        IEventBus eventBus,
        IServiceProvider serviceProvider)
    {
        _eventBus = eventBus;
        _serviceProvider = serviceProvider;
    }

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _eventBus.Subscribe<ProductCreatedEvent>("product.created", async evt =>
        {
            using var scope = _serviceProvider.CreateScope();
            var cacheService = scope.ServiceProvider.GetRequiredService<ICacheInvalidationService>();

            // Invalidate product cache
            await cacheService.InvalidateAsync("products");
        });

        _eventBus.Subscribe<ProductUpdatedEvent>("product.updated", async evt =>
        {
            using var scope = _serviceProvider.CreateScope();
            var cacheService = scope.ServiceProvider.GetRequiredService<ICacheInvalidationService>();

            await cacheService.InvalidateAsync("products");
            await cacheService.InvalidateAsync($"product:{evt.Data.Id}");
        });

        return Task.CompletedTask;
    }
}

public class ProductCreatedEvent
{
    public string Id { get; set; } = string.Empty;
}

public class ProductUpdatedEvent
{
    public string Id { get; set; } = string.Empty;
}
```

---

## Step 8: Testing (2-3 hours)

### 8.1 Integration Test

```csharp
// tests/Medusa.IntegrationTests/ProductEndpointTests.cs

using System.Net;
using System.Net.Http.Json;
using FastEndpoints.Testing;
using FluentAssertions;
using Medusa.API.Features.Products.Create;

namespace Medusa.IntegrationTests;

public class ProductEndpointTests : TestFixture<Program>
{
    [Fact]
    public async Task Should_Create_Product_Successfully()
    {
        // Arrange
        var request = new Request
        {
            Title = "Test Product",
            Handle = "test-product",
            Variants = new List<VariantInput>
            {
                new() { Title = "Default", Sku = "TEST-001", Inventory = 100 }
            }
        };

        // Act
        var (response, result) = await Client.POSTAsync<
            Features.Products.Create.Endpoint,
            Request,
            Response>(request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        result.Should().NotBeNull();
        result.Id.Should().NotBeNullOrEmpty();
        result.Title.Should().Be("Test Product");
    }

    [Fact]
    public async Task Should_Return_ValidationError_When_Title_Empty()
    {
        // Arrange
        var request = new Request
        {
            Title = "",
            Handle = "test"
        };

        // Act
        var (response, _) = await Client.POSTAsync<
            Features.Products.Create.Endpoint,
            Request,
            ErrorResponse>(request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task Should_Respect_RateLimit()
    {
        // Send 101 requests (limit is 100)
        var tasks = Enumerable.Range(0, 101).Select(_ =>
            Client.POSTAsync<Features.Products.Create.Endpoint, Request, Response>(
                new Request { Title = "Test", Handle = "test" }));

        var responses = await Task.WhenAll(tasks);

        // At least one should be rate limited
        responses.Should().Contain(r => r.Response.StatusCode == HttpStatusCode.TooManyRequests);
    }
}
```

---

## Step 9: Verify Everything Works (1 hour)

### 9.1 Start Redis

```bash
docker run -d -p 6379:6379 redis:7-alpine
```

### 9.2 Run API

```bash
dotnet run --project src/Medusa.API
```

### 9.3 Test Endpoints

```bash
# Health check
curl http://localhost:5000/api/health

# Create product
curl -X POST http://localhost:5000/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Test Product",
    "handle": "test-product",
    "variants": [
      {
        "title": "Default",
        "sku": "TEST-001",
        "inventory": 100
      }
    ]
  }'

# Swagger UI
open http://localhost:5000/swagger
```

### 9.4 Verify Redis Cache

```bash
# Connect to Redis
docker exec -it <container-id> redis-cli

# Check cached keys
KEYS medusa:*

# Check output cache
GET medusa:cache:products
```

---

## Success Criteria

- ✅ FastEndpoints serving endpoints
- ✅ Redis output caching working
- ✅ Event grouping in pre/post processors
- ✅ Command Bus executing commands
- ✅ Rate limiting enforced
- ✅ Validation working
- ✅ Swagger UI accessible
- ✅ Integration tests passing

---

## Common Issues

### Issue: Redis connection failed
```bash
# Make sure Redis is running
docker ps | grep redis

# Check connection string
dotnet user-secrets set "ConnectionStrings:Redis" "localhost:6379"
```

### Issue: Events not being released
Check that `EventGroupPostProcessor` is registered and `Context` is scoped:
```csharp
builder.Services.AddScoped<Context>();
```

### Issue: Cache not invalidating
Make sure cache tags match between endpoints and invalidation:
```csharp
Options(o => o.CacheOutput(p => p.Tag("products")));
await cacheService.InvalidateAsync("products");
```

---

## Next Steps

Once Phase 5 is complete:
1. Test all endpoints with Postman/Swagger
2. Verify Redis caching with `redis-cli MONITOR`
3. Check event grouping with logging
4. Move to [Phase 6: Remote Query](./PHASE_6_REMOTE_QUERY.md)

---

## Estimated Time

- FastEndpoints setup: 1 hour
- Pre/Post processors: 2-3 hours
- Example endpoints: 3-4 hours
- Command Bus: 2-3 hours
- Rate limiting: 1 hour
- Cache invalidation: 1-2 hours
- Testing: 2-3 hours

**Total: 12-17 hours (2-3 weeks part-time)**
