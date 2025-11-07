# Medusa Core Packages Structure Analysis

## Overview
The Medusa framework is built on 5 core packages that work together to provide:
- Type definitions and interfaces
- Shared utilities and defaults
- Module loading and management system
- Workflow orchestration
- HTTP server and routing framework

### Dependency Graph

```
┌─────────────────────────────────────────┐
│     @medusajs/framework                 │
│  (Main Framework - Entry Point)         │
│                                         │
│ ├─ Exports from all 4 other packages   │
│ ├─ HTTP/Express integration            │
│ ├─ Config management                   │
│ ├─ Database (Mikro-ORM)                │
│ └─ Application bootstrapping           │
└────────────┬────────────────────────────┘
             │
   ┌─────────┼─────────┬──────────┐
   │         │         │          │
   ▼         ▼         ▼          ▼
@medusajs/ @medusajs/ @medusajs/ @medusajs/
modules-sdk workflows-sdk utils types
   │         │         │
   └─────────┼─────────┘
             │
    ┌────────┴────────┐
    │                 │
orchestration  @medusajs/deps
    │                (External libs)
    │
  (Remote Joiner,
   Orchestration)
```

---

## 1. packages/core/types (@medusajs/types)
**PURPOSE**: Central type/interface definitions for the entire Medusa ecosystem

### Key Files/Directories
- `src/common/` - Core types (MedusaContainer, ConfigModule, etc.)
- `src/dml/` - Data Modeling Layer
- `src/modules-sdk/` - Module system types
- `src/workflows-sdk/` - Workflow types
- `src/http/` - HTTP/API types (51 subdirectories)
- Entity domain types: product/, order/, pricing/, inventory/, etc.

### Main Exports
```
MedusaContainer          - DI Container interface (based on Awilix)
ConfigModule            - Configuration interface
ModuleDefinition        - Module metadata
ModuleJoinerConfig      - Query/relationship config
InternalModuleDeclaration
ExternalModuleDeclaration
LoadedModule            - Runtime module instance
IModuleService          - Module service interface
RemoteQueryFunction     - Query execution interface
Context                 - Request context
Logger                  - Logging interface
```

### Key Interfaces/Types
- **MedusaContainer**: DI container extending Awilix with typed resolve<T>()
- **ConfigModule**: Configuration with modules, admin, database, etc.
- **ModuleDefinition**: Defines module: key, dependencies, queryability, default package
- **LoadedModule**: Runtime module instance with __definition, __joinerConfig
- **IModuleService**: Base interface for all module services
- **ModuleJoinerConfig**: GraphQL schema, relationships, primary keys

### Dependencies
- `bignumber.js` - Precision decimal calculations

---

## 2. packages/core/utils (@medusajs/utils)
**PURPOSE**: Shared utilities and default values used across modules

### Key Files/Directories
- `src/common/` - 40+ utility functions
  - String manipulation: camelToSnakeCase, toPascalCase, toHandle, etc.
  - Object utilities: filterObjectByKeys, flattenObjectToKeyValuePairs, mergePluginModules
  - Data conversion: convertItemResponseToUpdateRequest, generateEntityId
  - Validation: isErrorLike, isDefined, tryConvertToNumber
  - Database: handlePostgresDatabaseError
  - GraphQL: stringToSelectRelationObject

- `src/defaults/` - Default data
  - Countries and currencies definitions

- `src/dml/` - Data Model Layer utilities
  - EntityBuilder
  - Entity
  - Relations and Properties
  - Errors

- `src/core-flows/` - Built-in workflow definitions
- `src/graphql/` - GraphQL utilities
- `src/dal/` - Data Access Layer helpers

### Main Exports
```typescript
// Symbols
MedusaModuleType
MedusaModuleProviderType

// Key Classes/Functions
ContainerRegistrationKeys    - DI registration keys
createMedusaContainer()      - Container factory
EntityBuilder                - DSL for defining entities
dynamicImport()             - Dynamic module loading
promiseAll()                - Promise utilities
GraphQLUtils                - GraphQL schema manipulation
ModulesSdkUtils             - Module bootstrapping helpers
```

### Key Interfaces/Functions
- **EntityBuilder**: Fluent API for defining MikroORM entities
- **GraphQLUtils**: mergeTypeDefs, cleanGraphQLSchema, makeExecutableSchema
- **promiseAll()**: Parallel promise execution with error handling
- **ContainerRegistrationKeys**: Standard DI keys (CONFIG_MODULE, LOGGER, PG_CONNECTION, etc.)

### Dependencies
- `@graphql-codegen/*` - GraphQL schema generation
- `@graphql-tools/*` - GraphQL utilities
- `graphql` - GraphQL core
- `bignumber.js` - Precision math
- `zod` - Schema validation
- `jsonwebtoken` - JWT signing
- `pg-connection-string` - Database URL parsing
- `pluralize` - Word pluralization
- `ulid` - Unique ID generation

---

## 3. packages/core/workflows-sdk (@medusajs/workflows-sdk)
**PURPOSE**: Workflow composition and orchestration system

### Key Files
- `src/medusa-workflow.ts` - Global workflow registry
  - Static methods: registerWorkflow, getWorkflow, unregisterWorkflow
  - Central registry for all workflows
  
- `src/utils/composer/` - Workflow DSL for building workflows
  - `create-workflow.ts` - createWorkflow() function
  - `create-step.ts` - createStep() function
  - `create-hook.ts` - Hook creation
  - `when.ts` - Conditional execution
  - `parallelize.ts` - Parallel step execution
  - `transform.ts` - Data transformation between steps
  - `helpers/` - Internal helpers for proxy, response handling

- `src/helper/` - Type definitions and export utilities

### Main Exports
```typescript
createWorkflow()           - Define workflow
createStep()              - Define workflow step
Composer.when()           - Conditional logic
Composer.parallelize()    - Parallel execution
Composer.transform()      - Data transformation
MedusaWorkflow            - Global registry
ExportedWorkflow          - Workflow type
```

### Key Classes/Interfaces
- **MedusaWorkflow**: Static registry for workflow instances
  - getWorkflow(workflowId) - Retrieve registered workflow
  - registerWorkflow(id, fn) - Register new workflow
  
- **Workflow DSL**: 
  - Declarative workflow definition
  - Step composition with inputs/outputs
  - Conditional branching (when)
  - Parallel execution
  - Error handling and retries

### Dependencies
- `@medusajs/orchestration` - LocalWorkflow, RemoteJoiner
- `@medusajs/modules-sdk` - Module loading
- `@medusajs/utils` - Utilities
- `ulid` - Unique IDs for workflow runs

---

## 4. packages/core/modules-sdk (@medusajs/modules-sdk)
**PURPOSE**: Module loading, linking, and remote query system

### Key Files
- `src/medusa-module.ts` - Global module registry and bootstrapper
  - Static methods for lifecycle management
  - Instance caching and resolution
  - Module registration with aliases
  
- `src/medusa-app.ts` - MedusaApp() factory function
  - loadModules() - Load all configured modules
  - Creates RemoteQuery and Link instances
  - Migration management
  - GraphQL schema merging
  
- `src/link.ts` - Link/relationship management between modules
  - Handles entity relationships across modules
  - Cascade deletes/restores
  
- `src/remote-query/` - Query execution system
  - `remote-query.ts` - Main RemoteQuery class
  - Integrates with RemoteJoiner from orchestration
  - Batch data fetching
  
- `src/loaders/` - Module bootstrap loaders
  - `module-loader.ts` - Load module services
  - `register-modules.ts` - Register in container
  - `module-provider-loader.ts` - Load providers
  
- `src/definitions.ts` - ModulesDefinition - metadata for all built-in modules

### Main Exports
```typescript
MedusaApp()                    - Bootstrap entire app
MedusaAppMigrateUp/Down()      - Migration runners
MedusaModule                   - Global registry
Link                           - Entity linking
RemoteQuery                    - Cross-module queries
ModulesDefinition              - Module metadata
```

### Key Classes/Interfaces
- **MedusaApp()**: Factory function returning:
  - modules: Loaded module instances
  - query: RemoteQueryFunction for GraphQL-like queries
  - link: Link instance for entity relationships
  - runMigrations/revertMigrations
  - gqlSchema: Merged GraphQL schema
  
- **MedusaModule**: Static registry
  - bootstrapAll() - Load multiple modules
  - bootstrap<T>() - Load single module
  - getLoadedModules() - Get all instances
  - getJoinerConfig() - Get module's joiner config
  - Lifecycle hooks: onApplicationStart, onApplicationShutdown
  
- **RemoteQuery**: 
  - Query across linked modules
  - Batch fetching with size limits
  - GraphQL-like expand/select syntax
  
- **Link**: 
  - Manage relationships between modules
  - Cascade operations (delete/restore)

### Dependencies
- `@medusajs/orchestration` - RemoteJoiner, LocalWorkflow
- `@medusajs/types` - Type definitions
- `@medusajs/utils` - Utilities and container
- `@medusajs/deps/awilix` - DI container

---

## 5. packages/core/framework (@medusajs/framework)
**PURPOSE**: Main entry point - integrates all packages and provides HTTP/Express server

### Key Files/Directories
- `src/index.ts` - Main exports
- `src/medusa-app-loader.ts` - MedusaAppLoader class
  - Merges module configs with defaults
  - Orchestrates app initialization
  - Container registration
  
- `src/config/` - Configuration loading
  - `config.ts` - ConfigManager class
  - `loader.ts` - Config file loader
  
- `src/container.ts` - Main DI container setup
  - Awilix-based container
  - Singleton container instance
  
- `src/http/` - HTTP server setup (Express)
  - `express-loader.ts` - Express app setup
  - `router.ts` - Route compilation
  - `routes-loader.ts` - Dynamic route loading
  - `routes-sorter.ts` - Route sorting by priority
  - Middleware: authentication, logging, etc.
  
- `src/database/` - Database connection
  - `pg-connection-loader.ts` - PostgreSQL pool setup
  
- `src/workflows/` - Workflow integration
  - Re-exports workflows-sdk
  - Workflow registration hooks
  
- `src/subscribers/` - Event subscribers
  - Event handler registration
  
- `src/jobs/` - Job/queue system
  - `job-loader.ts` - Job discovery and registration
  
- `src/migrations/` - Database migrations
  - `migrator.ts` - Mikro-ORM migration runner
  - `run-migration-scripts.ts` - Custom migration scripts
  
- `src/feature-flags/` - Feature flag system
  - `feature-flag-loader.ts` - Discovery and loading
  - `flag-router.ts` - Feature flag routing
  
- `src/logger/` - Logging setup
- `src/telemetry/` - OpenTelemetry integration
- `src/build-tools/` - Build optimization utilities
- `src/deps/` - Re-exports external dependencies
  - Mikro-ORM libraries
  - OpenTelemetry packages
  - Awilix
  - PostgreSQL client

### Main Exports
```typescript
// From framework
MedusaAppLoader         - App initialization class
defineConfig()          - Config factory function
container               - Global DI container

// Re-exported from modules-sdk
MedusaApp
MedusaAppMigrateUp/Down

// Re-exported from other packages
Query                   - Remote query interface
[All workflow-sdk exports]
[All config exports]
[All database exports]
[All HTTP exports]
...
```

### Key Classes/Interfaces
- **MedusaAppLoader**: Orchestrates entire app initialization
  - mergeDefaultModules()
  - Bootstrap container
  - Load configuration
  - Register modules
  - Setup HTTP server
  - Initialize migrations
  
- **ConfigManager**: Handles medusa-config.ts loading
- **Express Integration**: Full HTTP server setup with:
  - Route discovery from filesystem
  - Middleware ordering
  - Authentication/authorization
  - Error handling

### Nested Package Exports
```
@medusajs/framework/config       → config manager
@medusajs/framework/database     → DB connection
@medusajs/framework/workflows    → Workflow system
@medusajs/framework/modules-sdk  → Module SDK
@medusajs/framework/workflows-sdk → Workflows
@medusajs/framework/http         → HTTP routing
@medusajs/framework/logger       → Logging
@medusajs/framework/telemetry    → OpenTelemetry
@medusajs/framework/utils        → Utilities
@medusajs/framework/types        → Types
```

### Dependencies
- `@medusajs/modules-sdk` - Module system
- `@medusajs/workflows-sdk` - Workflows
- `@medusajs/types` - Type definitions
- `@medusajs/utils` - Utilities
- `@medusajs/orchestration` - Orchestration engine
- `@medusajs/telemetry` - Telemetry
- `express` - HTTP server
- `@medusajs/deps/*` - Bundled external packages

---

## Build Order for .NET Replica

Based on dependencies, the recommended build order is:

### Phase 1: Foundation (No dependencies on other core packages)
1. **Types Package** (@medusajs/types)
   - Pure type definitions, interfaces, enums
   - No runtime code needed
   - Used by all other packages

### Phase 2: Core Infrastructure
2. **Utils Package** (@medusajs/utils)
   - Depends only on external libs and types
   - Provides utility functions for all packages
   - Defines EntityBuilder, GraphQL utilities, constants

3. **Orchestration** (External)
   - Not in core packages but needed by workflows-sdk and modules-sdk
   - Provides RemoteJoiner, LocalWorkflow, orchestration engine

### Phase 3: Module System
4. **Modules-SDK** (@medusajs/modules-sdk)
   - Depends on: types, utils, orchestration
   - Core for module loading and lifecycle
   - Provides MedusaModule, RemoteQuery, Link

5. **Workflows-SDK** (@medusajs/workflows-sdk)
   - Depends on: types, utils, modules-sdk, orchestration
   - Workflow composition and execution
   - Provides createWorkflow, createStep, DSL

### Phase 4: Framework Integration
6. **Framework** (@medusajs/framework)
   - Depends on: All above packages + external libs
   - Integrates everything into Express HTTP server
   - Provides MedusaAppLoader, HTTP routing, migrations, etc.

---

## Key Architectural Patterns

### 1. Dependency Injection (Awilix)
- All packages use MedusaContainer for DI
- Services are registered in container with keys
- ContainerRegistrationKeys defines standard keys
- Enables modularity and testability

### 2. Module System
- Modules are self-contained business logic units
- Each module has:
  - Service(s) implementing business logic
  - JoinerConfig for cross-module queries
  - Optional migrations for database schema
- Modules can be internal (database) or external

### 3. Workflow Orchestration
- Declarative workflow definition via DSL
- Steps are executed sequentially or in parallel
- Can be local (in-process) or distributed
- Supports conditional logic and error handling

### 4. Remote Query System
- GraphQL-like query system across modules
- RemoteJoiner handles relationship resolution
- Batches queries for efficiency
- Integrates with workflow steps

### 5. Plugin Architecture
- Feature flags for feature gating
- Custom links for entity relationships
- Subscribers for event-driven behavior
- Jobs for background processing

---

## Critical Implementation Notes for .NET

1. **Type Safety**: Use strong typing throughout, map from TypeScript interfaces
2. **DI Container**: Need equivalent to Awilix (likely Microsoft.Extensions.DependencyInjection)
3. **Entity Builder**: Fluent API for defining entities (similar to EF Core Fluent API)
4. **Module Lifecycle**: Implement hooks: onApplicationStart, onApplicationShutdown, etc.
5. **Remote Query**: Complex system requiring expression trees/query builders
6. **Workflow Engine**: Need orchestration engine (could use Elsa Workflows or custom)
7. **GraphQL Schema**: Need GraphQL-core .NET implementation
8. **Migrations**: Database migrations system (likely EF Core Migrations)
9. **HTTP Routing**: ASP.NET Core routing system
10. **Configuration**: Support appsettings.json + code-based configuration

