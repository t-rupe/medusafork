# Event Bus Architecture - FastEndpoints vs Redis vs RabbitMQ

## Executive Summary

**TL;DR: Don't rely solely on FastEndpoints EventBus. Use the Medusa pattern: Create an abstraction (`IEventBus`) with multiple implementations (In-Process, Redis, RabbitMQ).**

---

## How Medusa Does It

Medusa has **TWO event bus implementations**:

### 1. **Local Event Bus** (Development/Single Instance)
- **Implementation**: Node.js `EventEmitter` (in-process)
- **Location**: `/packages/modules/event-bus-local/`
- **Use Case**: Development, testing, simple deployments
- **Pros**: Fast, simple, no external dependencies
- **Cons**: Events lost on crash, can't scale horizontally

### 2. **Redis Event Bus** (Production/Distributed)
- **Implementation**: BullMQ + Redis (distributed message queue)
- **Location**: `/packages/modules/event-bus-redis/`
- **Use Case**: Production, horizontal scaling, reliability
- **Pros**: Persistent, reliable, horizontal scaling, worker mode
- **Cons**: Requires Redis, network overhead, more complex

**Key Insight**: Both implement the **same interface** (`IEventBusModuleService`), so you can swap them via configuration!

---

## Architecture Comparison

| Feature | FastEndpoints EventBus | Medusa Local | Medusa Redis | RabbitMQ |
|---------|------------------------|--------------|--------------|----------|
| **Type** | In-process | In-process | Distributed Queue | Distributed Queue |
| **Persistence** | ❌ No | ❌ No | ✅ Yes (Redis) | ✅ Yes (Disk) |
| **Horizontal Scaling** | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **Event Grouping** | ❓ Unknown | ✅ Yes | ✅ Yes | Manual |
| **Retry Logic** | ❓ Unknown | ❌ No | ✅ Yes (automatic) | ✅ Yes |
| **Worker Mode** | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **Performance** | ⚡ Fastest | ⚡ Fastest | ⚡ Fast | ⚡ Fast |
| **Complexity** | Simple | Simple | Medium | High |
| **Setup Required** | None | None | Redis | RabbitMQ |
| **Message Routing** | Basic | Basic | Basic | ✅ Advanced |
| **Dead Letter Queue** | ❌ No | ❌ No | ✅ Via BullMQ | ✅ Yes |
| **Best For** | Monolith | Monolith Dev | Monolith Prod | Microservices |

---

## How Each Works

### 1. FastEndpoints EventBus

```csharp
// FastEndpoints uses in-process pub/sub
app.MapGet("/products", async (IEventBus eventBus) =>
{
    await eventBus.PublishAsync(new ProductCreated { Id = "123" });
});

// Subscriber in same process
public class ProductCreatedHandler : IEventHandler<ProductCreated>
{
    public async Task HandleAsync(ProductCreated evt, CancellationToken ct)
    {
        // Handle event
    }
}
```

**How it works:**
- Events stored in memory (no persistence)
- Subscribers called directly via delegates
- If app crashes, events are lost
- If you scale to 2 instances, each instance has separate event bus
- **Workflow issue**: If workflow fails after event emitted, event already executed (can't rollback)

---

### 2. Medusa Local Event Bus (In-Process)

```typescript
// Uses Node.js EventEmitter
class LocalEventBusService {
  protected readonly eventEmitter_: EventEmitter
  protected groupedEventsMap_: Map<string, Message[]>  // For workflow grouping!

  async emit(eventData: Message) {
    const eventGroupId = eventData.metadata?.eventGroupId

    if (eventGroupId) {
      // STAGE the event (don't emit yet)
      await this.groupEvent(eventGroupId, eventData)
    } else {
      // Emit immediately
      this.eventEmitter_.emit(eventData.name, eventData)
    }
  }

  async releaseGroupedEvents(eventGroupId: string) {
    // Release all staged events atomically
    const groupedEvents = this.groupedEventsMap_.get(eventGroupId)

    for (const event of groupedEvents) {
      this.eventEmitter_.emit(event.name, event)
    }

    this.groupedEventsMap_.delete(eventGroupId)
  }
}
```

**Key Features:**
- ✅ **Event Grouping** - Stage events and release atomically (for workflows!)
- ✅ **Interceptors** - Hook into event emission
- ✅ **Star Subscriber** - Subscribe to all events (`*`)
- ❌ Not persistent
- ❌ Can't scale horizontally

---

### 3. Medusa Redis Event Bus (Distributed)

```typescript
// Uses BullMQ (Redis-backed job queue)
class RedisEventBusService {
  protected queue_: Queue          // BullMQ queue
  protected bullWorker_: Worker    // Background worker

  async emit(eventData: Message) {
    const eventGroupId = eventData.metadata?.eventGroupId

    if (eventGroupId) {
      // STAGE in Redis list
      await this.eventBusRedisConnection_.rpush(
        `staging:${eventGroupId}`,
        JSON.stringify(eventData)
      )
    } else {
      // Add to BullMQ queue (persistent)
      await this.queue_.add(eventData.name, eventData, {
        attempts: 3,  // Retry 3 times
        removeOnComplete: true
      })
    }
  }

  async releaseGroupedEvents(eventGroupId: string) {
    // Get all staged events from Redis
    const groupedEvents = await this.eventBusRedisConnection_
      .lrange(`staging:${eventGroupId}`, 0, -1)

    // Add all to queue atomically
    await this.queue_.addBulk(groupedEvents)

    // Clear staging area
    await this.eventBusRedisConnection_.unlink(`staging:${eventGroupId}`)
  }

  // Background worker processes events
  worker_ = async (job: BullJob) => {
    const subscribers = this.eventToSubscribersMap.get(job.name) || []

    // Track which subscribers completed (for retries)
    const completedSubscribers = job.data.completedSubscriberIds || []

    // Execute subscribers
    await Promise.all(subscribers.map(sub => sub.subscriber(job.data)))

    // If any failed and retries configured, throw to trigger retry
    if (someSubscribersFailed && job.attemptsMade < job.opts.attempts) {
      throw new Error("Retry...")
    }
  }
}
```

**Key Features:**
- ✅ **Persistent** - Events stored in Redis (survive crashes)
- ✅ **Horizontal Scaling** - Multiple instances share the same queue
- ✅ **Event Grouping** - Stage events in Redis lists
- ✅ **Automatic Retries** - Configurable retry logic
- ✅ **Worker Mode** - Separate worker processes for event processing
- ✅ **Partial Retry** - Track which subscribers failed, only retry those
- ⚠️ Requires Redis

---

### 4. RabbitMQ

```csharp
// More complex setup
var factory = new ConnectionFactory() { HostName = "localhost" };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

channel.ExchangeDeclare("medusa-events", ExchangeType.Topic);
channel.QueueDeclare("product-events", durable: true);
channel.QueueBind("product-events", "medusa-events", "product.*");

// Publish
var body = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(evt));
channel.BasicPublish("medusa-events", "product.created", null, body);

// Subscribe
var consumer = new EventingBasicConsumer(channel);
consumer.Received += (model, ea) =>
{
    var body = ea.Body.ToArray();
    var message = Encoding.UTF8.GetString(body);
    // Handle event
    channel.BasicAck(ea.DeliveryTag, false);
};
channel.BasicConsume("product-events", false, consumer);
```

**Key Features:**
- ✅ **Most Reliable** - Built for enterprise messaging
- ✅ **Advanced Routing** - Topic exchanges, fanout, headers
- ✅ **Dead Letter Queues** - Failed messages go to DLQ
- ✅ **Message Priority** - Prioritize important events
- ✅ **Horizontal Scaling** - Multiple consumers on same queue
- ⚠️ More complex than Redis
- ⚠️ Overkill for monolithic architecture

---

## Critical Feature: Event Grouping (For Workflows)

**This is the KILLER feature** for workflow compensation:

### Workflow Example

```csharp
// Workflow creates order
var workflow = CreateWorkflow("checkout", (input) => {
    var order = CreateOrderStep(input);       // Step 1
    var inventory = ReserveInventoryStep(order);  // Step 2
    var payment = ChargePaymentStep(order);   // Step 3 - FAILS!

    return order;
});

// What happens to events?
```

### Without Event Grouping (FastEndpoints)

```csharp
// CreateOrderStep
var order = await _orderService.CreateAsync(input);
await _eventBus.PublishAsync(new OrderCreated { Id = order.Id });  // ❌ Emitted immediately!

// Subscriber executes
public class OrderCreatedHandler : IEventHandler<OrderCreated>
{
    public async Task HandleAsync(OrderCreated evt) {
        await _emailService.SendOrderConfirmationAsync(evt.Id);  // ❌ Email sent!
    }
}

// Then ChargePaymentStep fails...
// Compensation deletes the order...
// But the customer already got a confirmation email! ❌ BAD!
```

### With Event Grouping (Medusa Pattern)

```csharp
// CreateOrderStep
var context = new Context { EventGroupId = Ulid.NewUlid().ToString() };
var order = await _orderService.CreateAsync(input, context);
await _eventBus.PublishAsync(
    new OrderCreated { Id = order.Id },
    new EventMetadata { EventGroupId = context.EventGroupId }
);
// ✅ Event STAGED, not emitted!

// Workflow continues...
// ChargePaymentStep fails...
// Compensation runs...

// Workflow catches error:
if (result.State == TransactionState.FAILED) {
    // ✅ Clear the staged events (never emit them)
    await _eventBus.ClearGroupedEventsAsync(context.EventGroupId);
}

// If workflow succeeded:
if (result.State == TransactionState.DONE) {
    // ✅ Release all events atomically
    await _eventBus.ReleaseGroupedEventsAsync(context.EventGroupId);
    // Now subscribers execute (email sent)
}
```

**This is CRITICAL for transactional workflows!**

---

## My Recommendation

### Use the Medusa Pattern: Abstraction + Multiple Implementations

```csharp
// 1. Define abstraction
public interface IEventBus
{
    Task PublishAsync<T>(
        string eventName,
        T data,
        EventMetadata? metadata = null,
        Context? context = null);

    void Subscribe<T>(string eventName, Func<Event<T>, Task> handler);

    Task ReleaseGroupedEventsAsync(string eventGroupId);
    Task ClearGroupedEventsAsync(string eventGroupId);
}

// 2. Implement multiple providers
public class InProcessEventBus : IEventBus { }      // For dev
public class RedisEventBus : IEventBus { }          // For production
public class RabbitMqEventBus : IEventBus { }       // Optional

// 3. Register based on configuration
services.AddMedusaEventBus(options =>
{
    if (builder.Environment.IsDevelopment())
    {
        options.UseInProcess();  // Fast, simple
    }
    else
    {
        options.UseRedis(redisConnectionString);  // Persistent, scalable
    }
});
```

---

## Implementation Guide

### Phase 1: Core Abstraction

```csharp
// Medusa.Core.EventBus/IEventBus.cs

namespace Medusa.Core.EventBus;

public interface IEventBus
{
    Task PublishAsync<T>(
        string eventName,
        T data,
        EventMetadata? metadata = null,
        Context? context = null);

    void Subscribe<T>(string eventName, Func<Event<T>, Task> handler);

    Task ReleaseGroupedEventsAsync(string eventGroupId);
    Task ClearGroupedEventsAsync(string eventGroupId);
}

public class Event<T>
{
    public string Name { get; set; } = string.Empty;
    public EventMetadata? Metadata { get; set; }
    public T Data { get; set; } = default!;
}

public class EventMetadata
{
    public string? EventGroupId { get; set; }
    public Dictionary<string, object>? Additional { get; set; }
}
```

### Phase 2: In-Process Implementation

```csharp
// Medusa.EventBus.InProcess/InProcessEventBus.cs

using System.Collections.Concurrent;

namespace Medusa.EventBus.InProcess;

public class InProcessEventBus : IEventBus
{
    private readonly ConcurrentDictionary<string, List<Delegate>> _subscribers = new();
    private readonly ConcurrentDictionary<string, List<Event<object>>> _groupedEvents = new();

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

        // If event has group ID, stage it
        if (metadata?.EventGroupId != null)
        {
            await StageEventAsync(metadata.EventGroupId, evt);
            return;
        }

        // Otherwise emit immediately
        await EmitEventAsync(evt);
    }

    private async Task StageEventAsync<T>(string groupId, Event<T> evt)
    {
        var events = _groupedEvents.GetOrAdd(groupId, _ => new List<Event<object>>());

        lock (events)
        {
            events.Add(new Event<object>
            {
                Name = evt.Name,
                Data = evt.Data!,
                Metadata = evt.Metadata
            });
        }

        await Task.CompletedTask;
    }

    private async Task EmitEventAsync<T>(Event<T> evt)
    {
        if (!_subscribers.TryGetValue(evt.Name, out var handlers))
            return;

        var tasks = handlers
            .Cast<Func<Event<T>, Task>>()
            .Select(handler => handler(evt));

        await Task.WhenAll(tasks);
    }

    public async Task ReleaseGroupedEventsAsync(string eventGroupId)
    {
        if (!_groupedEvents.TryRemove(eventGroupId, out var events))
            return;

        foreach (var evt in events)
        {
            await EmitEventAsync(evt);
        }
    }

    public async Task ClearGroupedEventsAsync(string eventGroupId)
    {
        _groupedEvents.TryRemove(eventGroupId, out _);
        await Task.CompletedTask;
    }

    public void Subscribe<T>(string eventName, Func<Event<T>, Task> handler)
    {
        var handlers = _subscribers.GetOrAdd(eventName, _ => new List<Delegate>());

        lock (handlers)
        {
            handlers.Add(handler);
        }
    }
}
```

### Phase 3: Redis Implementation

```csharp
// Medusa.EventBus.Redis/RedisEventBus.cs

using StackExchange.Redis;

namespace Medusa.EventBus.Redis;

public class RedisEventBus : IEventBus, IDisposable
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _db;
    private readonly ISubscriber _subscriber;
    private readonly ConcurrentDictionary<string, List<Func<Event<object>, Task>>> _handlers = new();

    public RedisEventBus(string connectionString)
    {
        _redis = ConnectionMultiplexer.Connect(connectionString);
        _db = _redis.GetDatabase();
        _subscriber = _redis.GetSubscriber();

        // Subscribe to Redis pub/sub
        _subscriber.Subscribe("medusa:events:*", async (channel, message) =>
        {
            var eventName = channel.ToString().Replace("medusa:events:", "");
            var evt = JsonSerializer.Deserialize<Event<object>>(message!);

            if (_handlers.TryGetValue(eventName, out var handlers))
            {
                await Task.WhenAll(handlers.Select(h => h(evt!)));
            }
        });
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
            var json = JsonSerializer.Serialize(evt);
            await _db.ListRightPushAsync($"medusa:staged:{metadata.EventGroupId}", json);

            // Set TTL (10 minutes)
            await _db.KeyExpireAsync($"medusa:staged:{metadata.EventGroupId}", TimeSpan.FromMinutes(10));
            return;
        }

        // Otherwise publish to Redis pub/sub
        var channel = $"medusa:events:{eventName}";
        var payload = JsonSerializer.Serialize(evt);
        await _subscriber.PublishAsync(channel, payload);
    }

    public async Task ReleaseGroupedEventsAsync(string eventGroupId)
    {
        var key = $"medusa:staged:{eventGroupId}";

        // Get all staged events
        var events = await _db.ListRangeAsync(key);

        // Publish each event
        foreach (var eventJson in events)
        {
            var evt = JsonSerializer.Deserialize<Event<object>>(eventJson!);
            var channel = $"medusa:events:{evt!.Name}";
            await _subscriber.PublishAsync(channel, eventJson!);
        }

        // Clear staged events
        await _db.KeyDeleteAsync(key);
    }

    public async Task ClearGroupedEventsAsync(string eventGroupId)
    {
        await _db.KeyDeleteAsync($"medusa:staged:{eventGroupId}");
    }

    public void Subscribe<T>(string eventName, Func<Event<T>, Task> handler)
    {
        var handlers = _handlers.GetOrAdd(eventName, _ => new List<Func<Event<object>, Task>>());

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
    }

    public void Dispose()
    {
        _redis?.Dispose();
    }
}
```

### Phase 4: FastEndpoints Integration (Optional Hybrid)

```csharp
// If you want to use FastEndpoints EventBus as a fallback

public class FastEndpointsEventBusAdapter : IEventBus
{
    private readonly FastEndpoints.IEventBus _fastEndpointsEventBus;
    private readonly ConcurrentDictionary<string, List<Event<object>>> _groupedEvents = new();

    public async Task PublishAsync<T>(
        string eventName,
        T data,
        EventMetadata? metadata = null,
        Context? context = null)
    {
        // If grouped, stage it (FastEndpoints doesn't support this natively)
        if (metadata?.EventGroupId != null)
        {
            var events = _groupedEvents.GetOrAdd(metadata.EventGroupId, _ => new());
            events.Add(new Event<object> { Name = eventName, Data = data!, Metadata = metadata });
            return;
        }

        // Otherwise use FastEndpoints
        await _fastEndpointsEventBus.PublishAsync(new FastEndpointsEvent<T>
        {
            EventType = eventName,
            Data = data
        });
    }

    // Implement grouping manually since FastEndpoints doesn't have it
}
```

---

## Configuration

```csharp
// Medusa.API/Program.cs

var builder = WebApplication.CreateBuilder(args);

// Register event bus based on environment
if (builder.Environment.IsDevelopment())
{
    // Use in-process for dev (fast, simple)
    builder.Services.AddSingleton<IEventBus, InProcessEventBus>();
}
else
{
    // Use Redis for production (persistent, scalable)
    var redisConnectionString = builder.Configuration.GetConnectionString("Redis");
    builder.Services.AddSingleton<IEventBus>(sp => new RedisEventBus(redisConnectionString));
}

// Or use FastEndpoints as fallback
// builder.Services.AddSingleton<IEventBus, FastEndpointsEventBusAdapter>();
```

---

## When to Use Each

### Use In-Process Event Bus When:
- ✅ Single instance deployment
- ✅ Development/testing
- ✅ Low volume (<1000 events/sec)
- ✅ Events can be lost (non-critical)

### Use Redis Event Bus When:
- ✅ Production deployment
- ✅ Horizontal scaling (multiple instances)
- ✅ Events must be persistent
- ✅ Need retries and reliability
- ✅ Medium-high volume (>1000 events/sec)

### Use RabbitMQ When:
- ✅ Microservices architecture (modules as separate services)
- ✅ Need advanced routing (topic exchanges, etc.)
- ✅ Need message priorities
- ✅ Very high volume (>10000 events/sec)
- ✅ Enterprise messaging requirements

---

## Performance Comparison

```
In-Process:     ~1,000,000 events/sec  (no network overhead)
Redis:          ~50,000 events/sec     (Redis pub/sub)
RabbitMQ:       ~20,000 events/sec     (more overhead, more features)
```

For Medusa use case:
- In-process is **more than enough** for development
- Redis is **more than enough** for production (even at massive scale)
- RabbitMQ is **overkill** unless you're doing microservices

---

## Summary

### Don't Rely on FastEndpoints EventBus Alone

**Why?**
1. ❌ No event grouping (breaks workflow compensation)
2. ❌ No persistence (events lost on crash)
3. ❌ Can't scale horizontally (each instance separate)
4. ❓ Unknown retry capabilities

### Use the Medusa Pattern Instead

1. ✅ **Create `IEventBus` abstraction**
2. ✅ **Implement in-process version** (for dev)
3. ✅ **Implement Redis version** (for production)
4. ✅ **Support event grouping** (for workflows)
5. ✅ **Swap via configuration**

### Recommended Stack

```
Development:    InProcessEventBus    (fast, simple)
Production:     RedisEventBus        (persistent, scalable)
Future:         RabbitMqEventBus     (if you go microservices)
```

**This gives you the best of all worlds!**
