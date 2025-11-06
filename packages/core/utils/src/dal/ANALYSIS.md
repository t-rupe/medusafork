# Database Layer Analysis - MedusaJS

## Purpose
PostgreSQL-based data access layer using MikroORM with custom DML (Data Modeling Language) for entity definition, repository pattern, and advanced relationship management.

## Architecture

### Core Components

**1. DML (Data Modeling Language)** (`dml/entity.ts`)
Custom entity definition DSL that generates MikroORM entities:

```typescript
const Product = model.define("product", {
  id: model.id(),
  name: model.text(),
  price: model.bigNumber(),
  variants: model.hasMany(() => Variant)
})
.cascades({ delete: ["variants"] })
.indexes([{ on: ["name"], unique: true }])
```

**Key Features:**
- Type-safe entity schemas
- Automatic ID generation with prefixes (`prod_01H...`)
- Relationship definitions (hasOne, hasMany, belongsTo, manyToMany)
- Cascade configurations
- Index/check constraint declarations

See: `dml/entity.ts`, `dml/entity-builder.ts`

**2. Base Entity** (`mikro-orm/base-entity.ts`)
All entities inherit from `BaseEntity`:
- Primary key with automatic prefix-based ID generation
- Uses `generateEntityId()` with entity name → prefix algorithm
- Lifecycle hooks (`@OnInit`, `@BeforeCreate`)

```typescript
@Entity()
class Product extends BaseEntity {
  // Automatically generates: "prod_01HXXXXXXXXX"
}
```

**3. Repository Pattern** (`mikro-orm/mikro-orm-repository.ts`)

Generic base repository with full CRUD + advanced operations:

```typescript
interface MikroOrmBaseRepository<T> {
  create(data: T[], context?: Context): Promise<T[]>
  update(data: { entity, update }[], context?: Context): Promise<T[]>
  delete(filters: Where<T>, context?: Context): Promise<string[]>
  find(options: FindOptions<T>, context?: Context): Promise<T[]>
  findAndCount(options, context?): Promise<[T[], number]>
  upsert(data: T[], context?): Promise<T[]>
  upsertWithReplace(data, config, context?): Promise<{entities, performedActions}>
  softDelete(filters, context?): Promise<[T[], Record<string, unknown[]>]>
  restore(filters, context?): Promise<[T[], Record<string, unknown[]>]>
}
```

**Key Methods:**
- **upsert**: Insert or update based on primary key
- **upsertWithReplace**: Advanced upsert with relation diff/sync
- **softDelete**: Sets `deleted_at` timestamp recursively
- **restore**: Clears `deleted_at` recursively

**4. Transaction Management**
Context-aware transaction handling:

```typescript
await repository.transaction(async (transactionManager) => {
  // All operations use this transaction
  await repository.create(data, { transactionManager })
})
```

Manager resolution order: `transactionManager` → `manager` → `fork()`

**5. MikroORM Configuration** (`mikro-orm/mikro-orm-create-connection.ts`)
PostgreSQL connection with:
- Connection pooling (min: 2, configurable max)
- Schema support (default: "public")
- Batch inserts/updates enabled
- Select-in strategy for pagination
- Custom migration generator
- Global filters (soft delete, etc.)

```typescript
const orm = await mikroOrmCreateConnection({
  clientUrl: "postgresql://...",
  schema: "public",
  pool: { min: 2, max: 10 },
  filters: {
    softDelete: { cond: { deleted_at: null } }
  }
}, entities, pathToMigrations)
```

**6. UpsertWithReplace** (Complex Operation)
Intelligent relation synchronization:

1. Upsert base entities
2. For each relation:
   - **hasMany/manyToMany**: Diff existing vs new
   - Create missing entries
   - Update existing entries (by ID)
   - Delete omitted entries
3. Return full object graph + performedActions tracking

```typescript
const { entities, performedActions } = await repository.upsertWithReplace([{
  id: "prod_1",
  name: "Updated Product",
  variants: [
    { id: "var_1", title: "Small" },  // Update
    { title: "Medium" }                // Create
    // "var_2" was omitted → will be deleted
  ]
}], { relations: ["variants"] })

// performedActions:
// {
//   created: { Variant: [{ id: "var_3", ... }] },
//   updated: { Product: [...], Variant: [{ id: "var_1" }] },
//   deleted: { Variant: [{ id: "var_2" }] }
// }
```

**7. Relation Handling**
Supports all relation types:
- **belongsTo**: Many-to-one (foreign key required)
- **hasOne**: One-to-one with FK on target
- **hasOneWithFK**: One-to-one with FK on owner
- **hasMany**: One-to-many
- **manyToMany**: Pivot table management

Cascade deletes only allowed parent → child.

See: `dml/relations/`, `mikro-orm-repository.ts` lines 774-989

**8. Entity Builder** (`dml/entity-builder.ts`)
Transforms DML entities to MikroORM entities:
- Applies decorators (`@Property`, `@OneToMany`, etc.)
- Generates indexes and checks
- Handles big number fields (numeric precision)
- Searchable field configuration

**9. Soft Delete Implementation**
Recursive soft delete across relationships:
```typescript
await repository.softDelete("prod_1")
// Sets deleted_at on product + cascaded variants
// Returns: [deletedEntities, cascadedMap]
```

Uses `mikroOrmUpdateDeletedAtRecursively()` helper.

See: `mikro-orm/utils.ts`

**10. Query Building**
Filter transformation:
```typescript
buildQuery({
  name: { $like: "%test%" },
  price: { $gte: 100 }
}, { withDeleted: false })
```

Supports MikroORM operators: `$eq`, `$ne`, `$in`, `$nin`, `$gt`, `$gte`, `$lt`, `$lte`, `$like`, `$ilike`, `$and`, `$or`

See: `modules-sdk/build-query.ts`

## .NET Mapping with Entity Framework Core

### 1. Replace DML with EF Core Fluent API

**Medusa DML:**
```typescript
const Product = model.define("product", {
  id: model.id(),
  name: model.text(),
  variants: model.hasMany(() => Variant)
})
```

**.NET EF Core:**
```csharp
public class Product
{
    public string Id { get; set; }
    public string Name { get; set; }
    public List<Variant> Variants { get; set; }
}

public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Id).HasDefaultValueSql("generate_id('prod')");

        builder.HasMany(p => p.Variants)
            .WithOne()
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### 2. Repository Pattern

**Medusa:**
```typescript
class ProductRepository extends MikroOrmBaseRepository<Product> {
  async findWithVariants(id: string) {
    return this.find({ where: { id }, options: { populate: ["variants"] }})
  }
}
```

**.NET:**
```csharp
public class ProductRepository : EfCoreRepository<Product>
{
    public async Task<Product?> FindWithVariants(string id, CancellationToken ct)
    {
        return await _context.Products
            .Include(p => p.Variants)
            .FirstOrDefaultAsync(p => p.Id == id, ct);
    }
}

// Generic base
public class EfCoreRepository<T> : IRepository<T> where T : class
{
    protected readonly DbContext _context;
    protected readonly DbSet<T> _dbSet;

    public virtual async Task<List<T>> Find(
        Expression<Func<T, bool>> predicate,
        CancellationToken ct)
    {
        return await _dbSet.Where(predicate).ToListAsync(ct);
    }

    // Implement Create, Update, Delete, Upsert, etc.
}
```

### 3. UpsertWithReplace Pattern

**Medusa:** Custom implementation with relation diffing
**.NET:** Use Change Tracker or custom logic

```csharp
public async Task<Product> UpsertWithRelations(Product product, CancellationToken ct)
{
    var existing = await _context.Products
        .Include(p => p.Variants)
        .FirstOrDefaultAsync(p => p.Id == product.Id, ct);

    if (existing == null)
    {
        _context.Products.Add(product);
    }
    else
    {
        _context.Entry(existing).CurrentValues.SetValues(product);

        // Sync variants (add/update/remove)
        var existingIds = existing.Variants.Select(v => v.Id).ToHashSet();
        var newIds = product.Variants.Select(v => v.Id).ToHashSet();

        // Remove deleted
        var toRemove = existing.Variants.Where(v => !newIds.Contains(v.Id));
        _context.Variants.RemoveRange(toRemove);

        // Add/update
        foreach (var variant in product.Variants)
        {
            var existingVariant = existing.Variants
                .FirstOrDefault(v => v.Id == variant.Id);

            if (existingVariant == null)
                existing.Variants.Add(variant);
            else
                _context.Entry(existingVariant).CurrentValues.SetValues(variant);
        }
    }

    await _context.SaveChangesAsync(ct);
    return product;
}
```

### 4. Soft Delete Implementation

**Medusa:** Recursive `deleted_at` update
**.NET:** Global query filter + interceptor

```csharp
public abstract class SoftDeletableEntity
{
    public DateTime? DeletedAt { get; set; }
    public bool IsDeleted => DeletedAt.HasValue;
}

// DbContext configuration
protected override void OnModelCreating(ModelBuilder builder)
{
    builder.Entity<Product>()
        .HasQueryFilter(p => p.DeletedAt == null);
}

// Soft delete interceptor
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData,
        InterceptionResult<int> result)
    {
        if (eventData.Context is null) return result;

        foreach (var entry in eventData.Context.ChangeTracker.Entries())
        {
            if (entry is not { State: EntityState.Deleted, Entity: SoftDeletableEntity entity })
                continue;

            entry.State = EntityState.Modified;
            entity.DeletedAt = DateTime.UtcNow;
        }

        return result;
    }
}
```

### 5. Transaction Management

**Medusa:**
```typescript
await repository.transaction(async (tm) => {
  await repo.create(data, { transactionManager: tm })
})
```

**.NET:**
```csharp
await using var transaction = await _context.Database.BeginTransactionAsync(ct);
try
{
    await _context.Products.AddAsync(product, ct);
    await _context.SaveChangesAsync(ct);
    await transaction.CommitAsync(ct);
}
catch
{
    await transaction.RollbackAsync(ct);
    throw;
}
```

### 6. ID Generation with Prefixes

**Medusa:** `generateEntityId(undefined, "prod")` → `prod_01HXXXXXXXXX`
**.NET:** Custom ValueGenerator

```csharp
public class PrefixedIdGenerator : ValueGenerator<string>
{
    private readonly string _prefix;

    public PrefixedIdGenerator(string prefix)
    {
        _prefix = prefix;
    }

    public override string Next(EntityEntry entry)
    {
        // Using Ulid or similar
        return $"{_prefix}_{Ulid.NewUlid().ToString()}";
    }

    public override bool GeneratesTemporaryValues => false;
}

// In configuration
builder.Property(p => p.Id)
    .HasValueGenerator<PrefixedIdGenerator>()
    .ValueGeneratedOnAdd();
```

### 7. Migration System

**Medusa:** MikroORM migrations with custom generator
**.NET:** EF Core migrations

```bash
# Generate migration
dotnet ef migrations add AddProduct

# Apply migrations
dotnet ef database update
```

### 8. Connection Pooling

**Medusa:** Knex connection pool (min: 2)
**.NET:** Built-in Npgsql pooling

```csharp
services.AddDbContext<MedusaDbContext>(options =>
    options.UseNpgsql(connectionString, npgsqlOptions =>
    {
        npgsqlOptions.MinBatchSize(2);
        npgsqlOptions.MaxBatchSize(100);
    }));
```

### 9. Batch Operations

**Medusa:** `useBatchInserts: true`, `useBatchUpdates: true`
**.NET:** Use `AddRange` + `SaveChangesAsync` or EFCore.BulkExtensions

```csharp
// Built-in batching
_context.Products.AddRange(products);
await _context.SaveChangesAsync(ct); // Batched

// Or use library
await _context.BulkInsertAsync(products, ct);
await _context.BulkUpdateAsync(products, ct);
```

### 10. Load Strategy

**Medusa:** LoadStrategy.SELECT_IN for pagination
**.NET:** Split queries or filtered includes

```csharp
var products = await _context.Products
    .Include(p => p.Variants)
    .AsSplitQuery() // Similar to SELECT_IN
    .ToListAsync(ct);
```

## Key Differences

| Aspect | MedusaJS | .NET EF Core |
|--------|----------|--------------|
| ORM | MikroORM | Entity Framework Core |
| Entity Definition | DML DSL | C# classes + Fluent API |
| ID Generation | Custom prefix-based | ValueGenerator or triggers |
| Repository | Manual base class | Generic repo or DbSet |
| Soft Delete | Manual `deleted_at` field | Query filters + interceptor |
| Relations | Explicit builders | Navigation properties |
| Migrations | MikroORM CLI | `dotnet ef` CLI |
| Transaction Ctx | Context parameter | DbContext transaction |
| Upsert | Custom implementation | `Update` or Bulk library |

## Implementation Recommendations

### Project Structure
```
/Data
  /Entities
    Product.cs
    Variant.cs
  /Configurations
    ProductConfiguration.cs
  /Repositories
    IRepository.cs
    EfCoreRepository.cs
    ProductRepository.cs
  MedusaDbContext.cs
```

### Key NuGet Packages
- **Npgsql.EntityFrameworkCore.PostgreSQL** - PostgreSQL provider
- **EFCore.NamingConventions** - Snake_case naming
- **EFCore.BulkExtensions** - Bulk operations
- **Ulid** - ID generation
- **Microsoft.EntityFrameworkCore.Design** - Migrations

### DbContext Setup
```csharp
public class MedusaDbContext : DbContext
{
    public MedusaDbContext(DbContextOptions<MedusaDbContext> options)
        : base(options) { }

    public DbSet<Product> Products { get; set; }
    public DbSet<Variant> Variants { get; set; }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfigurationsFromAssembly(typeof(MedusaDbContext).Assembly);

        // Global soft delete filter
        foreach (var entityType in builder.Model.GetEntityTypes())
        {
            if (typeof(SoftDeletableEntity).IsAssignableFrom(entityType.ClrType))
            {
                builder.Entity(entityType.ClrType)
                    .HasQueryFilter(
                        Expression.Lambda(
                            Expression.Equal(
                                Expression.Property(
                                    Expression.Parameter(entityType.ClrType, "e"),
                                    nameof(SoftDeletableEntity.DeletedAt)),
                                Expression.Constant(null)),
                            Expression.Parameter(entityType.ClrType, "e")));
            }
        }
    }
}
```

### Generic Repository with VSA Pattern
```csharp
public interface IRepository<T> where T : class
{
    Task<T?> FindById(string id, CancellationToken ct);
    Task<List<T>> Find(Expression<Func<T, bool>> predicate, CancellationToken ct);
    Task<T> Create(T entity, CancellationToken ct);
    Task<T> Update(T entity, CancellationToken ct);
    Task Delete(string id, CancellationToken ct);
    Task<T> Upsert(T entity, CancellationToken ct);
}
```

## References

**Core Files:**
- `utils/src/dml/entity.ts` - DML entity definition
- `utils/src/dal/mikro-orm/base-entity.ts` - Base entity class
- `utils/src/dal/mikro-orm/mikro-orm-repository.ts` - Repository implementation
- `utils/src/dal/mikro-orm/mikro-orm-create-connection.ts` - ORM setup
- `utils/src/modules-sdk/build-query.ts` - Query builder
- `utils/src/dml/entity-builder.ts` - DML → MikroORM transformation
