# Phase 3: Event Bus (1-2 weeks)

## Goal
Build event-driven communication system with **two implementations**: FastEndpoints EventBus (dev) and Redis (production), both supporting event grouping for workflows.

## Prerequisites
- ✅ Phase 1: Foundation complete
- ✅ Phase 2: Module System complete
- FastEndpoints installed
- Redis running (for production implementation)

## Deliverables
- ✅ `IEventBus` abstraction
- ✅ FastEndpoints EventBus adapter (in-process, dev)
- ✅ Redis EventBus implementation (distributed, production)
- ✅ Event grouping for workflows
- ✅ Event subscribers pattern

---

## Step 1: Event Bus Abstraction (1-2 hours)

### 1.1 Create Event Bus Projects

```bash
# Abstractions
dotnet new classlib -n Medusa.Core.EventBus.Abstractions -o src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Abstractions
dotnet sln add src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Abstractions

# FastEndpoints adapter
dotnet new classlib -n Medusa.Core.EventBus.FastEndpoints -o src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.FastEndpoints
dotnet sln add src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.FastEndpoints

# Redis implementation
dotnet new classlib -n Medusa.Core.EventBus.Redis -o src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Redis
dotnet sln add src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Redis

# Add references
cd src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Abstractions
dotnet add reference ../../Medusa.Core.Types

cd ../Medusa.Core.EventBus.FastEndpoints
dotnet add reference ../Medusa.Core.EventBus.Abstractions
dotnet add package FastEndpoints --version 5.25.0

cd ../Medusa.Core.EventBus.Redis
dotnet add reference ../Medusa.Core.EventBus.Abstractions
dotnet add package StackExchange.Redis --version 2.7.0
```

### 1.2 Event Bus Interface

```csharp
// src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Abstractions/IEventBus.cs

using Medusa.Core.Types;

namespace Medusa.Core.EventBus;

/// <summary>
/// Event bus abstraction for pub/sub messaging.
/// Supports event grouping for workflow compensation.
/// </summary>
public interface IEventBus
{
    /// <summary>
    /// Publish an event. If metadata contains EventGroupId, event is staged (not emitted immediately).
    /// </summary>
    Task PublishAsync<T>(
        string eventName,
        T data,
        EventMetadata? metadata = null,
        Context? context = null);

    /// <summary>
    /// Subscribe to an event
    /// </summary>
    void Subscribe<T>(string eventName, Func<Event<T>, Task> handler);

    /// <summary>
    /// Unsubscribe from an event
    /// </summary>
    void Unsubscribe<T>(string eventName, Func<Event<T>, Task> handler);

    /// <summary>
    /// Release all events in a group atomically (for successful workflows)
    /// </summary>
    Task ReleaseGroupedEventsAsync(string eventGroupId);

    /// <summary>
    /// Clear all events in a group without emitting (for failed workflows)
    /// </summary>
    Task ClearGroupedEventsAsync(string eventGroupId);
}
```

---

## Step 2: FastEndpoints Adapter (2-3 hours)

### 2.1 FastEndpoints Event Bus Adapter

```csharp
// src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.FastEndpoints/FastEndpointsEventBusAdapter.cs

using System.Collections.Concurrent;
using Medusa.Core.Types;
using Microsoft.Extensions.Logging;

namespace Medusa.Core.EventBus.FastEndpoints;

/// <summary>
/// Wraps FastEndpoints EventBus and adds event grouping support for workflows.
/// Use this for development/single-instance deployments.
/// </summary>
public class FastEndpointsEventBusAdapter : IEventBus
{
    private readonly global::FastEndpoints.IEventBus _fastEndpointsEventBus;
    private readonly ILogger<FastEndpointsEventBusAdapter> _logger;

    // Staged events (grouped by eventGroupId)
    private readonly ConcurrentDictionary<string, List<StagedEvent>> _stagedEvents = new();

    // Event handlers (since FastEndpoints manages its own subscriptions)
    private readonly ConcurrentDictionary<string, List<Delegate>> _handlers = new();

    public FastEndpointsEventBusAdapter(
        global::FastEndpoints.IEventBus fastEndpointsEventBus,
        ILogger<FastEndpointsEventBusAdapter> logger)
    {
        _fastEndpointsEventBus = fastEndpointsEventBus;
        _logger = logger;
    }

    public async Task PublishAsync<T>(
        string eventName,
        T data,
        EventMetadata? metadata = null,
        Context? context = null)
    {
        var evt = new Event<T>
        {
            Name = eventName,
            Data = data,
            Metadata = metadata
        };

        // If event has group ID, stage it (don't emit yet)
        if (metadata?.EventGroupId != null)
        {
            StageEvent(metadata.EventGroupId, evt);
            _logger.LogDebug("Staged event '{EventName}' in group '{GroupId}'", eventName, metadata.EventGroupId);
            return;
        }

        // Otherwise emit immediately
        await EmitEventAsync(evt);
    }

    public void Subscribe<T>(string eventName, Func<Event<T>, Task> handler)
    {
        // Store handler for later execution
        var handlers = _handlers.GetOrAdd(eventName, _ => new List<Delegate>());

        lock (handlers)
        {
            handlers.Add(handler);
        }

        _logger.LogInformation("Subscribed to event '{EventName}'", eventName);
    }

    public void Unsubscribe<T>(string eventName, Func<Event<T>, Task> handler)
    {
        if (_handlers.TryGetValue(eventName, out var handlers))
        {
            lock (handlers)
            {
                handlers.Remove(handler);
            }
        }
    }

    public async Task ReleaseGroupedEventsAsync(string eventGroupId)
    {
        if (!_stagedEvents.TryRemove(eventGroupId, out var events))
        {
            _logger.LogWarning("No staged events found for group '{GroupId}'", eventGroupId);
            return;
        }

        _logger.LogInformation("Releasing {Count} events from group '{GroupId}'", events.Count, eventGroupId);

        // Emit all staged events
        foreach (var stagedEvent in events)
        {
            await EmitEventCoreAsync(stagedEvent.EventName, stagedEvent.Event);
        }
    }

    public async Task ClearGroupedEventsAsync(string eventGroupId)
    {
        if (_stagedEvents.TryRemove(eventGroupId, out var events))
        {
            _logger.LogInformation("Cleared {Count} staged events from group '{GroupId}'", events.Count, eventGroupId);
        }

        await Task.CompletedTask;
    }

    private void StageEvent<T>(string groupId, Event<T> evt)
    {
        var staged = _stagedEvents.GetOrAdd(groupId, _ => new List<StagedEvent>());

        lock (staged)
        {
            staged.Add(new StagedEvent
            {
                EventName = evt.Name,
                Event = evt
            });
        }
    }

    private async Task EmitEventAsync<T>(Event<T> evt)
    {
        await EmitEventCoreAsync(evt.Name, evt);
    }

    private async Task EmitEventCoreAsync(string eventName, object evt)
    {
        _logger.LogDebug("Emitting event '{EventName}'", eventName);

        // Call our internal handlers
        if (_handlers.TryGetValue(eventName, out var handlers))
        {
            var handlersCopy = handlers.ToList();

            foreach (var handler in handlersCopy)
            {
                try
                {
                    // Invoke handler dynamically
                    var method = handler.GetType().GetMethod("Invoke");
                    var task = (Task)method!.Invoke(handler, new[] { evt })!;
                    await task;
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error executing handler for event '{EventName}'", eventName);
                }
            }
        }

        // Also emit to FastEndpoints EventBus (if you want both systems)
        // This depends on how you want to integrate with FastEndpoints
        // For now, we're using our own handler system
    }

    private class StagedEvent
    {
        public string EventName { get; set; } = string.Empty;
        public object Event { get; set; } = null!;
    }
}
```

---

## Step 3: Redis Implementation (3-4 hours)

### 3.1 Redis Event Bus

```csharp
// src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Redis/RedisEventBus.cs

using System.Collections.Concurrent;
using System.Text.Json;
using Medusa.Core.Types;
using Microsoft.Extensions.Logging;
using StackExchange.Redis;

namespace Medusa.Core.EventBus.Redis;

/// <summary>
/// Redis-backed event bus with persistence and horizontal scaling.
/// Use this for production deployments.
/// </summary>
public class RedisEventBus : IEventBus, IDisposable
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _db;
    private readonly ISubscriber _subscriber;
    private readonly ILogger<RedisEventBus> _logger;
    private readonly ConcurrentDictionary<string, List<Func<Event<object>, Task>>> _handlers = new();

    private const string EventChannelPrefix = "medusa:events:";
    private const string StagedEventsPrefix = "medusa:staged:";

    public RedisEventBus(
        string connectionString,
        ILogger<RedisEventBus> logger)
    {
        _redis = ConnectionMultiplexer.Connect(connectionString);
        _db = _redis.GetDatabase();
        _subscriber = _redis.GetSubscriber();
        _logger = logger;

        // Subscribe to all event channels
        _subscriber.Subscribe(new RedisChannel($"{EventChannelPrefix}*", RedisChannel.PatternMode.Pattern), async (channel, message) =>
        {
            await HandleRedisMessageAsync(channel!, message!);
        });

        _logger.LogInformation("Redis EventBus initialized");
    }

    public async Task PublishAsync<T>(
        string eventName,
        T data,
        EventMetadata? metadata = null,
        Context? context = null)
    {
        var evt = new Event<T>
        {
            Name = eventName,
            Data = data,
            Metadata = metadata
        };

        // If event has group ID, stage it in Redis list
        if (metadata?.EventGroupId != null)
        {
            await StageEventAsync(metadata.EventGroupId, evt);
            _logger.LogDebug("Staged event '{EventName}' in group '{GroupId}'", eventName, metadata.EventGroupId);
            return;
        }

        // Otherwise publish to Redis pub/sub
        await PublishToRedisAsync(evt);
    }

    public void Subscribe<T>(string eventName, Func<Event<T>, Task> handler)
    {
        var handlers = _handlers.GetOrAdd(eventName, _ => new List<Func<Event<object>, Task>>());

        // Wrap handler to convert Event<object> to Event<T>
        handlers.Add(async (evt) =>
        {
            var typedEvt = new Event<T>
            {
                Name = evt.Name,
                Data = JsonSerializer.Deserialize<T>(JsonSerializer.Serialize(evt.Data))!,
                Metadata = evt.Metadata
            };
            await handler(typedEvt);
        });

        _logger.LogInformation("Subscribed to event '{EventName}'", eventName);
    }

    public void Unsubscribe<T>(string eventName, Func<Event<T>, Task> handler)
    {
        if (_handlers.TryGetValue(eventName, out var handlers))
        {
            // Note: This is simplified. In production, you'd need to track the wrapper.
            _logger.LogInformation("Unsubscribed from event '{EventName}'", eventName);
        }
    }

    public async Task ReleaseGroupedEventsAsync(string eventGroupId)
    {
        var key = $"{StagedEventsPrefix}{eventGroupId}";

        // Get all staged events from Redis list
        var events = await _db.ListRangeAsync(key);

        if (events.Length == 0)
        {
            _logger.LogWarning("No staged events found for group '{GroupId}'", eventGroupId);
            return;
        }

        _logger.LogInformation("Releasing {Count} events from group '{GroupId}'", events.Length, eventGroupId);

        // Publish each event to Redis pub/sub
        foreach (var eventJson in events)
        {
            var evt = JsonSerializer.Deserialize<Event<object>>(eventJson!);
            if (evt != null)
            {
                await PublishToRedisAsync(evt);
            }
        }

        // Clear staged events
        await _db.KeyDeleteAsync(key);
    }

    public async Task ClearGroupedEventsAsync(string eventGroupId)
    {
        var key = $"{StagedEventsPrefix}{eventGroupId}";
        var deleted = await _db.KeyDeleteAsync(key);

        _logger.LogInformation("Cleared {Deleted} staged event group '{GroupId}'", deleted ? "1" : "0", eventGroupId);
    }

    private async Task StageEventAsync<T>(string groupId, Event<T> evt)
    {
        var key = $"{StagedEventsPrefix}{groupId}";
        var json = JsonSerializer.Serialize(evt);

        await _db.ListRightPushAsync(key, json);

        // Set TTL (10 minutes) to prevent stale data
        await _db.KeyExpireAsync(key, TimeSpan.FromMinutes(10));
    }

    private async Task PublishToRedisAsync<T>(Event<T> evt)
    {
        var channel = $"{EventChannelPrefix}{evt.Name}";
        var json = JsonSerializer.Serialize(evt);

        await _subscriber.PublishAsync(channel, json);

        _logger.LogDebug("Published event '{EventName}' to Redis", evt.Name);
    }

    private async Task HandleRedisMessageAsync(RedisChannel channel, RedisValue message)
    {
        try
        {
            var channelName = channel.ToString();
            var eventName = channelName.Replace(EventChannelPrefix, "");

            var evt = JsonSerializer.Deserialize<Event<object>>(message!);

            if (evt == null)
            {
                _logger.LogWarning("Failed to deserialize event from channel '{Channel}'", channelName);
                return;
            }

            // Execute handlers
            if (_handlers.TryGetValue(eventName, out var handlers))
            {
                await Task.WhenAll(handlers.Select(h => h(evt)));
            }

            // Also execute wildcard handlers (if any)
            if (_handlers.TryGetValue("*", out var wildcardHandlers))
            {
                await Task.WhenAll(wildcardHandlers.Select(h => h(evt)));
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling Redis message from channel '{Channel}'", channel);
        }
    }

    public void Dispose()
    {
        _redis?.Dispose();
    }
}
```

---

## Step 4: Event Bus Module (2 hours)

### 4.1 Create Event Bus Module

```csharp
// Create module project
dotnet new classlib -n Medusa.EventBus.Module -o src/Modules/Medusa.EventBus.Module
dotnet sln add src/Modules/Medusa.EventBus.Module

cd src/Modules/Medusa.EventBus.Module
dotnet add reference ../../Medusa.Core/Medusa.Core.Modules
dotnet add reference ../../Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Abstractions
dotnet add reference ../../Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.FastEndpoints
dotnet add reference ../../Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Redis
```

```csharp
// src/Modules/Medusa.EventBus.Module/EventBusModule.cs

using Medusa.Core.EventBus;
using Medusa.Core.EventBus.FastEndpoints;
using Medusa.Core.EventBus.Redis;
using Medusa.Core.Modules;
using Medusa.Core.Types;
using Microsoft.Extensions.Logging;

namespace Medusa.EventBus.Module;

public class EventBusModule : IModuleService
{
    private readonly IEventBus _eventBus;
    private readonly ILogger<EventBusModule> _logger;

    public EventBusModule(
        Dictionary<string, object> dependencies,
        ModuleDeclaration declaration)
    {
        // Get logger
        _logger = dependencies.TryGetValue("logger", out var logger)
            ? (ILogger<EventBusModule>)logger
            : throw new ArgumentException("Logger dependency not found");

        // Get event bus implementation based on configuration
        var eventBusType = declaration.Options?.GetValueOrDefault("eventBusType", "fastendpoints")?.ToString() ?? "fastendpoints";

        _eventBus = eventBusType.ToLower() switch
        {
            "redis" => CreateRedisEventBus(declaration, _logger),
            "fastendpoints" => CreateFastEndpointsEventBus(dependencies, _logger),
            _ => throw new ArgumentException($"Unknown event bus type: {eventBusType}")
        };

        _logger.LogInformation("EventBus module initialized with {Type}", eventBusType);
    }

    public ModuleDefinition Definition => new()
    {
        Key = "eventBus",
        Label = "Event Bus",
        DefaultPackage = "Medusa.EventBus.Module",
        IsRequired = true,
        IsQueryable = false,
        Dependencies = new[] { "logger" },
        DefaultOptions = new Dictionary<string, object>
        {
            { "eventBusType", "fastendpoints" }
        }
    };

    public ModuleJoinerConfig? JoinerConfig => null;

    public ModuleHooks? Hooks => new()
    {
        OnApplicationStart = async (sp) =>
        {
            _logger.LogInformation("EventBus module started");
            await Task.CompletedTask;
        }
    };

    public IEventBus GetEventBus() => _eventBus;

    public Task<(object Data, Dictionary<string, string[]> Ids)> SoftDeleteAsync(
        Dictionary<string, string[]> ids,
        Context? context = null)
    {
        throw new NotImplementedException();
    }

    public Task<(object Data, Dictionary<string, string[]> Ids)> RestoreAsync(
        Dictionary<string, string[]> ids,
        Context? context = null)
    {
        throw new NotImplementedException();
    }

    private static IEventBus CreateRedisEventBus(ModuleDeclaration declaration, ILogger logger)
    {
        var connectionString = declaration.Options?.GetValueOrDefault("redisConnectionString", "localhost:6379")?.ToString()
            ?? throw new ArgumentException("Redis connection string not provided");

        return new RedisEventBus(connectionString, (ILogger<RedisEventBus>)logger);
    }

    private static IEventBus CreateFastEndpointsEventBus(Dictionary<string, object> dependencies, ILogger logger)
    {
        var fastEndpointsEventBus = dependencies.TryGetValue("fastEndpointsEventBus", out var bus)
            ? (global::FastEndpoints.IEventBus)bus
            : throw new ArgumentException("FastEndpoints EventBus dependency not found");

        return new FastEndpointsEventBusAdapter(fastEndpointsEventBus, (ILogger<FastEndpointsEventBusAdapter>)logger);
    }
}
```

---

## Step 5: Subscriber Pattern (2-3 hours)

### 5.1 Background Service for Subscribers

```csharp
// src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Abstractions/EventSubscriberService.cs

using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace Medusa.Core.EventBus;

/// <summary>
/// Base class for event subscribers (background services)
/// </summary>
public abstract class EventSubscriberService : BackgroundService
{
    protected readonly IEventBus EventBus;
    protected readonly IServiceProvider ServiceProvider;
    protected readonly ILogger Logger;

    protected EventSubscriberService(
        IEventBus eventBus,
        IServiceProvider serviceProvider,
        ILogger logger)
    {
        EventBus = eventBus;
        ServiceProvider = serviceProvider;
        Logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        Logger.LogInformation("Starting event subscriber: {SubscriberName}", GetType().Name);

        // Register subscriptions
        await RegisterSubscriptionsAsync();

        // Keep running
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    /// <summary>
    /// Override this to register event subscriptions
    /// </summary>
    protected abstract Task RegisterSubscriptionsAsync();
}
```

### 5.2 Example Subscriber

```csharp
// Example: Product cache invalidation subscriber

using Medusa.Core.EventBus;
using Medusa.Core.Types;
using Microsoft.Extensions.Caching.Distributed;
using Microsoft.Extensions.Logging;

namespace Medusa.Product.Module.Subscribers;

public class ProductCacheInvalidationSubscriber : EventSubscriberService
{
    public ProductCacheInvalidationSubscriber(
        IEventBus eventBus,
        IServiceProvider serviceProvider,
        ILogger<ProductCacheInvalidationSubscriber> logger)
        : base(eventBus, serviceProvider, logger)
    {
    }

    protected override async Task RegisterSubscriptionsAsync()
    {
        EventBus.Subscribe<ProductCreatedEvent>("product.created", async evt =>
        {
            Logger.LogInformation("Product created: {ProductId}", evt.Data.Id);

            // Invalidate cache
            using var scope = ServiceProvider.CreateScope();
            var cache = scope.ServiceProvider.GetRequiredService<IDistributedCache>();
            await cache.RemoveAsync($"product:{evt.Data.Id}");
        });

        EventBus.Subscribe<ProductUpdatedEvent>("product.updated", async evt =>
        {
            Logger.LogInformation("Product updated: {ProductId}", evt.Data.Id);

            using var scope = ServiceProvider.CreateScope();
            var cache = scope.ServiceProvider.GetRequiredService<IDistributedCache>();
            await cache.RemoveAsync($"product:{evt.Data.Id}");
        });

        await Task.CompletedTask;
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

## Step 6: Service Registration (1 hour)

### 6.1 Extensions for DI

```csharp
// src/Medusa.Core/Medusa.Core.EventBus/Medusa.Core.EventBus.Abstractions/ServiceCollectionExtensions.cs

using Microsoft.Extensions.DependencyInjection;
using Medusa.Core.EventBus.FastEndpoints;
using Medusa.Core.EventBus.Redis;

namespace Medusa.Core.EventBus;

public static class ServiceCollectionExtensions
{
    /// <summary>
    /// Add FastEndpoints EventBus adapter (in-process)
    /// </summary>
    public static IServiceCollection AddFastEndpointsEventBus(this IServiceCollection services)
    {
        // FastEndpoints EventBus is registered by FastEndpoints itself
        services.AddSingleton<IEventBus, FastEndpointsEventBusAdapter>();
        return services;
    }

    /// <summary>
    /// Add Redis EventBus (distributed)
    /// </summary>
    public static IServiceCollection AddRedisEventBus(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddSingleton<IEventBus>(sp =>
        {
            var logger = sp.GetRequiredService<ILogger<RedisEventBus>>();
            return new RedisEventBus(connectionString, logger);
        });

        return services;
    }
}
```

---

## Step 7: Tests (3-4 hours)

### 7.1 Event Bus Tests

```csharp
// tests/Medusa.Core.Tests/EventBusTests.cs

using FluentAssertions;
using Medusa.Core.EventBus;
using Medusa.Core.EventBus.FastEndpoints;
using Medusa.Core.Types;
using Microsoft.Extensions.Logging.Abstractions;

namespace Medusa.Core.Tests;

public class EventBusTests
{
    [Fact]
    public async Task Should_Emit_Event_Immediately_Without_GroupId()
    {
        var fastEndpointsEventBus = new NullLogger<FastEndpointsEventBusAdapter>();
        var eventBus = new FastEndpointsEventBusAdapter(null!, NullLogger<FastEndpointsEventBusAdapter>.Instance);

        var received = false;
        eventBus.Subscribe<TestEvent>("test.event", async evt =>
        {
            received = true;
            await Task.CompletedTask;
        });

        await eventBus.PublishAsync("test.event", new TestEvent { Message = "test" });

        // Give time for async execution
        await Task.Delay(100);

        received.Should().BeTrue();
    }

    [Fact]
    public async Task Should_Stage_Event_With_GroupId()
    {
        var eventBus = new FastEndpointsEventBusAdapter(null!, NullLogger<FastEndpointsEventBusAdapter>.Instance);

        var received = false;
        eventBus.Subscribe<TestEvent>("test.event", async evt =>
        {
            received = true;
            await Task.CompletedTask;
        });

        // Publish with group ID (should be staged)
        await eventBus.PublishAsync("test.event", new TestEvent { Message = "test" }, new EventMetadata
        {
            EventGroupId = "group-123"
        });

        // Event should NOT be emitted yet
        await Task.Delay(100);
        received.Should().BeFalse();

        // Release grouped events
        await eventBus.ReleaseGroupedEventsAsync("group-123");

        // Now event should be emitted
        await Task.Delay(100);
        received.Should().BeTrue();
    }

    [Fact]
    public async Task Should_Clear_Staged_Events_Without_Emitting()
    {
        var eventBus = new FastEndpointsEventBusAdapter(null!, NullLogger<FastEndpointsEventBusAdapter>.Instance);

        var received = false;
        eventBus.Subscribe<TestEvent>("test.event", async evt =>
        {
            received = true;
            await Task.CompletedTask;
        });

        // Publish with group ID
        await eventBus.PublishAsync("test.event", new TestEvent { Message = "test" }, new EventMetadata
        {
            EventGroupId = "group-123"
        });

        // Clear grouped events (should discard without emitting)
        await eventBus.ClearGroupedEventsAsync("group-123");

        await Task.Delay(100);
        received.Should().BeFalse();
    }

    private class TestEvent
    {
        public string Message { get; set; } = string.Empty;
    }
}
```

---

## Step 8: Configuration Example (30 min)

### 8.1 Development Configuration

```csharp
// Program.cs - Development

if (builder.Environment.IsDevelopment())
{
    builder.Services.AddFastEndpointsEventBus();
}
```

### 8.2 Production Configuration

```csharp
// Program.cs - Production

if (!builder.Environment.IsDevelopment())
{
    var redisConnectionString = builder.Configuration.GetConnectionString("Redis");
    builder.Services.AddRedisEventBus(redisConnectionString);
}
```

---

## Success Criteria

- ✅ IEventBus abstraction defined
- ✅ FastEndpoints adapter wraps FastEndpoints EventBus
- ✅ Redis implementation working
- ✅ Event grouping works (stage → release/clear)
- ✅ Subscribers can register for events
- ✅ Tests pass for both implementations
- ✅ Can swap implementations via configuration

---

## Common Issues

### Issue: Events not being released
Check that `ReleaseGroupedEventsAsync` is called after workflow succeeds:
```csharp
if (result.State == TransactionState.DONE) {
    await eventBus.ReleaseGroupedEventsAsync(context.EventGroupId);
}
```

### Issue: Redis connection failed
```bash
docker run -d -p 6379:6379 redis:7-alpine
```

### Issue: Subscribers not receiving events
Make sure subscriber is registered as `BackgroundService`:
```csharp
services.AddHostedService<ProductCacheInvalidationSubscriber>();
```

---

## Next Steps

Once Phase 3 is complete:
1. Test event grouping with workflows
2. Verify Redis persistence
3. Test subscriber pattern
4. Move to [Phase 4: Workflows](./PHASE_4_WORKFLOWS.md)

---

## Estimated Time

- Abstraction: 1-2 hours
- FastEndpoints adapter: 2-3 hours
- Redis implementation: 3-4 hours
- Event Bus module: 2 hours
- Subscriber pattern: 2-3 hours
- Service registration: 1 hour
- Tests: 3-4 hours

**Total: 14-19 hours (1-2 weeks part-time)**
