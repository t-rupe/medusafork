# Dependency Injection Container Analysis - MedusaJS

## Purpose
Awilix-based dependency injection container providing service registration, module loading, request scoping, and typed dependency resolution.

## Architecture

### Core Components

**1. Container Creation** (`utils/src/common/medusa-container.ts`)
Extends Awilix container with custom functionality:

```typescript
const container = createMedusaContainer()

// Extended features:
container.registerAdd("subscribers", subscriber1)  // Adds to collection
container.registerAdd("subscribers", subscriber2)  // Adds to same collection
// Resolves to array: [subscriber1, subscriber2]

const scoped = container.createScope()  // Request-scoped container
```

**Key Patterns:**
- **registerAdd**: Accumulator registration for arrays (subscribers, middleware, etc.)
- **createScope**: Fork container for request isolation
- Type-safe resolution via `MedusaContainer<Cradle>` interface

**2. Registration Keys** (`common/container.ts`)
Centralized key constants:
```typescript
ContainerRegistrationKeys = {
  PG_CONNECTION: "__pg_connection__",
  MANAGER: "manager",
  CONFIG_MODULE: "configModule",
  LOGGER: "logger",
  REMOTE_QUERY: "remoteQuery",
  QUERY: "query",
  LINK: "link",
  FEATURE_FLAG_ROUTER: "featureFlagRouter"
}
```

Used for core framework services resolution.

**3. Module Container Type** (`framework/src/types/container.ts`)
Type augmentation for all registered modules:

```typescript
declare module "@medusajs/types" {
  export interface ModuleImplementations {
    [ContainerRegistrationKeys.CONFIG_MODULE]: ConfigModule
    [Modules.PRODUCT]: IProductModuleService
    [Modules.CART]: ICartModuleService
    // ... all domain modules
  }
}

type MedusaContainer<Cradle = ModuleImplementations> = {
  resolve<K extends keyof Cradle>(key: K): Cradle[K]
  // Provides IntelliSense for all registered services
}
```

**4. Module Container Loader** (`modules-sdk/loaders/container-loader-factory.ts`)
Auto-registers repositories and services:

```typescript
moduleContainerLoaderFactory({
  moduleModels: { Product, Variant },
  moduleServices: { ProductService },
  moduleRepositories: { ProductRepository }
})
// Auto-creates:
// - productService (custom or default)
// - variantService (default MedusaInternalService)
// - productRepository (custom or default mikroOrmBaseRepositoryFactory)
// - variantRepository (default)
// - baseRepository (global)
```

**Process:**
1. Load custom services or generate default `MedusaInternalService(Model)`
2. Load custom repositories or generate `mikroOrmBaseRepositoryFactory(Model)`
3. Register all as singletons with `asClass(...).singleton()`
4. Names: `productRepository`, `productService` (lowerCaseFirst convention)

**5. Registration Modes** (Awilix)
- **asValue**: Register instance directly
- **asClass**: Register class, instantiated on resolve
- **asFunction**: Register factory function
- **Singleton**: `.singleton()` - one instance per container
- **Scoped**: `.scoped()` - one instance per scope
- **Transient**: `.transient()` - new instance each resolve

```typescript
container.register({
  logger: asValue(winstonLogger),                    // Instance
  productService: asClass(ProductService).singleton(), // Singleton class
  emailSender: asFunction(createEmailSender).scoped()  // Factory, scoped
})
```

**6. Request Scoping**
HTTP middleware creates scoped container:

```typescript
// HTTP middleware
app.use((req, res, next) => {
  req.scope = container.createScope()
  req.scope.register({
    manager: asValue(req.em.fork()),  // Request-specific EntityManager
    currentUser: asValue(req.user)
  })
  next()
})

// In route handler
const productService = req.scope.resolve("productService")
// Uses request's EntityManager via constructor injection
```

**7. Constructor Injection**
Services auto-inject dependencies:

```typescript
class ProductService {
  constructor({
    productRepository,        // Auto-injected
    eventBusService,          // Auto-injected
    logger,                   // Auto-injected
    manager                   // Request-scoped EM
  }: {
    productRepository: ProductRepository
    eventBusService: IEventBusService
    logger: Logger
    manager: EntityManager
  }) {
    this.repository = productRepository
    this.eventBus = eventBusService
  }
}

container.register({
  productService: asClass(ProductService).singleton()
})
```

Awilix introspects constructor parameters and injects matched registrations.

**8. Module Service Pattern**
Default service generated if none provided:

```typescript
MedusaInternalService(Model) {
  return class extends AbstractModuleService {
    constructor({ manager }) {
      super({ [lowerCaseFirst(Model.name) + "Repository"]: ... })
    }
  }
}
```

Provides standard CRUD methods:
- `retrieve(id, config)`
- `list(filters, config)`
- `create(data)`
- `update(id, data)`
- `delete(id)`
- `softDelete(id)`

**9. Cradle Access**
Container.cradle provides direct access:
```typescript
const { productService, logger } = container.cradle
// Equivalent to:
const productService = container.resolve("productService")
const logger = container.resolve("logger")
```

**10. Global vs Scoped Registration**
```typescript
// Global container (application startup)
container.register({
  logger: asValue(logger),           // Singleton, shared
  configModule: asValue(config)      // Singleton, shared
})

// Scoped container (per-request)
requestScope.register({
  manager: asValue(em.fork()),       // Request-specific
  currentUser: asValue(req.user)     // Request-specific
})
```

## Service Resolution Flow

**1. Application Startup:**
```
createMedusaContainer()
→ Register core services (logger, config, pg_connection)
→ Load modules (moduleContainerLoaderFactory)
  → Register module repositories
  → Register module services
→ Container ready
```

**2. Per-Request:**
```
HTTP Request
→ Middleware: req.scope = container.createScope()
→ Register request-specific: manager, currentUser
→ Route handler: req.scope.resolve("productService")
→ ProductService constructor injects: productRepository, manager
→ Repository uses request's manager
```

## .NET Mapping with Built-in DI

### 1. Replace Awilix with ASP.NET Core DI

**Medusa Awilix:**
```typescript
container.register({
  productService: asClass(ProductService).singleton(),
  productRepository: asClass(ProductRepository).scoped()
})

const service = container.resolve("productService")
```

**.NET Built-in:**
```csharp
// Startup.cs / Program.cs
services.AddSingleton<IProductService, ProductService>();
services.AddScoped<IProductRepository, ProductRepository>();

// Resolution
public class ProductController
{
    private readonly IProductService _productService;

    public ProductController(IProductService productService)
    {
        _productService = productService;
    }
}
```

### 2. Registration Modes Mapping

| Medusa (Awilix) | .NET | Use Case |
|-----------------|------|----------|
| `asClass(...).singleton()` | `AddSingleton<>()` | Logger, config |
| `asClass(...).scoped()` | `AddScoped<>()` | DbContext, repositories |
| `asClass(...).transient()` | `AddTransient<>()` | Stateless services |
| `asValue(instance)` | `AddSingleton(instance)` | Pre-created instances |
| `asFunction(factory)` | `AddScoped(sp => factory(sp))` | Factory pattern |

### 3. Request Scoping

**Medusa:**
```typescript
req.scope = container.createScope()
req.scope.register({ manager: asValue(em.fork()) })
```

**.NET:**
```csharp
// Automatic per-request scope in ASP.NET Core
// Scoped services auto-created per request
public void ConfigureServices(IServiceCollection services)
{
    services.AddScoped<MedusaDbContext>(); // Auto-scoped to request
}
```

### 4. Constructor Injection

**Medusa:**
```typescript
class ProductService {
  constructor({ productRepository, logger }) { }
}
```

**.NET:**
```csharp
public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        IProductRepository repository,
        ILogger<ProductService> logger)
    {
        _repository = repository;
        _logger = logger;
    }
}
```

### 5. Module Auto-Registration

**Medusa:** `moduleContainerLoaderFactory` auto-registers models/services
**.NET:** Use Scrutor or custom extension

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddModuleServices(
        this IServiceCollection services,
        Assembly assembly)
    {
        // Auto-register all services ending with "Service"
        services.Scan(scan => scan
            .FromAssemblies(assembly)
            .AddClasses(classes => classes.AssignableTo<IModuleService>())
            .AsImplementedInterfaces()
            .WithScopedLifetime());

        return services;
    }
}

// Usage
services.AddModuleServices(typeof(ProductService).Assembly);
```

### 6. registerAdd Pattern (Collection Registration)

**Medusa:**
```typescript
container.registerAdd("subscribers", subscriber1)
container.registerAdd("subscribers", subscriber2)
// Resolves to: [subscriber1, subscriber2]
```

**.NET:**
```csharp
// Register multiple implementations
services.AddSingleton<ISubscriber, EmailSubscriber>();
services.AddSingleton<ISubscriber, SmsSubscriber>();

// Resolve collection
public class EventBus
{
    public EventBus(IEnumerable<ISubscriber> subscribers)
    {
        // Gets all registered ISubscriber instances
    }
}
```

### 7. Named Registration

**Medusa:** Uses string keys like `"productService"`
**.NET:** Uses types + optional Keyed Services (C# 12)

```csharp
// .NET 8+ Keyed Services
services.AddKeyedSingleton<ICache, RedisCache>("redis");
services.AddKeyedSingleton<ICache, MemoryCache>("memory");

// Resolution
public class Service
{
    public Service([FromKeyedServices("redis")] ICache cache) { }
}

// Pre-.NET 8: Use factory pattern
services.AddSingleton<Func<string, ICache>>(sp => key =>
{
    return key switch
    {
        "redis" => sp.GetRequiredService<RedisCache>(),
        "memory" => sp.GetRequiredService<MemoryCache>(),
        _ => throw new ArgumentException($"Unknown cache: {key}")
    };
});
```

### 8. Cradle Access Pattern

**Medusa:**
```typescript
const { productService, logger } = container.cradle
```

**.NET:**
```csharp
// Service locator (anti-pattern, avoid if possible)
public class MyClass
{
    private readonly IServiceProvider _serviceProvider;

    public MyClass(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public void DoWork()
    {
        var service = _serviceProvider.GetRequiredService<IProductService>();
    }
}

// Prefer: Direct injection in constructor
```

### 9. Default Service Generation

**Medusa:** `MedusaInternalService(Model)` generates CRUD service
**.NET:** Generic repository or base service class

```csharp
public class GenericService<TEntity> : IService<TEntity>
    where TEntity : class
{
    protected readonly IRepository<TEntity> _repository;

    public GenericService(IRepository<TEntity> repository)
    {
        _repository = repository;
    }

    public virtual Task<TEntity?> Retrieve(string id, CancellationToken ct)
        => _repository.FindById(id, ct);

    public virtual Task<List<TEntity>> List(
        Expression<Func<TEntity, bool>> filter,
        CancellationToken ct)
        => _repository.Find(filter, ct);
}

// Register
services.AddScoped(typeof(IService<>), typeof(GenericService<>));
```

### 10. Module Registration Pattern

**Medusa Module:**
```typescript
export default Module("product", {
  service: ProductService,
  loaders: [containerLoader({ moduleModels, moduleServices })]
})
```

**.NET Module:**
```csharp
public static class ProductModule
{
    public static IServiceCollection AddProductModule(
        this IServiceCollection services)
    {
        // Register services
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<IProductRepository, ProductRepository>();

        // Auto-register from assembly
        services.AddModuleServices(typeof(ProductModule).Assembly);

        return services;
    }
}

// In Program.cs
builder.Services.AddProductModule();
```

## Key Differences

| Aspect | MedusaJS (Awilix) | .NET Core DI |
|--------|-------------------|--------------|
| Registration | String keys | Type-based |
| Resolution | `resolve("key")` | Constructor injection |
| Scoping | Manual `createScope()` | Automatic per-request |
| Collections | `registerAdd` | `Add*` multiple times |
| Parameter Injection | Object destructuring | Constructor params |
| Named Services | Native | Keyed Services (.NET 8+) |
| Service Locator | `cradle` property | `IServiceProvider` |

## Implementation Recommendations

### Project Structure
```
/DependencyInjection
  ServiceCollectionExtensions.cs
  /Modules
    ProductModuleExtensions.cs
    CartModuleExtensions.cs
```

### Key NuGet Packages
- **Microsoft.Extensions.DependencyInjection** (built-in)
- **Scrutor** - Assembly scanning for auto-registration
- **Autofac** (optional) - If need Awilix-like features

### Service Registration Pattern
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Core services
builder.Services.AddSingleton<ILogger, SerilogLogger>();
builder.Services.AddScoped<MedusaDbContext>();

// Module services
builder.Services.AddProductModule();
builder.Services.AddCartModule();
builder.Services.AddOrderModule();

var app = builder.Build();
```

### Module Extension Pattern
```csharp
public static class ProductModuleExtensions
{
    public static IServiceCollection AddProductModule(
        this IServiceCollection services)
    {
        // Repositories
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IVariantRepository, VariantRepository>();

        // Services
        services.AddScoped<IProductService, ProductService>();

        // Auto-register remaining
        services.Scan(scan => scan
            .FromAssemblyOf<ProductService>()
            .AddClasses(c => c.InNamespaces("Medusa.Product.Services"))
            .AsImplementedInterfaces()
            .WithScopedLifetime());

        return services;
    }
}
```

### Request-Scoped Services (Like req.scope)
```csharp
// Already automatic in ASP.NET Core
public class ProductEndpoint : Endpoint<Request, Response>
{
    // DbContext is automatically scoped to request
    private readonly MedusaDbContext _context;

    public ProductEndpoint(MedusaDbContext context)
    {
        _context = context; // New instance per request
    }
}
```

### Factory Pattern for Dynamic Resolution
```csharp
public interface IServiceFactory
{
    T GetService<T>() where T : class;
}

public class ServiceFactory : IServiceFactory
{
    private readonly IServiceProvider _serviceProvider;

    public ServiceFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public T GetService<T>() where T : class
        => _serviceProvider.GetRequiredService<T>();
}

// Register
services.AddScoped<IServiceFactory, ServiceFactory>();
```

## References

**Core Files:**
- `utils/src/common/medusa-container.ts` - Container creation
- `utils/src/common/container.ts` - Registration keys
- `framework/src/types/container.ts` - Type augmentation
- `modules-sdk/loaders/container-loader-factory.ts` - Module loader
- `framework/src/container.ts` - Global container instance

**Awilix Documentation:**
- https://github.com/jeffijoe/awilix

**.NET DI Documentation:**
- https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection
