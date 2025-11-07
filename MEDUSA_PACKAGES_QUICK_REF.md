# Medusa Core Packages - Quick Reference

## Package Summary Table

| Package | Purpose | Key Class/Export | Files | Size |
|---------|---------|------------------|-------|------|
| **types** | Type definitions | MedusaContainer, ModuleDefinition | 48 dirs | ~60KB |
| **utils** | Utilities & defaults | EntityBuilder, createMedusaContainer | 30+ files | ~80KB |
| **workflows-sdk** | Workflow DSL | createWorkflow, Composer | 20 files | ~30KB |
| **modules-sdk** | Module system | MedusaApp, MedusaModule, Link | 15 files | ~40KB |
| **framework** | HTTP integration | MedusaAppLoader, configManager | 22 dirs | ~120KB |

---

## Critical Files by Package

### @medusajs/types
```
src/common/
  ├─ medusa-container.ts      (MedusaContainer interface)
  ├─ config-module.ts         (ConfigModule interface)
  └─ medusa-cli.ts            (CLI types)

src/modules-sdk/
  └─ index.ts                 (Module system types)

src/http/
  └─ [51 subdirs]             (API/REST endpoint types)
```

### @medusajs/utils
```
src/
  ├─ common/
  │   ├─ errors.ts
  │   ├─ generate-entity-id.ts
  │   ├─ to-handle.ts
  │   └─ [37 more utilities]
  ├─ dml/
  │   ├─ entity-builder.ts     (EntityBuilder class)
  │   └─ entity.ts
  ├─ defaults/
  │   ├─ countries.ts
  │   └─ currencies.ts
  ├─ graphql/
  │   └─ index.ts              (GraphQL utilities)
  └─ index.ts                  (Main exports)
```

### @medusajs/workflows-sdk
```
src/
  ├─ medusa-workflow.ts         (Global registry)
  ├─ index.ts                   (Main exports)
  ├─ utils/composer/
  │   ├─ create-workflow.ts     (createWorkflow)
  │   ├─ create-step.ts         (createStep)
  │   ├─ when.ts                (Conditional logic)
  │   ├─ parallelize.ts         (Parallel steps)
  │   └─ transform.ts           (Data transformation)
  └─ helper/
      └─ index.ts              (Type exports)
```

### @medusajs/modules-sdk
```
src/
  ├─ medusa-module.ts           (Module registry - 890 lines!)
  ├─ medusa-app.ts              (App factory - 684 lines!)
  ├─ link.ts                    (Entity linking - 400+ lines)
  ├─ remote-query/
  │   ├─ remote-query.ts        (Query execution)
  │   └─ query.ts               (Query builder)
  ├─ loaders/
  │   ├─ module-loader.ts
  │   ├─ register-modules.ts
  │   └─ module-provider-loader.ts
  └─ definitions.ts             (Module metadata)
```

### @medusajs/framework
```
src/
  ├─ medusa-app-loader.ts       (MedusaAppLoader class)
  ├─ container.ts               (Global DI container)
  ├─ index.ts                   (Main exports)
  ├─ config/
  │   ├─ config.ts              (ConfigManager)
  │   ├─ loader.ts
  │   └─ types.ts
  ├─ http/
  │   ├─ express-loader.ts
  │   ├─ router.ts              (Route compilation)
  │   ├─ routes-loader.ts       (Dynamic discovery)
  │   ├─ routes-sorter.ts
  │   └─ middlewares/
  ├─ database/
  │   └─ pg-connection-loader.ts
  ├─ migrations/
  │   ├─ migrator.ts
  │   └─ run-migration-scripts.ts
  ├─ jobs/
  │   └─ job-loader.ts
  ├─ feature-flags/
  │   ├─ feature-flag-loader.ts
  │   └─ flag-router.ts
  ├─ workflows/
  │   └─ [Re-exports workflows-sdk]
  ├─ subscribers/
  ├─ logger/
  ├─ telemetry/
  ├─ build-tools/
  └─ deps/
      └─ [External lib re-exports]
```

---

## Export Chains

### How users import from @medusajs/framework

```javascript
// Direct imports
import { MedusaAppLoader, defineConfig } from "@medusajs/framework"
import { container } from "@medusajs/framework"

// Subpath exports
import { createWorkflow, createStep } from "@medusajs/framework/workflows"
import { createQuery } from "@medusajs/framework/modules-sdk"
import type { MedusaContainer } from "@medusajs/framework/types"
import { EntityBuilder } from "@medusajs/framework/utils"

// Nested subpaths
import { PostgresConnectionManager } from "@medusajs/framework/database"
```

### Package Dependency Flow

```
User Application
       ↓
   framework (entry point)
       ↓
   ├─→ modules-sdk
   ├─→ workflows-sdk
   ├─→ utils
   ├─→ types
   └─→ orchestration (external)
       ↓
   External Libraries
   (Express, MikroORM, GraphQL, etc.)
```

---

## Key Concepts Mapping

| Concept | File | Class/Function | Purpose |
|---------|------|-----------------|---------|
| **Dependency Injection** | types/common/medusa-container.ts | MedusaContainer | Service registration & resolution |
| **Module Definition** | types/modules-sdk/index.ts | ModuleDefinition | Module metadata |
| **Config Loading** | framework/config/config.ts | ConfigManager | medusa-config.ts parsing |
| **Entity Definition** | utils/dml/entity-builder.ts | EntityBuilder | Fluent entity DSL |
| **Workflow Definition** | workflows-sdk/utils/composer/create-workflow.ts | createWorkflow() | Workflow composition |
| **App Bootstrap** | modules-sdk/medusa-app.ts | MedusaApp() | Initialize all modules |
| **Module Registry** | modules-sdk/medusa-module.ts | MedusaModule | Global module cache |
| **Cross-Module Queries** | modules-sdk/remote-query/remote-query.ts | RemoteQuery | Query execution |
| **Entity Linking** | modules-sdk/link.ts | Link | Manage relationships |
| **HTTP Server** | framework/http/express-loader.ts | ExpressLoader | Express setup |
| **Route Discovery** | framework/http/routes-loader.ts | RoutesLoader | Auto-load routes |
| **Migrations** | framework/migrations/migrator.ts | Migrator | Database schema changes |
| **Feature Flags** | framework/feature-flags/feature-flag-loader.ts | FlagLoader | Feature gating |

---

## Initialization Sequence

```
1. Load medusa-config.ts
   ↓
2. Create DI container
   ├─ Register ConfigModule
   ├─ Register Logger
   └─ Register Database connection
   ↓
3. Load all modules via MedusaApp()
   ├─ For each module:
   │  ├─ Load service
   │  ├─ Register in container
   │  └─ Get JoinerConfig
   ├─ Register modules in MedusaModule
   └─ Merge all GraphQL schemas
   ↓
4. Initialize Link module
   ├─ Build entity relationships
   └─ Setup cascade operations
   ↓
5. Create RemoteQuery instance
   ├─ Setup batch fetching
   └─ Register with RemoteJoiner
   ↓
6. Setup Express HTTP server
   ├─ Load middleware
   ├─ Discover & load routes
   ├─ Setup error handling
   └─ Start listening
   ↓
7. Initialize subscribers & jobs
   ├─ Load event subscribers
   └─ Load background jobs
   ↓
8. Call onApplicationStart hooks
   ↓
9. Server ready!
```

---

## Module Dependency Resolution Example

If Product module depends on EventBus:

```
ConfigModule defines:
  {
    "product": { /* config */ },
    "event_bus": { /* config */ }
  }

MedusaAppLoader.mergeDefaultModules():
  Merges with default definitions from ModulesDefinition

MedusaApp():
  1. Resolves product needs event_bus
  2. Loads event_bus first (dependency order)
  3. Registers event_bus in container
  4. Injects into product service constructor
  5. Product service gets EventBusService instance

At runtime:
  MedusaModule.getLoadedModules()
    Returns: { product: ProductService, event_bus: EventBusService }
    
  RemoteQuery can join:
    - Product data from product module
    - Event logs from event_bus if queryable
```

---

## Common Patterns

### Registering a Custom Module

```typescript
// In medusa-config.ts
modules: {
  "my_custom_module": {
    resolve: "./src/modules/my-custom-module",
    options: { /* module options */ }
  }
}

// MedusaApp() will:
// 1. Load module from path
// 2. Call module.initialize(container, options)
// 3. Register services in container
// 4. Add joiner config for queries
```

### Creating a Workflow

```typescript
// In src/workflows/my-workflow.ts
const myWorkflow = createWorkflow(
  "my-workflow",
  (input: MyInput) => {
    const result1 = step1(input)
    const result2 = step2(result1)
    return { result1, result2 }
  }
)

// MedusaWorkflow.registerWorkflow("my-workflow", myWorkflow)
// Can be called with: MedusaWorkflow.getWorkflow("my-workflow").run(input)
```

### Querying Across Modules

```typescript
// Get RemoteQuery from MedusaApp
const { query } = await MedusaApp()

// Query data with cross-module joins
const products = await query.graph({
  entity: "product",
  fields: ["id", "name", { pricing: ["amount"] }]
})

// RemoteQuery uses RemoteJoiner to:
// 1. Query product module
// 2. Join with pricing data
// 3. Return merged results
```

---

## Integration Points for .NET

1. **DI Container Registration**
   - Map MedusaContainer → IServiceCollection
   - ContainerRegistrationKeys → String constants

2. **Module Loading**
   - Module discovery (filesystem or assembly scanning)
   - Service registration pattern
   - Dependency resolution order

3. **Workflow Engine**
   - Step definition and execution
   - Parallel/conditional execution
   - Error handling and retries

4. **Query System**
   - GraphQL schema building
   - Expression tree/LINQ query building
   - Cross-module joins

5. **HTTP Routing**
   - Attribute-based or convention-based routing
   - Middleware ordering
   - Request/response transformation

6. **Entity Mapping**
   - Fluent configuration API
   - Relationship definitions
   - Migration generation

