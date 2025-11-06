# Medusa .NET Implementation Roadmap

## Overview

This is your **step-by-step guide** to building Medusa in .NET with FastEndpoints, Redis caching, and modular architecture.

**Time Estimate**: 16-20 weeks (solo developer, part-time)

---

## Quick Reference

| Phase | Focus | Duration | Status |
|-------|-------|----------|--------|
| [Phase 1](#phase-1-foundation) | DI Container + Core Types | 1-2 weeks | 📋 Not Started |
| [Phase 2](#phase-2-module-system) | Module System | 2-3 weeks | 📋 Not Started |
| [Phase 3](#phase-3-event-bus) | Event Bus (Redis) | 1-2 weeks | 📋 Not Started |
| [Phase 4](#phase-4-workflows) | Workflow Engine | 3-4 weeks | 📋 Not Started |
| [Phase 5](#phase-5-fastendpoints) | FastEndpoints + Redis Cache | 2-3 weeks | 📋 Not Started |
| [Phase 6](#phase-6-remote-query) | Cross-Module Queries | 2-3 weeks | 📋 Not Started |
| [Phase 7](#phase-7-first-module) | First Real Module | 1-2 weeks | 📋 Not Started |

**Total**: ~16 weeks

---

## Architecture Stack

### Core Technologies
- **.NET 8+** - Latest LTS version
- **C# 12** - Latest language features
- **FastEndpoints** - REPR pattern, minimal APIs
- **Redis** - Caching, event bus, workflow state
- **PostgreSQL** - Primary database
- **Entity Framework Core** - ORM

### Key Libraries
- **Autofac** - DI Container (closest to Awilix)
- **StackExchange.Redis** - Redis client
- **Npgsql.EntityFrameworkCore.PostgreSQL** - EF Core provider
- **FluentValidation** - Input validation
- **MediatR** (Optional) - Alternative to FastEndpoints Command Bus
- **Ulid** - Distributed IDs

---

## Project Structure

```
Medusa.NET/
├── src/
│   ├── Medusa.Core/                          # Core engine
│   │   ├── Medusa.Core.DependencyInjection/
│   │   ├── Medusa.Core.Types/
│   │   ├── Medusa.Core.EventBus/
│   │   │   ├── Medusa.Core.EventBus.Abstractions/
│   │   │   ├── Medusa.Core.EventBus.InProcess/
│   │   │   └── Medusa.Core.EventBus.Redis/
│   │   ├── Medusa.Core.Workflows/
│   │   ├── Medusa.Core.RemoteQuery/
│   │   └── Medusa.Core.Modules/
│   │
│   ├── Modules/                              # Commerce modules
│   │   ├── Medusa.Product.Module/
│   │   ├── Medusa.Pricing.Module/
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
    ├── Medusa.Core.Tests/
    ├── Medusa.Product.Module.Tests/
    └── Medusa.IntegrationTests/
```

---

## Phase Breakdown

### Phase 1: Foundation (1-2 weeks)
**Goal**: Core abstractions and DI container

**Deliverables**:
- ✅ Autofac-based DI container with `registerAdd()` pattern
- ✅ Core types: `Context`, `FindOptions`, `FilterQuery`, `Event`
- ✅ Repository interfaces
- ✅ Base service with auto-generated CRUD

**Prerequisites**: None

**See**: [PHASE_1_FOUNDATION.md](./PHASE_1_FOUNDATION.md)

---

### Phase 2: Module System (2-3 weeks)
**Goal**: Module discovery, loading, and lifecycle

**Deliverables**:
- ✅ `ModuleDefinition` and `ModuleResolution`
- ✅ Module registry and loader
- ✅ Module lifecycle hooks
- ✅ 2-3 infrastructure modules (Logger, Cache, EventBus)

**Prerequisites**: Phase 1

**See**: [PHASE_2_MODULE_SYSTEM.md](./PHASE_2_MODULE_SYSTEM.md)

---

### Phase 3: Event Bus (1-2 weeks)
**Goal**: Event-driven communication with Redis backend

**Deliverables**:
- ✅ `IEventBus` abstraction
- ✅ In-process implementation (dev)
- ✅ Redis implementation (production)
- ✅ Event grouping for workflows
- ✅ Integration with FastEndpoints EventBus (optional)

**Prerequisites**: Phase 1, Phase 2

**See**: [PHASE_3_EVENT_BUS.md](./PHASE_3_EVENT_BUS.md)

---

### Phase 4: Workflows (3-4 weeks)
**Goal**: Workflow orchestration with automatic compensation

**Deliverables**:
- ✅ Transaction orchestrator
- ✅ Step execution engine
- ✅ Compensation mechanism
- ✅ Workflow DSL (`CreateWorkflow`, `CreateStep`)
- ✅ Parallel execution, conditionals, transforms

**Prerequisites**: Phase 1, Phase 2, Phase 3

**See**: [PHASE_4_WORKFLOWS.md](./PHASE_4_WORKFLOWS.md)

---

### Phase 5: FastEndpoints Integration (2-3 weeks)
**Goal**: API layer with Redis caching and Command Bus

**Deliverables**:
- ✅ FastEndpoints setup with REPR pattern
- ✅ Redis output caching
- ✅ Command Bus integration
- ✅ Pre/Post processors for event grouping
- ✅ Rate limiting, validation

**Prerequisites**: Phase 1-4

**See**: [PHASE_5_FASTENDPOINTS.md](./PHASE_5_FASTENDPOINTS.md)

---

### Phase 6: Remote Query (2-3 weeks)
**Goal**: Cross-module data access

**Deliverables**:
- ✅ Joiner config builder
- ✅ Remote joiner (relationship resolver)
- ✅ Query optimizer (N+1 prevention)
- ✅ Field projection
- ✅ Link system (many-to-many)

**Prerequisites**: Phase 1-5

**See**: [PHASE_6_REMOTE_QUERY.md](./PHASE_6_REMOTE_QUERY.md)

---

### Phase 7: First Real Module (1-2 weeks)
**Goal**: Build Product module end-to-end

**Deliverables**:
- ✅ Product module with all patterns
- ✅ FastEndpoints integration
- ✅ Workflow example
- ✅ Event subscribers
- ✅ Tests

**Prerequisites**: Phase 1-6

**See**: [PHASE_7_FIRST_MODULE.md](./PHASE_7_FIRST_MODULE.md)

---

## Key Decisions

### ✅ Decided

| Decision | Choice | Reason |
|----------|--------|--------|
| **API Framework** | FastEndpoints | Modern, performant, REPR pattern |
| **DI Container** | Autofac | Closest to Medusa's Awilix |
| **Event Bus** | Redis (production) | Persistent, scalable, event grouping |
| **Caching** | Redis (Output Cache) | Shared cache across instances |
| **Command Bus** | FastEndpoints Built-in | Integrated, simple |
| **ORM** | Entity Framework Core | Industry standard, familiar |
| **Database** | PostgreSQL | Same as Medusa |
| **IDs** | Ulid | Distributed, sortable |

### ⚠️ To Decide Later

- [ ] Message broker (if moving to microservices): RabbitMQ vs Kafka
- [ ] Search engine: Elasticsearch vs Meilisearch vs Typesense
- [ ] Observability: OpenTelemetry, Prometheus, Grafana
- [ ] API Gateway (if microservices): Ocelot vs YARP

---

## Redis Architecture

Redis serves **3 purposes** in this architecture:

### 1. Output Cache
```csharp
builder.Services.AddStackExchangeRedisOutputCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "medusa:";
});
```

### 2. Event Bus
```csharp
// Events stored in Redis Streams or Lists
// Shared across all instances
// Survives server restarts
```

### 3. Workflow State (Future)
```csharp
// Store workflow execution state
// Enable resume on failure
// Distributed workflow coordination
```

**Redis Keys Structure**:
```
medusa:cache:products:123           # Output cache
medusa:events:staged:wf_abc         # Staged events (workflow grouping)
medusa:workflows:state:wf_xyz       # Workflow state
medusa:locks:product:123            # Distributed locks
```

---

## Testing Strategy

### Unit Tests
- Core types and utilities
- Module service logic
- Workflow steps (isolated)

### Integration Tests
- Module loading and registration
- Event bus (in-process and Redis)
- Workflow execution end-to-end
- FastEndpoints endpoints

### Architecture Tests
- Module boundaries (no cross-references)
- Dependency rules
- Naming conventions

**Testing Stack**:
- **xUnit** - Test framework
- **FluentAssertions** - Assertion library
- **NSubstitute** - Mocking
- **Testcontainers** - Postgres/Redis containers
- **WebApplicationFactory** - Integration tests

---

## Development Workflow

### Local Development
```bash
# Start dependencies
docker-compose up -d postgres redis

# Run migrations
dotnet ef database update

# Run API
dotnet run --project src/Medusa.API

# Run tests
dotnet test
```

### Environment Variables
```bash
ConnectionStrings__Postgres=Host=localhost;Database=medusa;Username=medusa;Password=medusa
ConnectionStrings__Redis=localhost:6379
ASPNETCORE_ENVIRONMENT=Development
```

---

## Success Criteria

By the end of Phase 7, you should have:

- ✅ **Modular Architecture**: Independent module assemblies
- ✅ **Event-Driven**: Modules communicate via events
- ✅ **Workflow Engine**: Automatic compensation on failure
- ✅ **Cross-Module Queries**: Remote Query working
- ✅ **FastEndpoints**: REPR pattern, Redis caching
- ✅ **One Complete Module**: Product module with all patterns
- ✅ **Tests**: Unit, integration, architecture tests passing

---

## Next Steps

1. **Read this index** to understand the big picture
2. **Start with Phase 1**: [PHASE_1_FOUNDATION.md](./PHASE_1_FOUNDATION.md)
3. **Complete each phase** in order (don't skip!)
4. **Test as you go** (don't accumulate tech debt)
5. **Ask questions** when stuck

---

## Common Pitfalls to Avoid

❌ **Don't**: Build modules before building the engine
✅ **Do**: Build the foundation first (Phases 1-6)

❌ **Don't**: Skip event grouping in Event Bus
✅ **Do**: Implement staging/releasing from day 1

❌ **Don't**: Use direct module-to-module references
✅ **Do**: Communicate via events and RemoteQuery

❌ **Don't**: Build everything in one assembly
✅ **Do**: Create separate projects for each module

❌ **Don't**: Test only at the end
✅ **Do**: Write tests for each phase as you go

---

## Resources

### Reference Documentation
- [MEDUSA_CORE_ARCHITECTURE_FOR_DOTNET.md](../MEDUSA_CORE_ARCHITECTURE_FOR_DOTNET.md) - Core concepts
- [MEDUSA_FASTENDPOINTS_ARCHITECTURE.md](../MEDUSA_FASTENDPOINTS_ARCHITECTURE.md) - FastEndpoints patterns
- [EVENT_BUS_ARCHITECTURE.md](../EVENT_BUS_ARCHITECTURE.md) - Event bus deep dive

### External Resources
- [FastEndpoints Docs](https://fast-endpoints.com/docs)
- [Autofac Documentation](https://autofac.readthedocs.io/)
- [StackExchange.Redis](https://stackexchange.github.io/StackExchange.Redis/)
- [EF Core Docs](https://learn.microsoft.com/en-us/ef/core/)

---

## Progress Tracking

Mark phases as you complete them:

- [ ] Phase 1: Foundation
- [ ] Phase 2: Module System
- [ ] Phase 3: Event Bus
- [ ] Phase 4: Workflows
- [ ] Phase 5: FastEndpoints
- [ ] Phase 6: Remote Query
- [ ] Phase 7: First Module

**Let's build! 🚀**
