---
layout: default
title: Mapper
nav_order: 8
---

# LSCore Mapper

The LSCore Mapper module provides a lightweight, convention-based object mapping system for transforming one class into another. Instead of relying on reflection-heavy or configuration-heavy mapping libraries, LSCore Mapper lets you write explicit mapping logic in small, focused classes that are automatically discovered, registered with the DI container, and invoked through convenient extension methods.

## NuGet Packages

| Package | Description |
|---|---|
| `LSCore.Mapper.Contracts` | Contains the `ILSCoreMapper<TSource, TDestination>` interface. No dependencies beyond .NET 9.0. Reference this package from any layer that needs to define mapper implementations. |
| `LSCore.Mapper.Domain` | Contains the `ToMapped` and `ToMappedList` extension methods that resolve mappers at runtime. Depends on `LSCore.Mapper.Contracts` and `LSCore.DependencyInjection`. Reference this package from layers that need to invoke mapping. |

Both packages target **.NET 9.0** (version 9.1.4.1 at time of writing).

## The Mapper Interface

All mappers implement a single generic interface defined in `LSCore.Mapper.Contracts`:

```csharp
namespace LSCore.Mapper.Contracts;

public interface ILSCoreMapper<in TSource, out TDestination>
    where TSource : class
    where TDestination : class
{
    TDestination ToMapped(TSource source);
}
```

Key points:

- `TSource` is contravariant (`in`) -- the type you are mapping from.
- `TDestination` is covariant (`out`) -- the type you are mapping to.
- Both type parameters are constrained to reference types (`class`).
- Each mapper handles exactly one source-to-destination pair, keeping the mapping logic focused and testable.

## Creating a Mapper

To create a mapper, define a class that implements `ILSCoreMapper<TSource, TDestination>` and write your transformation logic inside the `ToMapped` method.

### Step 1 -- Define your source and destination types

```csharp
// Entity (source)
public class ProductEntity
{
    public long Id { get; set; }
    public string Name { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; }
}

// DTO (destination)
public class ProductDto
{
    public long Id { get; set; }
    public string Name { get; set; }
}
```

### Step 2 -- Implement the mapper

```csharp
using LSCore.Mapper.Contracts;

public class ProductDtoMapper : ILSCoreMapper<ProductEntity, ProductDto>
{
    public ProductDto ToMapped(ProductEntity source) =>
        new() { Id = source.Id, Name = source.Name };
}
```

The mapper is a plain class with no base class or attribute requirements. You write the property assignments yourself, giving you full control over the transformation, including computed properties, conditional logic, or calls to other services (mappers are resolved from the DI container, so constructor injection works).

## DI Registration

LSCore Mapper integrates with the `LSCore.DependencyInjection` module. When you call `AddLSCoreDependencyInjection`, the assembly scanner automatically discovers all classes implementing `ILSCoreMapper<,>` and registers them as **singletons** in the DI container.

### Setup in Program.cs

```csharp
using LSCore.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

// Scans all assemblies whose names start with "MyProject"
// and auto-registers mappers, validators, and services.
builder.AddLSCoreDependencyInjection("MyProject");

var app = builder.Build();

// Required: stores the service provider so extension methods can resolve mappers.
app.UseLSCoreDependencyInjection();

app.MapControllers();
app.Run();
```

**Important:** You must call `app.UseLSCoreDependencyInjection()` after building the application. This stores the `IServiceProvider` in a static container that the `ToMapped` extension methods use at runtime to resolve mapper instances. Without this call, mapping will throw an exception.

### Disabling automatic mapper registration

If you do not use LSCore mappers and want to skip the scanning step, you can disable it through the options:

```csharp
builder.AddLSCoreDependencyInjection("MyProject", options =>
{
    options.Scan.DisableLSCoreDtoMappers();
});
```

## Using the Extension Methods

Once registration is in place, you invoke mapping through two extension methods defined in `LSCore.Mapper.Domain`:

### Mapping a single object

```csharp
using LSCore.Mapper.Domain;

ProductEntity entity = repository.Get(id);
ProductDto dto = entity.ToMapped<ProductEntity, ProductDto>();
```

### Mapping a collection

```csharp
using LSCore.Mapper.Domain;

List<ProductEntity> entities = repository.GetAll();
List<ProductDto> dtos = entities.ToMappedList<ProductEntity, ProductDto>();
```

`ToMappedList` accepts any `IEnumerable<TSource>` and returns a `List<TDestination>`. Internally, it calls `ToMapped` on each element.

### Error handling

The extension methods throw in two cases:

1. **Service provider not initialized.** If `UseLSCoreDependencyInjection()` was not called, `ToMapped` throws an `Exception` with a message instructing you to call `IHost.UseLSCoreDependencyInjection()`.
2. **Mapper not found.** If no `ILSCoreMapper<TSource, TDestination>` is registered for the requested type pair, `ToMapped` throws an `ArgumentNullException` identifying the missing mapper.

## Important: DI Lifetime Caveat

When using Automatic IoC, mappers are registered as **Singletons**. This means you cannot inject Transient or Scoped services into mapper constructors. If your mapper needs data from a Transient or Scoped service, use the Factory pattern: inject a Singleton factory that can resolve the dependency at mapping time.

## Full Example

Below is a complete minimal API project that demonstrates the mapper module end to end.

### Project structure

```
MyProject.Api/
  Program.cs
  Entities/
    ProductEntity.cs
  Dtos/
    ProductDto.cs
  Mappers/
    ProductDtoMapper.cs
  Interfaces/
    IProductManager.cs
    IProductRepository.cs
  Managers/
    ProductManager.cs
  Repositories/
    ProductRepository.cs
  Controllers/
    ProductsController.cs
```

### Program.cs

```csharp
using LSCore.DependencyInjection;
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.AddLSCoreDependencyInjection("MyProject");

var app = builder.Build();
app.UseLSCoreDependencyInjection();
app.UseLSCoreExceptionsHandler();
app.MapControllers();
app.Run();
```

### Mapper

```csharp
using LSCore.Mapper.Contracts;

public class ProductDtoMapper : ILSCoreMapper<ProductEntity, ProductDto>
{
    public ProductDto ToMapped(ProductEntity source) =>
        new() { Id = source.Id, Name = source.Name };
}
```

### Manager (consuming the mapper)

```csharp
using LSCore.Mapper.Domain;

public class ProductManager(IProductRepository productRepository) : IProductManager
{
    public ProductDto Get(int id) =>
        productRepository.Get(id).ToMapped<ProductEntity, ProductDto>();

    public List<ProductDto> GetAll() =>
        productRepository.GetAll().ToMappedList<ProductEntity, ProductDto>();
}
```

### Controller

```csharp
using Microsoft.AspNetCore.Mvc;

public class ProductsController(IProductManager productManager) : ControllerBase
{
    [HttpGet]
    [Route("/products")]
    public IActionResult GetMultiple() => Ok(productManager.GetAll());

    [HttpGet]
    [Route("/products/{id:int}")]
    public IActionResult GetSingle([FromRoute] int id) => Ok(productManager.Get(id));
}
```

## Integration with Other LSCore Modules

### LSCore.DependencyInjection

The mapper module depends on `LSCore.DependencyInjection` for two things:

1. **Assembly scanning** -- `AddLSCoreDependencyInjection` scans assemblies matching your project root name, finds classes implementing `ILSCoreMapper<,>`, and registers them as singletons.
2. **Service provider access** -- `UseLSCoreDependencyInjection` stores the `IServiceProvider` in `Container.ServiceProvider`, which the `ToMapped` extension method uses to resolve mapper instances at runtime.

### LSCore.SortAndPage

The mapper extension methods compose naturally with the SortAndPage module. You can pass a mapper call as the projection function when building sorted and paged responses:

```csharp
using LSCore.Mapper.Domain;
using LSCore.SortAndPage.Domain;

public LSCoreSortedAndPagedResponse<ProductDto> GetAll(ProductsGetAllRequest request) =>
    productRepository
        .GetAll()
        .ToSortedAndPagedResponse<ProductEntity, ProductsSortColumn, ProductDto>(
            request,
            SortColumnRules.ProductsSortColumnCodesRules,
            x => x.ToMapped<ProductEntity, ProductDto>()
        );
```

### LSCore.Exceptions

Mapper error cases (missing mapper registration, uninitialized service provider) throw standard .NET exceptions. You can combine these with the `LSCore.Exceptions` middleware to handle other error paths (not-found, bad request, etc.) in the same API pipeline.
