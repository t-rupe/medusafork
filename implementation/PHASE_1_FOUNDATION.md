# Phase 1: Foundation (1-2 weeks)

## Goal
Build the core abstractions that everything else depends on: DI Container, Core Types, Context pattern, and base service generation.

## Prerequisites
- .NET 8 SDK installed
- Visual Studio 2022 or Rider
- Basic understanding of dependency injection

## Deliverables
- ✅ Autofac-based DI container with `registerAdd()` pattern
- ✅ Core types: `Context`, `FindOptions`, `FilterQuery`, `Event`
- ✅ Repository interface
- ✅ Base service factory (optional: auto-CRUD generation)

---

## Step 1: Create Solution Structure (30 min)

### 1.1 Create Solution

```bash
mkdir Medusa.NET
cd Medusa.NET

# Create solution
dotnet new sln -n Medusa

# Create core projects
dotnet new classlib -n Medusa.Core.Types -o src/Medusa.Core/Medusa.Core.Types
dotnet new classlib -n Medusa.Core.DependencyInjection -o src/Medusa.Core/Medusa.Core.DependencyInjection

# Create test project
dotnet new xunit -n Medusa.Core.Tests -o tests/Medusa.Core.Tests

# Add to solution
dotnet sln add src/Medusa.Core/Medusa.Core.Types
dotnet sln add src/Medusa.Core/Medusa.Core.DependencyInjection
dotnet sln add tests/Medusa.Core.Tests
```

### 1.2 Install Dependencies

```bash
# Core.DependencyInjection
cd src/Medusa.Core/Medusa.Core.DependencyInjection
dotnet add package Autofac --version 8.0.0

# Tests
cd ../../../tests/Medusa.Core.Tests
dotnet add package FluentAssertions --version 6.12.0
dotnet add package NSubstitute --version 5.1.0

# Add project references
dotnet add reference ../../src/Medusa.Core/Medusa.Core.Types
dotnet add reference ../../src/Medusa.Core/Medusa.Core.DependencyInjection
```

---

## Step 2: Core Types (2-3 hours)

### 2.1 Context

```csharp
// src/Medusa.Core/Medusa.Core.Types/Context.cs

namespace Medusa.Core.Types;

/// <summary>
/// Execution context passed through all service methods.
/// Used for transaction management, event grouping, and custom metadata.
/// </summary>
public class Context : Dictionary<string, object?>
{
    /// <summary>
    /// Groups events to be released atomically (for workflows).
    /// If set, events are staged and only emitted when explicitly released.
    /// </summary>
    public string? EventGroupId { get; set; }

    /// <summary>
    /// ORM transaction manager (e.g., DbContext transaction)
    /// </summary>
    public object? TransactionManager { get; set; }

    /// <summary>
    /// ORM manager/session (e.g., DbContext)
    /// </summary>
    public object? Manager { get; set; }

    /// <summary>
    /// Include soft-deleted records in queries
    /// </summary>
    public bool WithDeleted { get; set; }

    /// <summary>
    /// User ID initiating this operation (for audit trails)
    /// </summary>
    public string? UserId
    {
        get => this.TryGetValue("userId", out var value) ? value as string : null;
        set => this["userId"] = value;
    }

    /// <summary>
    /// Tenant/organization ID (for multi-tenancy)
    /// </summary>
    public string? TenantId
    {
        get => this.TryGetValue("tenantId", out var value) ? value as string : null;
        set => this["tenantId"] = value;
    }
}
```

### 2.2 Query Types

```csharp
// src/Medusa.Core/Medusa.Core.Types/FindOptions.cs

namespace Medusa.Core.Types;

public class FindOptions<T>
{
    public FilterQuery<T>? Where { get; set; }
    public string[]? Select { get; set; }
    public string[]? Relations { get; set; }
    public int? Take { get; set; }
    public int? Skip { get; set; }
    public Dictionary<string, SortDirection>? Sort { get; set; }
}

public enum SortDirection
{
    ASC,
    DESC
}
```

```csharp
// src/Medusa.Core/Medusa.Core.Types/FilterQuery.cs

namespace Medusa.Core.Types;

public class FilterQuery<T>
{
    public Dictionary<string, object>? Fields { get; set; }
    public FilterQuery<T>[]? And { get; set; }
    public FilterQuery<T>[]? Or { get; set; }
    public FilterQuery<T>? Not { get; set; }

    // Builder pattern for fluent API
    public static FilterQuery<T> Create() => new();

    public FilterQuery<T> Where(string field, object value)
    {
        Fields ??= new();
        Fields[field] = value;
        return this;
    }

    public FilterQuery<T> AndWhere(FilterQuery<T> query)
    {
        And = And == null ? new[] { query } : And.Append(query).ToArray();
        return this;
    }
}
```

### 2.3 Event Types

```csharp
// src/Medusa.Core/Medusa.Core.Types/Event.cs

namespace Medusa.Core.Types;

public class Event<T>
{
    public string Name { get; set; } = string.Empty;
    public EventMetadata? Metadata { get; set; }
    public T Data { get; set; } = default!;
}

public class EventMetadata
{
    /// <summary>
    /// Group events for atomic emission (workflow pattern)
    /// </summary>
    public string? EventGroupId { get; set; }

    public Dictionary<string, object>? Additional { get; set; }
}
```

### 2.4 Repository Interface

```csharp
// src/Medusa.Core/Medusa.Core.Types/IRepository.cs

namespace Medusa.Core.Types;

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
```

---

## Step 3: DI Container (3-4 hours)

### 3.1 Service Container Interface

```csharp
// src/Medusa.Core/Medusa.Core.DependencyInjection/IServiceContainer.cs

namespace Medusa.Core.DependencyInjection;

public interface IServiceContainer
{
    /// <summary>
    /// Resolve a service by type
    /// </summary>
    T Resolve<T>(string? name = null) where T : notnull;

    /// <summary>
    /// Resolve a service by type (non-generic)
    /// </summary>
    object Resolve(Type type, string? name = null);

    /// <summary>
    /// Register a service with a factory
    /// </summary>
    void Register<T>(string name, Func<IServiceContainer, T> factory) where T : notnull;

    /// <summary>
    /// Register a service instance
    /// </summary>
    void Register<T>(string name, T instance) where T : notnull;

    /// <summary>
    /// Add to a collection (like Medusa's registerAdd pattern)
    /// Multiple services can be registered with the same name.
    /// Use ResolveAll to get all instances.
    /// </summary>
    void RegisterAdd<T>(string name, Func<IServiceContainer, T> factory) where T : notnull;

    /// <summary>
    /// Resolve all services registered with a name
    /// </summary>
    IEnumerable<T> ResolveAll<T>(string name) where T : notnull;

    /// <summary>
    /// Create a new scope (for per-request services)
    /// </summary>
    IServiceContainer CreateScope();

    /// <summary>
    /// Check if a service is registered
    /// </summary>
    bool IsRegistered<T>(string? name = null) where T : notnull;

    /// <summary>
    /// Dispose resources (for scopes)
    /// </summary>
    void Dispose();
}
```

### 3.2 Autofac Implementation

```csharp
// src/Medusa.Core/Medusa.Core.DependencyInjection/AutofacServiceContainer.cs

using Autofac;
using Autofac.Core;

namespace Medusa.Core.DependencyInjection;

public class AutofacServiceContainer : IServiceContainer
{
    private readonly ILifetimeScope _scope;
    private readonly ContainerBuilder? _builder;
    private bool _isBuilt;

    // For building phase
    private AutofacServiceContainer(ContainerBuilder builder)
    {
        _builder = builder;
        _scope = null!;
        _isBuilt = false;
    }

    // For runtime phase
    private AutofacServiceContainer(ILifetimeScope scope)
    {
        _scope = scope;
        _builder = null;
        _isBuilt = true;
    }

    public static AutofacServiceContainer Create()
    {
        var builder = new ContainerBuilder();
        return new AutofacServiceContainer(builder);
    }

    public IServiceContainer Build()
    {
        if (_isBuilt)
            throw new InvalidOperationException("Container already built");

        var container = _builder!.Build();
        return new AutofacServiceContainer(container);
    }

    public T Resolve<T>(string? name = null) where T : notnull
    {
        EnsureBuilt();

        if (name != null)
            return _scope.ResolveNamed<T>(name);

        return _scope.Resolve<T>();
    }

    public object Resolve(Type type, string? name = null)
    {
        EnsureBuilt();

        if (name != null)
            return _scope.ResolveNamed(name, type);

        return _scope.Resolve(type);
    }

    public void Register<T>(string name, Func<IServiceContainer, T> factory) where T : notnull
    {
        EnsureNotBuilt();

        _builder!.Register(ctx => factory(new AutofacServiceContainer(ctx.Resolve<ILifetimeScope>())))
            .Named<T>(name)
            .InstancePerLifetimeScope();
    }

    public void Register<T>(string name, T instance) where T : notnull
    {
        EnsureNotBuilt();

        _builder!.RegisterInstance(instance)
            .Named<T>(name)
            .SingleInstance();
    }

    public void RegisterAdd<T>(string name, Func<IServiceContainer, T> factory) where T : notnull
    {
        EnsureNotBuilt();

        // Autofac automatically collects services with the same name when using IEnumerable
        _builder!.Register(ctx => factory(new AutofacServiceContainer(ctx.Resolve<ILifetimeScope>())))
            .Named<T>(name)
            .InstancePerLifetimeScope();
    }

    public IEnumerable<T> ResolveAll<T>(string name) where T : notnull
    {
        EnsureBuilt();

        return _scope.ResolveNamed<IEnumerable<T>>(name);
    }

    public IServiceContainer CreateScope()
    {
        EnsureBuilt();

        return new AutofacServiceContainer(_scope.BeginLifetimeScope());
    }

    public bool IsRegistered<T>(string? name = null) where T : notnull
    {
        EnsureBuilt();

        if (name != null)
            return _scope.IsRegisteredWithName<T>(name);

        return _scope.IsRegistered<T>();
    }

    public void Dispose()
    {
        _scope?.Dispose();
    }

    private void EnsureBuilt()
    {
        if (!_isBuilt)
            throw new InvalidOperationException("Container must be built before resolving services");
    }

    private void EnsureNotBuilt()
    {
        if (_isBuilt)
            throw new InvalidOperationException("Cannot register services after container is built");
    }
}
```

---

## Step 4: Tests (2-3 hours)

### 4.1 Context Tests

```csharp
// tests/Medusa.Core.Tests/ContextTests.cs

using FluentAssertions;
using Medusa.Core.Types;

namespace Medusa.Core.Tests;

public class ContextTests
{
    [Fact]
    public void Should_Store_EventGroupId()
    {
        var context = new Context
        {
            EventGroupId = "test-group-id"
        };

        context.EventGroupId.Should().Be("test-group-id");
    }

    [Fact]
    public void Should_Store_Custom_Properties()
    {
        var context = new Context
        {
            UserId = "user-123",
            TenantId = "tenant-456"
        };

        context.UserId.Should().Be("user-123");
        context.TenantId.Should().Be("tenant-456");
    }

    [Fact]
    public void Should_Store_Dictionary_Values()
    {
        var context = new Context
        {
            ["custom-key"] = "custom-value"
        };

        context["custom-key"].Should().Be("custom-value");
    }
}
```

### 4.2 DI Container Tests

```csharp
// tests/Medusa.Core.Tests/ServiceContainerTests.cs

using FluentAssertions;
using Medusa.Core.DependencyInjection;

namespace Medusa.Core.Tests;

public class ServiceContainerTests
{
    [Fact]
    public void Should_Resolve_Registered_Service()
    {
        var container = AutofacServiceContainer.Create();
        container.Register("testService", _ => new TestService());
        var built = container.Build();

        var service = built.Resolve<TestService>("testService");

        service.Should().NotBeNull();
        service.Should().BeOfType<TestService>();
    }

    [Fact]
    public void Should_Resolve_All_Services_With_RegisterAdd()
    {
        var container = AutofacServiceContainer.Create();
        container.RegisterAdd("subscribers", _ => new TestSubscriber1());
        container.RegisterAdd("subscribers", _ => new TestSubscriber2());
        var built = container.Build();

        var subscribers = built.ResolveAll<ITestSubscriber>("subscribers").ToList();

        subscribers.Should().HaveCount(2);
        subscribers.Should().ContainItemsAssignableTo<ITestSubscriber>();
    }

    [Fact]
    public void Should_Create_Scope()
    {
        var container = AutofacServiceContainer.Create();
        container.Register("testService", _ => new TestService());
        var built = container.Build();

        using var scope = built.CreateScope();
        var service = scope.Resolve<TestService>("testService");

        service.Should().NotBeNull();
    }

    private class TestService { }

    private interface ITestSubscriber { }
    private class TestSubscriber1 : ITestSubscriber { }
    private class TestSubscriber2 : ITestSubscriber { }
}
```

---

## Step 5: Verify Everything Works (30 min)

### Run Tests

```bash
dotnet test

# Expected output:
# Passed!  - Failed:     0, Passed:     5, Skipped:     0, Total:     5
```

### Create Example Usage

```csharp
// Example.cs (temporary file to verify)

using Medusa.Core.DependencyInjection;
using Medusa.Core.Types;

var container = AutofacServiceContainer.Create();

// Register services
container.Register("logger", _ => new ConsoleLogger());
container.RegisterAdd("subscriber", _ => new EmailSubscriber());
container.RegisterAdd("subscriber", _ => new SmsSubscriber());

var built = container.Build();

// Resolve services
var logger = built.Resolve<ConsoleLogger>("logger");
var subscribers = built.ResolveAll<ISubscriber>("subscriber");

Console.WriteLine($"Resolved {subscribers.Count()} subscribers");

// Test context
var context = new Context
{
    EventGroupId = "wf_123",
    UserId = "user_456"
};

Console.WriteLine($"Context EventGroupId: {context.EventGroupId}");
Console.WriteLine($"Context UserId: {context.UserId}");
```

---

## Success Criteria

- ✅ All tests pass
- ✅ Can create and resolve services from container
- ✅ `registerAdd` pattern works (multiple services with same key)
- ✅ Context can store metadata
- ✅ Repository interface defined
- ✅ FilterQuery and FindOptions work

---

## Common Issues

### Issue: Autofac not found
```bash
dotnet add package Autofac --version 8.0.0
```

### Issue: Tests not finding types
Make sure project references are correct:
```bash
dotnet add tests/Medusa.Core.Tests reference src/Medusa.Core/Medusa.Core.Types
```

### Issue: Container already built
You're trying to register after calling `Build()`. Register all services before building.

---

## Next Steps

Once Phase 1 is complete:
1. Commit your code: `git add . && git commit -m "Phase 1: Foundation complete"`
2. Move to [Phase 2: Module System](./PHASE_2_MODULE_SYSTEM.md)

---

## Estimated Time

- Solution setup: 30 min
- Core types: 2-3 hours
- DI Container: 3-4 hours
- Tests: 2-3 hours
- Verification: 30 min

**Total: 8-11 hours (1-2 weeks part-time)**
