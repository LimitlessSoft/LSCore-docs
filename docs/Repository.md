---
layout: default
title: Repository
nav_order: 5
---

# LSCore Repository

The LSCore Repository module provides a lightweight repository pattern implementation built on top of Entity Framework Core. It offers base classes and interfaces for entities, repositories, entity mappings, and database contexts, giving you a consistent data access layer with built-in soft-delete support and automatic audit fields.

## NuGet Packages

| Package | Description | Use When |
|---------|-------------|----------|
| `LSCore.Repository.Contracts` | Interfaces and base entity class only (`ILSCoreEntity`, `ILSCoreEntityBase`, `LSCoreEntity`, `ILSCoreRepositoryBase`, `ILSCoreEntityMap`, `ILSCoreDbContext`) | You are defining entities or repository interfaces in a project that should not depend on EF Core implementations |
| `LSCore.Repository` | Concrete implementations (`LSCoreRepositoryBase`, `LSCoreEntityMap`, `LSCoreDbContext`, `LSCoreExtensions`) | You are building the data access layer and need the full repository, mapping, and DbContext implementations |

`LSCore.Repository` depends on `LSCore.Repository.Contracts` and `LSCore.Exceptions`, so you only need to reference `LSCore.Repository` directly in your data access project. Reference `LSCore.Repository.Contracts` alone in projects that only need the interfaces (for example, a service layer or domain project).

Both packages target **.NET 9.0**.

---

## Entity Interfaces and Base Class

### ILSCoreEntityBase

The minimal entity interface. It requires only a `long Id` property.

```csharp
public interface ILSCoreEntityBase
{
    long Id { get; set; }
}
```

Use this when you need a bare-minimum entity contract without audit fields.

### ILSCoreEntity

Extends `ILSCoreEntityBase` with audit and soft-delete fields.

```csharp
public interface ILSCoreEntity : ILSCoreEntityBase
{
    new long Id { get; set; }
    bool IsActive { get; set; }
    DateTime CreatedAt { get; set; }
    long CreatedBy { get; set; }
    long? UpdatedBy { get; set; }
    DateTime? UpdatedAt { get; set; }
}
```

| Property | Type | Purpose |
|----------|------|---------|
| `Id` | `long` | Primary key |
| `IsActive` | `bool` | Soft-delete flag. `true` means the record is active |
| `CreatedAt` | `DateTime` | Set automatically on insert (UTC) |
| `CreatedBy` | `long` | ID of the user who created the record. Set automatically on insert when a current user provider is registered |
| `UpdatedBy` | `long?` | ID of the user who last updated the record. Set automatically on update and soft delete when a current user provider is registered |
| `UpdatedAt` | `DateTime?` | Set automatically on update (UTC) |

### LSCoreEntity

The concrete base class that implements `ILSCoreEntity`. **All entities used with `LSCoreRepositoryBase` must inherit from this class.**

```csharp
public class LSCoreEntity : ILSCoreEntity
{
    public long Id { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
    public long CreatedBy { get; set; }
    public long? UpdatedBy { get; set; }
    public DateTime? UpdatedAt { get; set; }
}
```

#### Creating an Entity

```csharp
using LSCore.Repository.Contracts;

public class Product : LSCoreEntity
{
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string? Description { get; set; }
}
```

You do not need to declare `Id`, `IsActive`, `CreatedAt`, `CreatedBy`, `UpdatedBy`, or `UpdatedAt` -- they are inherited from `LSCoreEntity`.

---

## Repository Interface and Base Class

### ILSCoreRepositoryBase\<TEntity\>

Defines the standard CRUD operations for any entity.

```csharp
public interface ILSCoreRepositoryBase<TEntity>
    where TEntity : LSCoreEntity
{
    TEntity Get(long id);
    TEntity? GetOrDefault(long id);
    IQueryable<TEntity> GetMultiple();
    void Insert(TEntity entity);
    void Insert(IEnumerable<TEntity> entities);
    void Update(TEntity entity);
    void Update(IEnumerable<TEntity> entities);
    void UpdateOrInsert(TEntity entity);
    void SoftDelete(long id);
    void HardDelete(long id);
    void SoftDelete(TEntity entity);
    void HardDelete(TEntity entity);
    void SoftDelete(IEnumerable<long> ids);
    void HardDelete(IEnumerable<long> ids);
    void SoftDelete(IEnumerable<TEntity> entities);
    void HardDelete(IEnumerable<TEntity> entities);
}
```

| Method | Behavior |
|--------|----------|
| `Get(id)` | Returns the entity or throws `LSCoreNotFoundException` if not found or inactive |
| `GetOrDefault(id)` | Returns the entity or `null`. Only returns active records (`IsActive == true`) |
| `GetMultiple()` | Returns an `IQueryable<TEntity>` filtered to active records only |
| `Insert(entity)` / `Insert(entities)` | Sets `CreatedAt` to the current UTC time, sets `IsActive` to `true`, sets `CreatedBy` if a current user is available, then saves |
| `Update(entity)` / `Update(entities)` | Sets `UpdatedAt` to the current UTC time, sets `UpdatedBy` if a current user is available, then saves |
| `UpdateOrInsert(entity)` | Calls `Insert` if `Id == 0`, otherwise calls `Update` |
| `SoftDelete(...)` | Sets `IsActive = false`, `UpdatedAt` to the current UTC time, and `UpdatedBy` if a current user is available, then saves |
| `HardDelete(...)` | Permanently removes the record(s) from the database |

### LSCoreRepositoryBase\<TEntity\>

The concrete implementation. It takes an `ILSCoreDbContext` through its primary constructor, plus two optional dependencies.

```csharp
public class LSCoreRepositoryBase<TEntity>(
    ILSCoreDbContext dbContext,
    ILSCoreCurrentUserProvider? currentUserProvider = null,
    TimeProvider? timeProvider = null
) : ILSCoreRepositoryBase<TEntity>
    where TEntity : LSCoreEntity
```

| Parameter | Purpose |
|-----------|---------|
| `dbContext` | The context the entity set is read from and written to |
| `currentUserProvider` | Supplies the user stamped on `CreatedBy` and `UpdatedBy`. Omit it and those fields are left to you |
| `timeProvider` | Source of the audit timestamps. Defaults to `TimeProvider.System`; pass a fake one to make timestamps assertable in tests |

All methods are `virtual`, so you can override any of them in a derived repository. The two seams are also available as protected members -- `UtcNow` and `CurrentUserId` -- which you can override instead of injecting.

#### Filling CreatedBy and UpdatedBy

`ILSCoreCurrentUserProvider` is how the repository learns who is writing:

```csharp
public interface ILSCoreCurrentUserProvider
{
    long? CurrentUserId { get; }
}
```

Implement it over whatever your application uses for the current user. With `LSCore.Auth`, that is the scoped `LSCoreAuthContextEntity<long>`:

```csharp
public class CurrentUserProvider(LSCoreAuthContextEntity<long> authContext)
    : ILSCoreCurrentUserProvider
{
    public long? CurrentUserId => authContext.IsAuthenticated ? authContext.Identifier : null;
}
```

Register it and take it in your repository's constructor:

```csharp
builder.Services.AddScoped<ILSCoreCurrentUserProvider, CurrentUserProvider>();

public class ProductRepository(ILSCoreDbContext dbContext, ILSCoreCurrentUserProvider currentUser)
    : LSCoreRepositoryBase<Product>(dbContext, currentUser), IProductRepository { }
```

From then on, `Insert` fills `CreatedBy`, and `Update` and `SoftDelete` fill `UpdatedBy`. When `CurrentUserId` is `null` -- an unauthenticated request, a background job -- the fields are left untouched. A `CreatedBy` you set yourself before calling `Insert` is never overwritten, so you can still record an insert made on behalf of another user.

#### Creating a Repository

For basic CRUD with no extra logic, you can use `LSCoreRepositoryBase` directly:

```csharp
using LSCore.Repository;
using LSCore.Repository.Contracts;

public interface IProductRepository : ILSCoreRepositoryBase<Product> { }

public class ProductRepository(ILSCoreDbContext dbContext)
    : LSCoreRepositoryBase<Product>(dbContext), IProductRepository { }
```

#### Adding Custom Methods

```csharp
public interface IProductRepository : ILSCoreRepositoryBase<Product>
{
    IEnumerable<Product> GetByPriceRange(decimal min, decimal max);
}

public class ProductRepository(ILSCoreDbContext dbContext)
    : LSCoreRepositoryBase<Product>(dbContext), IProductRepository
{
    public IEnumerable<Product> GetByPriceRange(decimal min, decimal max)
    {
        return GetMultiple()
            .Where(p => p.Price >= min && p.Price <= max)
            .ToList();
    }
}
```

#### Overriding Default Behavior

```csharp
public class ProductRepository(ILSCoreDbContext dbContext)
    : LSCoreRepositoryBase<Product>(dbContext), IProductRepository
{
    public override void Insert(IEnumerable<Product> entities)
    {
        // Custom validation before inserting
        foreach (var product in entities)
        {
            if (product.Price < 0)
                throw new ArgumentException("Price cannot be negative.");
        }

        base.Insert(entities);
    }
}
```

---

## Entity Mapping

### ILSCoreEntityMap\<TEntity\>

Defines the contract for EF Core entity configuration.

```csharp
public interface ILSCoreEntityMap<TEntity>
    where TEntity : class
{
    Action<EntityTypeBuilder<TEntity>> Mapper { get; }
    EntityTypeBuilder<TEntity> Map(EntityTypeBuilder<TEntity> entityTypeBuilder);
}
```

### LSCoreEntityMap\<TEntity\>

Abstract base class that automatically maps the standard `LSCoreEntity` fields (`Id` as primary key, `CreatedAt`, `CreatedBy` and `IsActive` as required, `UpdatedAt` and `UpdatedBy` as optional). You provide entity-specific mapping through the `Mapper` property.

```csharp
public abstract class LSCoreEntityMap<TEntity> : ILSCoreEntityMap<TEntity>
    where TEntity : class, ILSCoreEntity
```

Default field mappings applied automatically:

| Field | Mapping |
|-------|---------|
| `Id` | Primary key (`HasKey`) |
| `CreatedAt` | Required |
| `CreatedBy` | Required |
| `IsActive` | Required |
| `UpdatedAt` | Optional |
| `UpdatedBy` | Optional |

#### Creating an Entity Map

```csharp
using LSCore.Repository;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class ProductMap : LSCoreEntityMap<Product>
{
    public override Action<EntityTypeBuilder<Product>> Mapper => builder =>
    {
        builder.ToTable("Products");

        builder.Property(x => x.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(x => x.Price)
            .IsRequired()
            .HasColumnType("decimal(18,2)");

        builder.Property(x => x.Description)
            .HasMaxLength(1000);
    };
}
```

You do not need to map `Id`, `CreatedAt`, `CreatedBy`, `IsActive`, `UpdatedAt`, or `UpdatedBy` -- they are handled by the base class.

#### Suppressing Default Mapping

If you need full control over all field mappings, pass `true` to the base constructor:

```csharp
public class ProductMap : LSCoreEntityMap<Product>
{
    public ProductMap() : base(suppressDefaultMapping: true) { }

    public override Action<EntityTypeBuilder<Product>> Mapper => builder =>
    {
        // You must now map all fields yourself, including Id, IsActive, etc.
        builder.ToTable("Products");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Name).IsRequired();
        // ... all other fields
    };
}
```

---

## DbContext

### ILSCoreDbContext

A minimal interface exposing `Set<T>()` and `SaveChanges()`. This is what `LSCoreRepositoryBase` depends on.

```csharp
public interface ILSCoreDbContext
{
    DbSet<T> Set<T>() where T : class;
    int SaveChanges();
}
```

### LSCoreDbContext\<TContext\>

Abstract base class that extends EF Core's `DbContext` and implements `ILSCoreDbContext`.

```csharp
public abstract class LSCoreDbContext<TContext>(DbContextOptions<TContext> options)
    : DbContext(options), ILSCoreDbContext
    where TContext : DbContext;
```

#### Creating a DbContext

```csharp
using LSCore.Repository;
using Microsoft.EntityFrameworkCore;

public class AppDbContext(DbContextOptions<AppDbContext> options)
    : LSCoreDbContext<AppDbContext>(options)
{
    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Product>(new ProductMap().Map);
    }
}
```

---

## Extension Methods

### AddMap

The `AddMap` extension method provides a fluent way to apply entity maps within `OnModelCreating`.

```csharp
public static EntityTypeBuilder<TEntity> AddMap<TEntity>(
    this EntityTypeBuilder<TEntity> entityTypeBuilder,
    ILSCoreEntityMap<TEntity> map
) where TEntity : class => map.Map(entityTypeBuilder);
```

Using `AddMap` as an alternative syntax:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<Product>().AddMap(new ProductMap());
}
```

Both approaches -- `modelBuilder.Entity<Product>(new ProductMap().Map)` and `modelBuilder.Entity<Product>().AddMap(new ProductMap())` -- are equivalent.

---

## Complete Example

Below is a full working example showing how all the pieces fit together.

### 1. Define the Entity

```csharp
using LSCore.Repository.Contracts;

public class Order : LSCoreEntity
{
    public string OrderNumber { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public string CustomerName { get; set; } = string.Empty;
}
```

### 2. Define the Entity Map

```csharp
using LSCore.Repository;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class OrderMap : LSCoreEntityMap<Order>
{
    public override Action<EntityTypeBuilder<Order>> Mapper => builder =>
    {
        builder.ToTable("Orders");

        builder.Property(x => x.OrderNumber)
            .IsRequired()
            .HasMaxLength(50);

        builder.Property(x => x.TotalAmount)
            .IsRequired()
            .HasColumnType("decimal(18,2)");

        builder.Property(x => x.CustomerName)
            .IsRequired()
            .HasMaxLength(200);
    };
}
```

### 3. Define the Repository Interface and Implementation

```csharp
using LSCore.Repository;
using LSCore.Repository.Contracts;

public interface IOrderRepository : ILSCoreRepositoryBase<Order>
{
    IEnumerable<Order> GetByCustomer(string customerName);
}

public class OrderRepository(ILSCoreDbContext dbContext)
    : LSCoreRepositoryBase<Order>(dbContext), IOrderRepository
{
    public IEnumerable<Order> GetByCustomer(string customerName)
    {
        return GetMultiple()
            .Where(o => o.CustomerName == customerName)
            .ToList();
    }
}
```

### 4. Define the DbContext

```csharp
using LSCore.Repository;
using Microsoft.EntityFrameworkCore;

public class AppDbContext(DbContextOptions<AppDbContext> options)
    : LSCoreDbContext<AppDbContext>(options)
{
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Order>().AddMap(new OrderMap());
    }
}
```

### 5. Register in Dependency Injection

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped<ILSCoreDbContext>(sp => sp.GetRequiredService<AppDbContext>());
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
```

### 6. Use in a Service or Controller

```csharp
public class OrderService(IOrderRepository orderRepository)
{
    public Order CreateOrder(string orderNumber, decimal total, string customer, long userId)
    {
        var order = new Order
        {
            OrderNumber = orderNumber,
            TotalAmount = total,
            CustomerName = customer,
            CreatedBy = userId
        };

        orderRepository.Insert(order);
        return order;
    }

    public Order GetOrder(long id) => orderRepository.Get(id);

    public void CancelOrder(long id) => orderRepository.SoftDelete(id);
}
```

---

## Key Behaviors

- **All reads filter by `IsActive`**: `Get`, `GetOrDefault`, and `GetMultiple` only return records where `IsActive == true`. Soft-deleted records are invisible to standard queries.
- **Insert sets audit fields automatically**: `CreatedAt` is set to the current UTC time and `IsActive` is set to `true` on every insert. `CreatedBy` is filled too when a current user provider is registered, unless you set it yourself first.
- **Update and soft delete set audit fields automatically**: Every update sets `UpdatedAt` to the current UTC time, and `UpdatedBy` when a current user provider is registered.
- **SaveChanges is called within each operation**: Each `Insert`, `Update`, `SoftDelete`, and `HardDelete` call triggers `SaveChanges()` immediately.
- **`Get` throws on missing records**: `Get(id)` throws `LSCoreNotFoundException` (from `LSCore.Exceptions`) if the record does not exist or is inactive. Use `GetOrDefault(id)` for a null-returning alternative.
- **`UpdateOrInsert` checks `Id == 0`**: If the entity's `Id` is `0`, it inserts; otherwise, it updates.
