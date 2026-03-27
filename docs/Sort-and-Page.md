---
layout: default
title: Sort and Page
nav_order: 9
---

# LSCore SortAndPage

The SortAndPage module provides a standardized approach to sorting and paginating `IQueryable<T>` data sources. It is split into two NuGet packages: a lightweight **Contracts** package for shared models and a **Domain** package that contains the query-execution logic.

## NuGet Packages

| Package | Description |
|---|---|
| `LSCore.SortAndPage.Contracts` | Request/response models, sort rules, pagination data, and the `IQueryable` extension methods for building queries. No external dependencies. |
| `LSCore.SortAndPage.Domain` | The `ToSortedAndPagedResponse` extension methods that execute queries and return fully populated responses. Depends on the Contracts package. |

Both packages target **.NET 9.0**.

---

## Core Concepts

### Sort Column Enum

Every sortable request is generic over a `TSortColumn` type parameter constrained to `struct`. In practice this is an `enum` you define in your project that lists the columns a client is allowed to sort by.

```csharp
public enum UserSortColumn
{
    Name,
    Email,
    CreatedAt
}
```

### Sort Rules Dictionary

A `Dictionary<TSortColumn, LSCoreSortRule<T>>` maps each enum value to the LINQ expression that should be used when that column is requested. This dictionary is the single source of truth for how sort columns translate into database expressions.

```csharp
var sortRules = new Dictionary<UserSortColumn, LSCoreSortRule<User>>
{
    { UserSortColumn.Name,      new LSCoreSortRule<User>(u => u.Name) },
    { UserSortColumn.Email,     new LSCoreSortRule<User>(u => u.Email) },
    { UserSortColumn.CreatedAt, new LSCoreSortRule<User>(u => u.CreatedAt) }
};
```

---

## Request Models

### LSCoreSortableRequest\<TSortColumn\>

The base request for sorting without pagination.

```csharp
public class LSCoreSortableRequest<TSortColumn>
    where TSortColumn : struct
{
    public TSortColumn? SortColumn { get; set; }
    public ListSortDirection SortDirection { get; set; } = ListSortDirection.Ascending;
}
```

| Property | Type | Default | Description |
|---|---|---|---|
| `SortColumn` | `TSortColumn?` | `null` | The column to sort by. When `null`, no sorting is applied. |
| `SortDirection` | `ListSortDirection` | `Ascending` | `Ascending` or `Descending`. Uses `System.ComponentModel.ListSortDirection`. |

### LSCoreSortableAndPageableRequest\<TSortColumn\>

Extends `LSCoreSortableRequest<TSortColumn>` with pagination fields.

```csharp
public class LSCoreSortableAndPageableRequest<TSortColumn> : LSCoreSortableRequest<TSortColumn>
    where TSortColumn : struct
{
    public int PageSize { get; set; } = 10;
    public int CurrentPage { get; set; } = 1;
}
```

| Property | Type | Default | Description |
|---|---|---|---|
| `PageSize` | `int` | `10` | Number of items per page. |
| `CurrentPage` | `int` | `1` | The 1-based page number to retrieve. |

All properties from `LSCoreSortableRequest` (`SortColumn`, `SortDirection`) are inherited.

---

## Response Model

### LSCoreSortedAndPagedResponse\<TPayload\>

The standard response wrapper returned after sorting and paging a query.

```csharp
public class LSCoreSortedAndPagedResponse<TPayload>
{
    public List<TPayload>? Payload { get; set; }
    public LSCorePaginationData? Pagination { get; set; }
}
```

| Property | Type | Description |
|---|---|---|
| `Payload` | `List<TPayload>?` | The items for the current page. |
| `Pagination` | `LSCorePaginationData?` | Metadata about the current page, page size, total count, and total pages. |

---

## Pagination Data

### LSCorePaginationData

A `record` that carries pagination metadata.

```csharp
public record LSCorePaginationData(int Page, int PageSize, int TotalCount)
{
    public int TotalPages => TotalCount / PageSize + (TotalCount % PageSize == 0 ? 0 : 1);
}
```

| Property | Type | Description |
|---|---|---|
| `Page` | `int` | The current page number (1-based). |
| `PageSize` | `int` | The number of items per page. |
| `TotalCount` | `int` | The total number of items across all pages. |
| `TotalPages` | `int` | Computed. The total number of pages, accounting for a partial final page. |

---

## Sort Rules

### LSCoreSortRule\<T\>

Wraps a LINQ expression that tells the sorting engine which property (or computed value) to sort by.

```csharp
public class LSCoreSortRule<T>(Expression<Func<T, object>> sortExpression)
    where T : class
{
    public Expression<Func<T, object>> SortExpression { get; } = sortExpression;
}
```

The expression is typed as `Expression<Func<T, object>>` so it can be passed directly to `OrderBy` / `OrderByDescending` on an `IQueryable<T>`.

---

## QueryableExtensions (Contracts)

These extension methods live in `LSCore.SortAndPage.Contracts` and operate on `IQueryable<T>` without executing the query. They are useful when you need to compose additional query operations after sorting/paging.

### SortQuery

Applies sorting to a query based on the request and sort rules. Returns the query unchanged if `SortColumn` is `null`.

```csharp
public static IQueryable<T> SortQuery<T, TSortColumn>(
    this IQueryable<T> source,
    LSCoreSortableRequest<TSortColumn> request,
    Dictionary<TSortColumn, LSCoreSortRule<T>> sortRules
)
```

### SortAndPageQuery

Applies sorting (via `SortQuery`) and then pagination (`Skip` / `Take`) to a query.

```csharp
public static IQueryable<T> SortAndPageQuery<T, TSortColumn>(
    this IQueryable<T> source,
    LSCoreSortableAndPageableRequest<TSortColumn> request,
    Dictionary<TSortColumn, LSCoreSortRule<T>> sortRules
)
```

---

## Domain Extensions

These extension methods live in `LSCore.SortAndPage.Domain` and **execute** the query, returning a fully populated `LSCoreSortedAndPagedResponse`.

### ToSortedAndPagedResponse (same type)

Counts the total items, applies sort and page, and materializes the results.

```csharp
public static LSCoreSortedAndPagedResponse<T> ToSortedAndPagedResponse<T, TSortColumn>(
    this IQueryable<T> source,
    LSCoreSortableAndPageableRequest<TSortColumn> request,
    Dictionary<TSortColumn, LSCoreSortRule<T>> sortRules
)
```

### ToSortedAndPagedResponse (with mapping)

Same as above, but accepts a `Func<T, TPayload>` mapping function to project each entity into a different type (e.g., a DTO or view model).

```csharp
public static LSCoreSortedAndPagedResponse<TPayload> ToSortedAndPagedResponse<T, TSortColumn, TPayload>(
    this IQueryable<T> source,
    LSCoreSortableAndPageableRequest<TSortColumn> request,
    Dictionary<TSortColumn, LSCoreSortRule<T>> sortRules,
    Func<T, TPayload> mapFunc
)
```

---

## Usage Examples

### 1. Define a sort column enum and sort rules

```csharp
public enum ProductSortColumn
{
    Name,
    Price,
    CreatedAt
}

public static class ProductSortRules
{
    public static readonly Dictionary<ProductSortColumn, LSCoreSortRule<Product>> Rules = new()
    {
        { ProductSortColumn.Name,      new LSCoreSortRule<Product>(p => p.Name) },
        { ProductSortColumn.Price,     new LSCoreSortRule<Product>(p => p.Price) },
        { ProductSortColumn.CreatedAt, new LSCoreSortRule<Product>(p => p.CreatedAt) }
    };
}
```

### 2. Accept the request in a controller

```csharp
[HttpGet("products")]
public IActionResult GetProducts([FromQuery] LSCoreSortableAndPageableRequest<ProductSortColumn> request)
{
    var response = _productRepository.GetProducts(request);
    return Ok(response);
}
```

The client sends query parameters like:

```
GET /products?SortColumn=Price&SortDirection=Descending&PageSize=20&CurrentPage=2
```

### 3. Execute in a repository -- returning entities directly

```csharp
public LSCoreSortedAndPagedResponse<Product> GetProducts(
    LSCoreSortableAndPageableRequest<ProductSortColumn> request)
{
    return _context.Products
        .AsQueryable()
        .ToSortedAndPagedResponse(request, ProductSortRules.Rules);
}
```

### 4. Execute in a repository -- mapping to a DTO

```csharp
public LSCoreSortedAndPagedResponse<ProductDto> GetProducts(
    LSCoreSortableAndPageableRequest<ProductSortColumn> request)
{
    return _context.Products
        .AsQueryable()
        .ToSortedAndPagedResponse(
            request,
            ProductSortRules.Rules,
            product => new ProductDto
            {
                Id = product.Id,
                Name = product.Name,
                Price = product.Price
            }
        );
}
```

### 5. Sort-only (no pagination)

If you only need sorting without pagination, use `SortQuery` from the Contracts package directly:

```csharp
public List<Product> GetSortedProducts(LSCoreSortableRequest<ProductSortColumn> request)
{
    return _context.Products
        .AsQueryable()
        .SortQuery(request, ProductSortRules.Rules)
        .ToList();
}
```

### 6. Composing additional query logic

Because `SortQuery` and `SortAndPageQuery` return `IQueryable<T>`, you can chain additional LINQ operations:

```csharp
var query = _context.Products
    .Where(p => p.IsActive)
    .SortAndPageQuery(request, ProductSortRules.Rules);

// Add projection or further filtering
var results = query
    .Select(p => new { p.Name, p.Price })
    .ToList();
```

---

## Example JSON Response

A typical API response using `LSCoreSortedAndPagedResponse` looks like:

```json
{
  "payload": [
    { "id": 1, "name": "Widget A", "price": 9.99 },
    { "id": 2, "name": "Widget B", "price": 14.99 }
  ],
  "pagination": {
    "page": 2,
    "pageSize": 10,
    "totalCount": 55,
    "totalPages": 6
  }
}
```
