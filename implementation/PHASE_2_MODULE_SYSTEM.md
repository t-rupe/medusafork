# Phase 2: Module System (2-3 weeks)

## Goal
Build the module discovery, loading, and lifecycle management system that enables independent, decoupled modules.

## Prerequisites
- ✅ Phase 1: Foundation complete
- Understanding of dependency injection
- Understanding of reflection in C#

## Deliverables
- ✅ `ModuleDefinition` and `ModuleResolution` types
- ✅ Module registry (static)
- ✅ Module loader with dependency resolution
- ✅ Lifecycle hooks (startup, shutdown)
- ✅ 2-3 working infrastructure modules (Logger, Cache, EventBus)

---

## Step 1: Module Core Types (2-3 hours)

### 1.1 Create Module SDK Project

```bash
dotnet new classlib -n Medusa.Core.Modules -o src/Medusa.Core/Medusa.Core.Modules
dotnet sln add src/Medusa.Core/Medusa.Core.Modules

cd src/Medusa.Core/Medusa.Core.Modules
dotnet add reference ../Medusa.Core.Types
dotnet add reference ../Medusa.Core.DependencyInjection
```

### 1.2 Module Definition

```csharp
// src/Medusa.Core/Medusa.Core.Modules/ModuleDefinition.cs

namespace Medusa.Core.Modules;

/// <summary>
/// Metadata about a module. Defines what the module is, what it provides, and what it depends on.
/// </summary>
public class ModuleDefinition
{
    /// <summary>
    /// Unique identifier for the module (e.g., "product", "pricing")
    /// </summary>
    public string Key { get; set; } = string.Empty;

    /// <summary>
    /// Human-readable name (e.g., "Product Management")
    /// </summary>
    public string Label { get; set; } = string.Empty;

    /// <summary>
    /// Default package/assembly name
    /// </summary>
    public string DefaultPackage { get; set; } = string.Empty;

    /// <summary>
    /// Whether this module must be loaded
    /// </summary>
    public bool IsRequired { get; set; }

    /// <summary>
    /// Whether this module can be queried via RemoteQuery
    /// </summary>
    public bool IsQueryable { get; set; }

    /// <summary>
    /// Other modules this module depends on (e.g., ["eventBus", "logger"])
    /// </summary>
    public string[] Dependencies { get; set; } = Array.Empty<string>();

    /// <summary>
    /// Default configuration for the module
    /// </summary>
    public Dictionary<string, object>? DefaultOptions { get; set; }
}
```

### 1.3 Module Resolution

```csharp
// src/Medusa.Core/Medusa.Core.Modules/ModuleResolution.cs

namespace Medusa.Core.Modules;

/// <summary>
/// Resolved module with its actual implementation path and configuration.
/// </summary>
public class ModuleResolution
{
    /// <summary>
    /// Path to the module assembly or false if disabled
    /// </summary>
    public string? ResolutionPath { get; set; }

    /// <summary>
    /// The module definition (metadata)
    /// </summary>
    public ModuleDefinition Definition { get; set; } = null!;

    /// <summary>
    /// Resolved dependencies (service names to inject)
    /// </summary>
    public string[] Dependencies { get; set; } = Array.Empty<string>();

    /// <summary>
    /// Module declaration (how it should be loaded)
    /// </summary>
    public ModuleDeclaration Declaration { get; set; } = null!;

    /// <summary>
    /// Module-specific options
    /// </summary>
    public Dictionary<string, object>? Options { get; set; }
}
```

### 1.4 Module Declaration

```csharp
// src/Medusa.Core/Medusa.Core.Modules/ModuleDeclaration.cs

namespace Medusa.Core.Modules;

/// <summary>
/// How a module should be loaded and configured.
/// </summary>
public class ModuleDeclaration
{
    /// <summary>
    /// Where the module runs (internal to app, or external service)
    /// </summary>
    public ModuleScope Scope { get; set; } = ModuleScope.Internal;

    /// <summary>
    /// Override the default resolution path
    /// </summary>
    public string? Resolve { get; set; }

    /// <summary>
    /// Module-specific options
    /// </summary>
    public Dictionary<string, object>? Options { get; set; }

    /// <summary>
    /// Alias for multiple instances (e.g., "redis-cache", "memory-cache")
    /// </summary>
    public string? Alias { get; set; }

    /// <summary>
    /// Mark as primary instance when using aliases
    /// </summary>
    public bool Main { get; set; }

    /// <summary>
    /// Worker mode for background processing
    /// </summary>
    public WorkerMode WorkerMode { get; set; } = WorkerMode.Shared;
}

public enum ModuleScope
{
    Internal,  // Module runs in same process
    External   // Module is external service (future: microservices)
}

public enum WorkerMode
{
    Shared,    // Module used by all processes
    Worker,    // Module only runs on worker processes
    Server     // Module only runs on web server processes
}
```

### 1.5 Module Service Interface

```csharp
// src/Medusa.Core/Medusa.Core.Modules/IModuleService.cs

using Medusa.Core.Types;

namespace Medusa.Core.Modules;

/// <summary>
/// Base interface all modules must implement.
/// </summary>
public interface IModuleService
{
    /// <summary>
    /// Module metadata
    /// </summary>
    ModuleDefinition Definition { get; }

    /// <summary>
    /// Configuration for cross-module queries (RemoteQuery)
    /// </summary>
    ModuleJoinerConfig? JoinerConfig { get; }

    /// <summary>
    /// Lifecycle hooks
    /// </summary>
    ModuleHooks? Hooks { get; }

    /// <summary>
    /// Soft delete entities in this module (for cascade operations)
    /// </summary>
    Task<(object Data, Dictionary<string, string[]> Ids)> SoftDeleteAsync(
        Dictionary<string, string[]> ids,
        Context? context = null);

    /// <summary>
    /// Restore soft-deleted entities
    /// </summary>
    Task<(object Data, Dictionary<string, string[]> Ids)> RestoreAsync(
        Dictionary<string, string[]> ids,
        Context? context = null);
}

public class ModuleHooks
{
    public Func<IServiceProvider, Task>? OnApplicationStart { get; set; }
    public Func<IServiceProvider, Task>? OnApplicationShutdown { get; set; }
    public Func<IServiceProvider, Task>? OnApplicationPrepareShutdown { get; set; }
}

/// <summary>
/// Module joiner config for RemoteQuery (Phase 6)
/// </summary>
public class ModuleJoinerConfig
{
    public string ServiceName { get; set; } = string.Empty;
    public string[] PrimaryKeys { get; set; } = Array.Empty<string>();
    public Dictionary<string, string[]> LinkableKeys { get; set; } = new();
    public ModuleJoinerRelationship[] Relationships { get; set; } = Array.Empty<ModuleJoinerRelationship>();
}

public class ModuleJoinerRelationship
{
    public string ServiceName { get; set; } = string.Empty;
    public string PrimaryKey { get; set; } = string.Empty;
    public string ForeignKey { get; set; } = string.Empty;
    public bool HasMany { get; set; }
    public bool DeleteCascade { get; set; }
}
```

---

## Step 2: Module Registry (2-3 hours)

### 2.1 Static Module Registry

```csharp
// src/Medusa.Core/Medusa.Core.Modules/MedusaModule.cs

using System.Collections.Concurrent;

namespace Medusa.Core.Modules;

/// <summary>
/// Static registry for all modules (like Medusa's MedusaModule)
/// </summary>
public static class MedusaModule
{
    // Module instances by key
    private static readonly ConcurrentDictionary<string, IModuleService> _instances = new();

    // Module definitions by key
    private static readonly ConcurrentDictionary<string, ModuleDefinition> _definitions = new();

    // Module resolutions by key
    private static readonly ConcurrentDictionary<string, ModuleResolution> _resolutions = new();

    // Joiner configs by service name
    private static readonly ConcurrentDictionary<string, ModuleJoinerConfig> _joinerConfigs = new();

    /// <summary>
    /// Register a module definition
    /// </summary>
    public static void RegisterDefinition(ModuleDefinition definition)
    {
        if (!_definitions.TryAdd(definition.Key, definition))
        {
            throw new InvalidOperationException($"Module '{definition.Key}' already registered");
        }
    }

    /// <summary>
    /// Register a module resolution
    /// </summary>
    public static void RegisterResolution(string key, ModuleResolution resolution)
    {
        _resolutions[key] = resolution;
    }

    /// <summary>
    /// Register a module instance
    /// </summary>
    public static void RegisterInstance(string key, IModuleService instance)
    {
        _instances[key] = instance;

        // Also register joiner config if available
        if (instance.JoinerConfig != null)
        {
            _joinerConfigs[instance.JoinerConfig.ServiceName] = instance.JoinerConfig;
        }
    }

    /// <summary>
    /// Get module instance
    /// </summary>
    public static IModuleService? GetInstance(string key)
    {
        return _instances.TryGetValue(key, out var instance) ? instance : null;
    }

    /// <summary>
    /// Get all module instances
    /// </summary>
    public static IEnumerable<IModuleService> GetAllInstances()
    {
        return _instances.Values;
    }

    /// <summary>
    /// Get module definition
    /// </summary>
    public static ModuleDefinition? GetDefinition(string key)
    {
        return _definitions.TryGetValue(key, out var definition) ? definition : null;
    }

    /// <summary>
    /// Get all definitions
    /// </summary>
    public static IEnumerable<ModuleDefinition> GetAllDefinitions()
    {
        return _definitions.Values;
    }

    /// <summary>
    /// Get joiner config by service name
    /// </summary>
    public static ModuleJoinerConfig? GetJoinerConfig(string serviceName)
    {
        return _joinerConfigs.TryGetValue(serviceName, out var config) ? config : null;
    }

    /// <summary>
    /// Get all joiner configs
    /// </summary>
    public static IEnumerable<ModuleJoinerConfig> GetAllJoinerConfigs()
    {
        return _joinerConfigs.Values;
    }

    /// <summary>
    /// Clear all registrations (for testing)
    /// </summary>
    public static void Clear()
    {
        _instances.Clear();
        _definitions.Clear();
        _resolutions.Clear();
        _joinerConfigs.Clear();
    }
}
```

---

## Step 3: Module Loader (3-4 hours)

### 3.1 Module Loader

```csharp
// src/Medusa.Core/Medusa.Core.Modules/ModuleLoader.cs

using System.Reflection;
using Medusa.Core.DependencyInjection;
using Microsoft.Extensions.Logging;

namespace Medusa.Core.Modules;

/// <summary>
/// Loads modules into the DI container
/// </summary>
public class ModuleLoader
{
    private readonly ILogger<ModuleLoader> _logger;

    public ModuleLoader(ILogger<ModuleLoader> logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// Load all module resolutions into the container
    /// </summary>
    public async Task LoadModulesAsync(
        IServiceContainer container,
        IEnumerable<ModuleResolution> resolutions)
    {
        var resolutionList = resolutions.ToList();

        _logger.LogInformation("Loading {Count} modules", resolutionList.Count);

        // Sort by dependencies (topological sort)
        var sorted = TopologicalSort(resolutionList);

        foreach (var resolution in sorted)
        {
            await LoadModuleAsync(container, resolution);
        }

        _logger.LogInformation("All modules loaded successfully");
    }

    private async Task LoadModuleAsync(IServiceContainer container, ModuleResolution resolution)
    {
        var key = resolution.Definition.Key;
        _logger.LogInformation("Loading module: {Key}", key);

        try
        {
            // 1. Load assembly
            var assembly = LoadAssembly(resolution.ResolutionPath);

            // 2. Find module type (must implement IModuleService)
            var moduleType = FindModuleType(assembly, key);

            // 3. Resolve dependencies
            var dependencies = ResolveDependencies(container, resolution.Dependencies);

            // 4. Create module instance
            var instance = CreateModuleInstance(moduleType, dependencies, resolution);

            // 5. Register in container
            container.Register(key, _ => instance);

            // 6. Register in static registry
            MedusaModule.RegisterInstance(key, instance);

            _logger.LogInformation("Module '{Key}' loaded successfully", key);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to load module '{Key}'", key);

            if (resolution.Definition.IsRequired)
            {
                throw new InvalidOperationException($"Required module '{key}' failed to load", ex);
            }
        }
    }

    private Assembly LoadAssembly(string? path)
    {
        if (string.IsNullOrEmpty(path))
        {
            throw new ArgumentException("Module resolution path is null or empty");
        }

        return Assembly.LoadFrom(path);
    }

    private Type FindModuleType(Assembly assembly, string moduleKey)
    {
        var moduleTypes = assembly.GetTypes()
            .Where(t => typeof(IModuleService).IsAssignableFrom(t) && !t.IsInterface && !t.IsAbstract)
            .ToList();

        if (moduleTypes.Count == 0)
        {
            throw new InvalidOperationException($"No IModuleService implementation found in module '{moduleKey}'");
        }

        if (moduleTypes.Count > 1)
        {
            _logger.LogWarning("Multiple IModuleService implementations found in module '{Key}', using first", moduleKey);
        }

        return moduleTypes.First();
    }

    private Dictionary<string, object> ResolveDependencies(IServiceContainer container, string[] dependencies)
    {
        var resolved = new Dictionary<string, object>();

        foreach (var dep in dependencies)
        {
            try
            {
                var service = container.Resolve<object>(dep);
                resolved[dep] = service;
            }
            catch (Exception ex)
            {
                throw new InvalidOperationException($"Failed to resolve dependency '{dep}'", ex);
            }
        }

        return resolved;
    }

    private IModuleService CreateModuleInstance(
        Type moduleType,
        Dictionary<string, object> dependencies,
        ModuleResolution resolution)
    {
        // Find constructor that takes dependencies and declaration
        var constructor = moduleType.GetConstructors()
            .FirstOrDefault(c =>
            {
                var parameters = c.GetParameters();
                return parameters.Length == 2 &&
                       parameters[0].ParameterType == typeof(Dictionary<string, object>) &&
                       parameters[1].ParameterType == typeof(ModuleDeclaration);
            });

        if (constructor == null)
        {
            throw new InvalidOperationException(
                $"Module '{moduleType.Name}' must have constructor: " +
                $"(Dictionary<string, object> dependencies, ModuleDeclaration declaration)");
        }

        var instance = constructor.Invoke(new object[] { dependencies, resolution.Declaration });

        return (IModuleService)instance;
    }

    private List<ModuleResolution> TopologicalSort(List<ModuleResolution> resolutions)
    {
        var sorted = new List<ModuleResolution>();
        var visited = new HashSet<string>();
        var visiting = new HashSet<string>();

        void Visit(ModuleResolution resolution)
        {
            if (visited.Contains(resolution.Definition.Key))
                return;

            if (visiting.Contains(resolution.Definition.Key))
            {
                throw new InvalidOperationException(
                    $"Circular dependency detected for module '{resolution.Definition.Key}'");
            }

            visiting.Add(resolution.Definition.Key);

            foreach (var dep in resolution.Dependencies)
            {
                var depResolution = resolutions.FirstOrDefault(r => r.Definition.Key == dep);
                if (depResolution != null)
                {
                    Visit(depResolution);
                }
            }

            visiting.Remove(resolution.Definition.Key);
            visited.Add(resolution.Definition.Key);
            sorted.Add(resolution);
        }

        foreach (var resolution in resolutions)
        {
            Visit(resolution);
        }

        return sorted;
    }
}
```

---

## Step 4: Module Discovery (2-3 hours)

### 4.1 Module Discovery Service

```csharp
// src/Medusa.Core/Medusa.Core.Modules/ModuleDiscovery.cs

using System.Reflection;
using Microsoft.Extensions.Logging;

namespace Medusa.Core.Modules;

/// <summary>
/// Discovers modules from assemblies
/// </summary>
public class ModuleDiscovery
{
    private readonly ILogger<ModuleDiscovery> _logger;

    public ModuleDiscovery(ILogger<ModuleDiscovery> logger)
    {
        _logger = logger;
    }

    /// <summary>
    /// Scan assemblies matching pattern (e.g., "Medusa.*.Module.dll")
    /// </summary>
    public IEnumerable<ModuleResolution> ScanAssemblies(string pattern)
    {
        _logger.LogInformation("Scanning for modules with pattern: {Pattern}", pattern);

        var baseDirectory = AppDomain.CurrentDomain.BaseDirectory;
        var files = Directory.GetFiles(baseDirectory, pattern, SearchOption.AllDirectories);

        _logger.LogInformation("Found {Count} potential module assemblies", files.Length);

        var resolutions = new List<ModuleResolution>();

        foreach (var file in files)
        {
            try
            {
                var assembly = Assembly.LoadFrom(file);
                var resolution = DiscoverModuleFromAssembly(assembly, file);

                if (resolution != null)
                {
                    resolutions.Add(resolution);
                    _logger.LogInformation("Discovered module: {Key} from {File}", resolution.Definition.Key, file);
                }
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to load assembly: {File}", file);
            }
        }

        return resolutions;
    }

    private ModuleResolution? DiscoverModuleFromAssembly(Assembly assembly, string filePath)
    {
        // Find types implementing IModuleService
        var moduleTypes = assembly.GetTypes()
            .Where(t => typeof(IModuleService).IsAssignableFrom(t) && !t.IsInterface && !t.IsAbstract)
            .ToList();

        if (moduleTypes.Count == 0)
        {
            return null;
        }

        var moduleType = moduleTypes.First();

        // Create temporary instance to get definition
        var tempInstance = CreateTemporaryInstance(moduleType);

        var definition = tempInstance.Definition;

        // Register definition
        MedusaModule.RegisterDefinition(definition);

        return new ModuleResolution
        {
            ResolutionPath = filePath,
            Definition = definition,
            Dependencies = definition.Dependencies,
            Declaration = new ModuleDeclaration
            {
                Scope = ModuleScope.Internal,
                Options = definition.DefaultOptions
            },
            Options = definition.DefaultOptions
        };
    }

    private IModuleService CreateTemporaryInstance(Type moduleType)
    {
        // Try to create instance with minimal dependencies for discovery
        var constructor = moduleType.GetConstructors().FirstOrDefault();

        if (constructor == null)
        {
            throw new InvalidOperationException($"Module '{moduleType.Name}' has no public constructor");
        }

        var parameters = constructor.GetParameters();
        var args = new object[parameters.Length];

        // Fill with defaults
        for (int i = 0; i < parameters.Length; i++)
        {
            var paramType = parameters[i].ParameterType;

            if (paramType == typeof(Dictionary<string, object>))
            {
                args[i] = new Dictionary<string, object>();
            }
            else if (paramType == typeof(ModuleDeclaration))
            {
                args[i] = new ModuleDeclaration();
            }
            else if (paramType.IsClass)
            {
                args[i] = null!;
            }
            else
            {
                args[i] = Activator.CreateInstance(paramType)!;
            }
        }

        return (IModuleService)constructor.Invoke(args);
    }
}
```

---

## Step 5: Example Infrastructure Modules (4-6 hours)

### 5.1 Logger Module

```csharp
// Create new project
dotnet new classlib -n Medusa.Logger.Module -o src/Modules/Medusa.Logger.Module
dotnet sln add src/Modules/Medusa.Logger.Module

cd src/Modules/Medusa.Logger.Module
dotnet add reference ../../Medusa.Core/Medusa.Core.Modules
dotnet add package Microsoft.Extensions.Logging
```

```csharp
// src/Modules/Medusa.Logger.Module/LoggerModule.cs

using Medusa.Core.Modules;
using Medusa.Core.Types;
using Microsoft.Extensions.Logging;

namespace Medusa.Logger.Module;

public class LoggerModule : IModuleService
{
    private readonly ILogger<LoggerModule> _logger;

    public LoggerModule(
        Dictionary<string, object> dependencies,
        ModuleDeclaration declaration)
    {
        // Logger is provided by ASP.NET Core, not injected as dependency
        var loggerFactory = LoggerFactory.Create(builder => builder.AddConsole());
        _logger = loggerFactory.CreateLogger<LoggerModule>();

        _logger.LogInformation("Logger module initialized");
    }

    public ModuleDefinition Definition => new()
    {
        Key = "logger",
        Label = "Logger",
        DefaultPackage = "Medusa.Logger.Module",
        IsRequired = true,
        IsQueryable = false,
        Dependencies = Array.Empty<string>()
    };

    public ModuleJoinerConfig? JoinerConfig => null;

    public ModuleHooks? Hooks => new()
    {
        OnApplicationStart = async (sp) =>
        {
            _logger.LogInformation("Logger module started");
            await Task.CompletedTask;
        }
    };

    public Task<(object Data, Dictionary<string, string[]> Ids)> SoftDeleteAsync(
        Dictionary<string, string[]> ids,
        Context? context = null)
    {
        throw new NotImplementedException("Logger module does not support soft delete");
    }

    public Task<(object Data, Dictionary<string, string[]> Ids)> RestoreAsync(
        Dictionary<string, string[]> ids,
        Context? context = null)
    {
        throw new NotImplementedException("Logger module does not support restore");
    }
}
```

### 5.2 Cache Module

```csharp
// Create project
dotnet new classlib -n Medusa.Cache.Module -o src/Modules/Medusa.Cache.Module
dotnet sln add src/Modules/Medusa.Cache.Module

cd src/Modules/Medusa.Cache.Module
dotnet add reference ../../Medusa.Core/Medusa.Core.Modules
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// src/Modules/Medusa.Cache.Module/CacheModule.cs

using Medusa.Core.Modules;
using Medusa.Core.Types;
using Microsoft.Extensions.Caching.Distributed;

namespace Medusa.Cache.Module;

public class CacheModule : IModuleService
{
    private readonly IDistributedCache _cache;

    public CacheModule(
        Dictionary<string, object> dependencies,
        ModuleDeclaration declaration)
    {
        // Cache will be injected from host app configuration
        _cache = dependencies.TryGetValue("cache", out var cache)
            ? (IDistributedCache)cache
            : throw new ArgumentException("Cache dependency not found");
    }

    public ModuleDefinition Definition => new()
    {
        Key = "cache",
        Label = "Cache",
        DefaultPackage = "Medusa.Cache.Module",
        IsRequired = false,
        IsQueryable = false,
        Dependencies = new[] { "logger" }
    };

    public ModuleJoinerConfig? JoinerConfig => null;

    public ModuleHooks? Hooks => null;

    public async Task<string?> GetAsync(string key)
    {
        return await _cache.GetStringAsync(key);
    }

    public async Task SetAsync(string key, string value, TimeSpan? expiry = null)
    {
        await _cache.SetStringAsync(key, value, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiry ?? TimeSpan.FromMinutes(5)
        });
    }

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
}
```

---

## Step 6: Service Collection Extensions (2 hours)

### 6.1 Extensions for Easy Registration

```csharp
// src/Medusa.Core/Medusa.Core.Modules/ServiceCollectionExtensions.cs

using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

namespace Medusa.Core.Modules;

public static class ServiceCollectionExtensions
{
    /// <summary>
    /// Add Medusa module system to service collection
    /// </summary>
    public static IServiceCollection AddMedusaModules(
        this IServiceCollection services,
        Action<ModuleOptions> configure)
    {
        var options = new ModuleOptions();
        configure(options);

        // Register module services
        services.AddSingleton<ModuleDiscovery>();
        services.AddSingleton<ModuleLoader>();

        // Discover and register modules
        var sp = services.BuildServiceProvider();
        var discovery = sp.GetRequiredService<ModuleDiscovery>();
        var loader = sp.GetRequiredService<ModuleLoader>();

        var resolutions = new List<ModuleResolution>();

        // Scan assemblies
        if (options.AssemblyPatterns.Any())
        {
            foreach (var pattern in options.AssemblyPatterns)
            {
                resolutions.AddRange(discovery.ScanAssemblies(pattern));
            }
        }

        // Add explicitly registered modules
        resolutions.AddRange(options.ExplicitModules);

        // Create container and load modules
        // (In real implementation, integrate with Autofac)

        return services;
    }
}

public class ModuleOptions
{
    public List<string> AssemblyPatterns { get; } = new();
    public List<ModuleResolution> ExplicitModules { get; } = new();

    public void ScanAssemblies(string pattern)
    {
        AssemblyPatterns.Add(pattern);
    }

    public void AddModule(ModuleResolution resolution)
    {
        ExplicitModules.Add(resolution);
    }
}
```

---

## Step 7: Tests (3-4 hours)

### 7.1 Module Tests

```csharp
// tests/Medusa.Core.Tests/ModuleSystemTests.cs

using FluentAssertions;
using Medusa.Core.Modules;

namespace Medusa.Core.Tests;

public class ModuleSystemTests
{
    [Fact]
    public void Should_Register_Module_Definition()
    {
        MedusaModule.Clear();

        var definition = new ModuleDefinition
        {
            Key = "test-module",
            Label = "Test Module",
            IsRequired = true
        };

        MedusaModule.RegisterDefinition(definition);

        var retrieved = MedusaModule.GetDefinition("test-module");
        retrieved.Should().NotBeNull();
        retrieved!.Label.Should().Be("Test Module");
    }

    [Fact]
    public void Should_Topologically_Sort_Modules_By_Dependencies()
    {
        var resolutions = new List<ModuleResolution>
        {
            new()
            {
                Definition = new ModuleDefinition { Key = "c", Dependencies = new[] { "b" } },
                Dependencies = new[] { "b" },
                Declaration = new ModuleDeclaration()
            },
            new()
            {
                Definition = new ModuleDefinition { Key = "b", Dependencies = new[] { "a" } },
                Dependencies = new[] { "a" },
                Declaration = new ModuleDeclaration()
            },
            new()
            {
                Definition = new ModuleDefinition { Key = "a", Dependencies = Array.Empty<string>() },
                Dependencies = Array.Empty<string>(),
                Declaration = new ModuleDeclaration()
            }
        };

        var loader = new ModuleLoader(new TestLogger<ModuleLoader>());
        var sorted = InvokePrivateMethod<List<ModuleResolution>>(loader, "TopologicalSort", resolutions);

        sorted[0].Definition.Key.Should().Be("a");
        sorted[1].Definition.Key.Should().Be("b");
        sorted[2].Definition.Key.Should().Be("c");
    }

    private T InvokePrivateMethod<T>(object obj, string methodName, params object[] parameters)
    {
        var method = obj.GetType().GetMethod(methodName,
            System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance);

        return (T)method!.Invoke(obj, parameters)!;
    }
}
```

---

## Step 8: Integration Test (1-2 hours)

### 8.1 End-to-End Module Loading

```csharp
// tests/Medusa.IntegrationTests/ModuleLoadingTests.cs

using FluentAssertions;
using Medusa.Core.Modules;
using Medusa.Logger.Module;

namespace Medusa.IntegrationTests;

public class ModuleLoadingTests
{
    [Fact]
    public async Task Should_Load_Logger_Module()
    {
        MedusaModule.Clear();

        // Register definition
        var definition = new ModuleDefinition
        {
            Key = "logger",
            Label = "Logger",
            IsRequired = true,
            Dependencies = Array.Empty<string>()
        };

        MedusaModule.RegisterDefinition(definition);

        // Create instance manually for test
        var instance = new LoggerModule(new Dictionary<string, object>(), new ModuleDeclaration());

        // Register instance
        MedusaModule.RegisterInstance("logger", instance);

        // Verify
        var retrieved = MedusaModule.GetInstance("logger");
        retrieved.Should().NotBeNull();
        retrieved!.Definition.Key.Should().Be("logger");
    }
}
```

---

## Success Criteria

- ✅ Module definitions can be registered
- ✅ Module loader can discover assemblies
- ✅ Modules are loaded in dependency order
- ✅ Logger and Cache modules work
- ✅ Module hooks execute on startup/shutdown
- ✅ Static registry tracks all modules

---

## Common Issues

### Issue: Circular dependency
```
Error: Circular dependency detected for module 'moduleA'
```
Fix: Review module dependencies, remove circular references

### Issue: Module not found during scan
Make sure assembly naming matches pattern:
```csharp
options.ScanAssemblies("Medusa.*.Module.dll");
```

### Issue: Constructor not found
Module must have this constructor:
```csharp
public MyModule(Dictionary<string, object> dependencies, ModuleDeclaration declaration)
```

---

## Next Steps

Once Phase 2 is complete:
1. Build and test Logger module
2. Verify module discovery works
3. Test dependency resolution
4. Move to [Phase 3: Event Bus](./PHASE_3_EVENT_BUS.md)

---

## Estimated Time

- Module types: 2-3 hours
- Registry: 2-3 hours
- Loader: 3-4 hours
- Discovery: 2-3 hours
- Example modules: 4-6 hours
- Extensions: 2 hours
- Tests: 4-6 hours

**Total: 19-27 hours (2-3 weeks part-time)**
