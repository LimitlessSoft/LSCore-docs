---
layout: default
title: Getting Started
nav_order: 2
---

# Getting Started with LSCore

LSCore is a modular .NET 9 framework that provides building blocks for ASP.NET Core applications, including authentication, validation, object mapping, sorting/pagination, repository abstractions, exception handling, and more. Each module is published as a separate NuGet package so you only pull in what you need.

---

## Prerequisites

- [.NET 9 SDK](https://dotnet.microsoft.com/download/dotnet/9.0) or later
- An IDE such as Visual Studio, Rider, or VS Code with the C# extension

---

## Available NuGet Packages

All packages share the same version (currently `9.1.4.1`) and are published from the [LimitlessSoft/LSCore](https://github.com/LimitlessSoft/LSCore) repository.

### Core / Infrastructure

| Package | Description |
|---|---|
| `LSCore.Build` | Meta-package that references every LSCore library. Install this to pull in the entire framework at once. |
| `LSCore.DependencyInjection` | Convention-based dependency injection resolution for the LSCore ecosystem. Scans assemblies by prefix and auto-registers services. |
| `LSCore.Exceptions` | Typed exception classes (`LSCoreNotFoundException`, etc.) used across the framework. |
| `LSCore.Exceptions.DependencyInjection` | Exception-handling middleware for ASP.NET Core. Converts LSCore exceptions into appropriate HTTP responses. |
| `LSCore.Logging` | Logging infrastructure for ASP.NET Core applications. |
| `LSCore.Common.Contracts` | Shared contracts and interfaces that have not yet been promoted to their own packages. |
| `LSCore.Common.Extensions` | Shared extension methods that have not yet been promoted to their own packages. |

### Authentication -- API Key

| Package | Description |
|---|---|
| `LSCore.Auth.Contracts` | Base authentication contracts (e.g. `[LSCoreAuth]` attribute) shared by all auth modules. |
| `LSCore.Auth.Key.Contracts` | Contracts for API-key authentication (`LSCoreAuthKeyConfiguration`, `ILSCoreAuthKeyProvider`). |
| `LSCore.Auth.Key.DependencyInjection` | DI registration and middleware for API-key authentication. |

### Authentication -- Username / Password (JWT)

| Package | Description |
|---|---|
| `LSCore.Auth.UserPass.Contracts` | Contracts for username/password auth (`ILSCoreAuthUserPassEntity`, `ILSCoreAuthUserPassManager`, configuration). |
| `LSCore.Auth.UserPass.DependencyInjection` | DI registration and JWT middleware for username/password authentication. |
| `LSCore.Auth.UserPass.Domain` | Domain logic: `LSCoreAuthUserPassManager`, password hashing helpers, token generation. |

### Authentication -- Role-Based

| Package | Description |
|---|---|
| `LSCore.Auth.Role.Contracts` | Contracts for role-based authorization (`ILSCoreAuthRoleEntity`, `[LSCoreAuthRole]` attribute). |
| `LSCore.Auth.Role.DependencyInjection` | DI registration and middleware for role-based authorization. |
| `LSCore.Auth.Role.Domain` | Domain logic for role-based authorization. |

### Authentication -- Permission-Based

| Package | Description |
|---|---|
| `LSCore.Auth.Permission.Contracts` | Contracts for permission-based authorization (`ILSCoreAuthPermissionEntity`, `[LSCoreAuthPermission]` attribute). |
| `LSCore.Auth.Permission.DependencyInjection` | DI registration and middleware for permission-based authorization. |
| `LSCore.Auth.Permission.Domain` | Domain logic for permission-based authorization. |

### Object Mapping

| Package | Description |
|---|---|
| `LSCore.Mapper.Contracts` | Mapper contract (`ILSCoreMapper<TSource, TDestination>`). |
| `LSCore.Mapper.Domain` | Extension methods (`ToMapped`, `ToMappedList`) that resolve mappers through DI. |

### Validation

| Package | Description |
|---|---|
| `LSCore.Validation.Contracts` | Validation contracts and the `[LSCoreValidationMessage]` attribute for enum-based validation codes. |
| `LSCore.Validation.Domain` | `LSCoreValidatorBase<T>` base class and the `.Validate()` extension method. Built on FluentValidation. |

### Sorting and Pagination

| Package | Description |
|---|---|
| `LSCore.SortAndPage.Contracts` | Contracts: `LSCoreSortableAndPageableRequest`, `LSCoreSortedAndPagedResponse`, `LSCoreSortRule`. |
| `LSCore.SortAndPage.Domain` | Extension method `ToSortedAndPagedResponse` for `IQueryable<T>`. |

### Repository

| Package | Description |
|---|---|
| `LSCore.Repository.Contracts` | Repository contracts and Entity Framework Core abstractions. |
| `LSCore.Repository` | Repository implementations built on EF Core. |

### API Client

| Package | Description |
|---|---|
| `LSCore.ApiClient.Rest` | REST API client for service-to-service communication. |
| `LSCore.ApiClient.Rest.DependencyInjection` | DI registration for the REST API client. |

---

## Installation

Install packages from NuGet. For example, to add exception handling and API-key authentication:

```bash
dotnet add package LSCore.Exceptions.DependencyInjection
dotnet add package LSCore.Auth.Key.Contracts
dotnet add package LSCore.Auth.Key.DependencyInjection
```

Or, to install the entire framework at once:

```bash
dotnet add package LSCore.Build
```

---

## Quick Start -- Minimal API-Key Authenticated App

This example creates an ASP.NET Core API that protects selected endpoints with an API key. It mirrors the `Sample.AuthKey.Api` sample application.

### 1. Create the project

```bash
dotnet new webapi -n MyApp
cd MyApp
dotnet add package LSCore.Auth.Key.Contracts
dotnet add package LSCore.Auth.Key.DependencyInjection
dotnet add package LSCore.Exceptions.DependencyInjection
```

### 2. Configure `Program.cs`

```csharp
using LSCore.Auth.Key.Contracts;
using LSCore.Auth.Key.DependencyInjection;
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.AddLSCoreAuthKey(
    new LSCoreAuthKeyConfiguration
    {
        ValidKeys = ["my-secret-api-key-1", "my-secret-api-key-2"]
    }
);
var app = builder.Build();
app.UseLSCoreExceptionsHandler();
app.UseLSCoreAuthKey();
app.MapControllers();
app.Run();
```

### 3. Create a controller

```csharp
using LSCore.Auth.Contracts;
using Microsoft.AspNetCore.Mvc;

public class ProductsController : ControllerBase
{
    [HttpGet]
    [Route("/products")]
    public IActionResult Get() => Ok("Public data -- no key required");

    [HttpGet]
    [LSCoreAuth]
    [Route("/products-auth")]
    public IActionResult GetAuth() => Ok("Protected data -- valid API key required");
}
```

Endpoints without `[LSCoreAuth]` are publicly accessible. Endpoints decorated with `[LSCoreAuth]` require a valid API key in the request.

> **See also:** The [Sample.AuthKey.Api](../src/SampleApps/AuthKey/Sample.AuthKey.Api/) sample application.

---

## Setting Up Username/Password (JWT) Authentication

For login-based authentication with JWT tokens, use the `LSCore.Auth.UserPass` packages. This mirrors the `Sample.AuthUserPass.Api` sample application.

### 1. Install packages

```bash
dotnet add package LSCore.Auth.UserPass.Contracts
dotnet add package LSCore.Auth.UserPass.DependencyInjection
dotnet add package LSCore.Auth.UserPass.Domain
dotnet add package LSCore.Exceptions.DependencyInjection
```

### 2. Define a user entity

Your user entity must implement `ILSCoreAuthUserPassEntity<TIdentifier>`:

```csharp
using LSCore.Auth.UserPass.Contracts;

public class UserEntity : ILSCoreAuthUserPassEntity<string>
{
    public long Id { get; set; }
    public string Username { get; set; }
    public string Identifier => Username;
    public string? RefreshToken { get; set; }
    public string Password { get; set; }
}
```

### 3. Implement the repository

Implement `ILSCoreAuthUserPassIdentityEntityRepository<TIdentifier>` to look up users and persist refresh tokens:

```csharp
using LSCore.Auth.UserPass.Contracts;
using LSCore.Auth.UserPass.Domain;
using LSCore.Exceptions;

public class UserRepository : ILSCoreAuthUserPassIdentityEntityRepository<string>
{
    private static List<UserEntity> _users = [];

    public ILSCoreAuthUserPassEntity<string>? GetOrDefault(string identifier) =>
        _users.FirstOrDefault(x => x.Identifier == identifier);

    public void SetRefreshToken(string entityIdentifier, string refreshToken)
    {
        var user = _users.FirstOrDefault(x => x.Identifier == entityIdentifier);
        if (user == null)
            throw new LSCoreNotFoundException();
        user.RefreshToken = refreshToken;
    }
}
```

### 4. Create an auth manager

Extend `LSCoreAuthUserPassManager<TIdentifier>`:

```csharp
using LSCore.Auth.UserPass.Contracts;
using LSCore.Auth.UserPass.Domain;

public class AuthManager(
    ILSCoreAuthUserPassIdentityEntityRepository<string> userPassIdentityEntityRepository,
    LSCoreAuthUserPassConfiguration configuration
) : LSCoreAuthUserPassManager<string>(userPassIdentityEntityRepository, configuration);
```

### 5. Wire up `Program.cs`

```csharp
using LSCore.Auth.UserPass.Contracts;
using LSCore.Auth.UserPass.DependencyInjection;
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.AddLSCoreAuthUserPass<string, AuthManager, UserRepository>(
    new LSCoreAuthUserPassConfiguration
    {
        SecurityKey = "your-secret-key-at-least-32-characters-long",
        Issuer = "MyApp",
        Audience = "MyApp"
    }
);
var app = builder.Build();
app.UseLSCoreExceptionsHandler();
app.UseLSCoreAuthUserPass<string>();
app.MapControllers();
app.Run();
```

### 6. Create a controller with login and protected endpoints

```csharp
using LSCore.Auth.Contracts;
using LSCore.Auth.UserPass.Contracts;
using Microsoft.AspNetCore.Mvc;

public class UsersController(ILSCoreAuthUserPassManager<string> authPasswordManager)
    : ControllerBase
{
    [HttpPost]
    [Route("/login")]
    public IActionResult Login([FromBody] LoginRequest request) =>
        Ok(authPasswordManager.Authenticate(request.Username, request.Password));

    [HttpGet]
    [Route("/users")]
    public IActionResult Get() => Ok("Public data");

    [HttpGet]
    [LSCoreAuth]
    [Route("/users-auth")]
    public IActionResult GetAuth() => Ok("Protected data -- valid JWT required");
}
```

> **See also:** The [Sample.AuthUserPass.Api](../src/SampleApps/AuthPassword/Sample.AuthUserPass.Api/) sample application.

---

## Adding Role-Based Authorization

Build on top of username/password auth by adding role checks. This mirrors the `Sample.AuthRole.Api` sample application.

### 1. Install additional packages

```bash
dotnet add package LSCore.Auth.Role.Contracts
dotnet add package LSCore.Auth.Role.DependencyInjection
```

### 2. Define a role enum

```csharp
public enum UserRole
{
    User,
    Administrator
}
```

### 3. Extend the user entity

```csharp
using LSCore.Auth.Role.Contracts;
using LSCore.Auth.UserPass.Contracts;

public class UserEntity : ILSCoreAuthUserPassEntity<string>, ILSCoreAuthRoleEntity<string, UserRole>
{
    public long Id { get; set; }
    public UserRole Role { get; set; } = UserRole.User;
    public string Username { get; set; }
    public string Identifier => Username;
    public string? RefreshToken { get; set; }
    public string Password { get; set; }
}
```

### 4. Extend the repository

The repository must also implement `ILSCoreAuthRoleIdentityEntityRepository<string, UserRole>`:

```csharp
public class UserRepository
    : ILSCoreAuthUserPassIdentityEntityRepository<string>,
      ILSCoreAuthRoleIdentityEntityRepository<string, UserRole>
{
    // ... existing methods ...

    public ILSCoreAuthRoleEntity<string, UserRole>? GetOrDefault(string identifier) =>
        _users.FirstOrDefault(x => x.Identifier == identifier);
}
```

### 5. Register in `Program.cs`

```csharp
builder.AddLSCoreAuthUserPass<string, AuthManager, UserRepository>(
    new LSCoreAuthUserPassConfiguration { /* ... */ }
);
builder.AddLSCoreAuthRole<string, UserRole, UserRepository>();

// ...

app.UseLSCoreAuthUserPass<string>();
app.UseLSCoreAuthRole<string, UserRole>();
```

### 6. Protect endpoints by role

```csharp
using LSCore.Auth.Contracts;
using LSCore.Auth.Role.Contracts;

// Any authenticated user
[LSCoreAuth]
[Route("/users-auth-any-role")]
public IActionResult GetAnyRole() => Ok("Any authenticated user can see this");

// Only administrators
[LSCoreAuthRole<UserRole>(UserRole.Administrator)]
[Route("/users-auth-administrator-role")]
public IActionResult GetAdministratorRole() => Ok("Administrators only");
```

> **See also:** The [Sample.AuthRole.Api](../src/SampleApps/AuthRole/Sample.AuthRole.Api/) sample application.

---

## Adding Permission-Based Authorization

Permission-based auth works similarly to roles, but allows assigning multiple granular permissions per user. This mirrors the `Sample.AuthPermission.Api` sample application.

### 1. Define a permission enum

```csharp
public enum UserPermission
{
    Permission_One,
    Permission_Two,
    Permission_Three
}
```

### 2. Implement the entity

```csharp
using LSCore.Auth.Permission.Contracts;
using LSCore.Auth.UserPass.Contracts;

public class UserEntity
    : ILSCoreAuthUserPassEntity<string>,
      ILSCoreAuthPermissionEntity<string, UserPermission>
{
    public long Id { get; set; }
    public ICollection<UserPermission> Permissions { get; set; } = [];
    public string Username { get; set; }
    public string Identifier => Username;
    public string? RefreshToken { get; set; }
    public string Password { get; set; }
}
```

### 3. Register in `Program.cs`

```csharp
builder.AddLSCoreAuthUserPass<string, AuthManager, UserRepository>(/* ... */);
builder.AddLSCoreAuthPermission<string, UserPermission, UserRepository>();

// ...

app.UseLSCoreAuthUserPass<string>();
app.UseLSCoreAuthPermission<string, UserPermission>();
```

### 4. Protect endpoints by permission

```csharp
using LSCore.Auth.Permission.Contracts;

// Require ALL listed permissions (default behavior)
[LSCoreAuthPermission<UserPermission>(
    UserPermission.Permission_One,
    UserPermission.Permission_Two
)]
[Route("/users-auth-all-permissions")]
public IActionResult GetHasPermissions() =>
    Ok("User must have all of the permissions to see this");

// Require ANY of the listed permissions (pass false as first argument)
[LSCoreAuthPermission<UserPermission>(
    false,
    UserPermission.Permission_One,
    UserPermission.Permission_Two,
    UserPermission.Permission_Three
)]
[Route("/users-auth-any-permission")]
public IActionResult GetAnyPermission() =>
    Ok("User must have any of the permissions to see this");
```

> **See also:** The [Sample.AuthPermission.Api](../src/SampleApps/AuthPermission/Sample.AuthPermission.Api/) sample application.

---

## Combining Multiple Auth Strategies

You can layer API-key and username/password auth on the same application. Set `BreakOnFailedAuth = false` on the first strategy so that if it fails, the next strategy gets a chance. This mirrors the `Sample.AuthCombined.Api` sample application.

```csharp
using LSCore.Auth.Key.Contracts;
using LSCore.Auth.Key.DependencyInjection;
using LSCore.Auth.UserPass.Contracts;
using LSCore.Auth.UserPass.DependencyInjection;
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.AddLSCoreAuthKey(
    new LSCoreAuthKeyConfiguration
    {
        ValidKeys = ["ThisIsFirstValidKey", "ThisIsSecondValidKey"],
        BreakOnFailedAuth = false  // Allow fallthrough to next auth strategy
    }
);
builder.AddLSCoreAuthUserPass<string, AuthManager, UserRepository>(
    new LSCoreAuthUserPassConfiguration
    {
        SecurityKey = "your-secret-key-at-least-32-characters-long",
        Audience = "MyApp",
        Issuer = "MyApp",
    }
);
var app = builder.Build();
app.UseLSCoreExceptionsHandler();
app.MapControllers();
app.UseLSCoreAuthKey();
app.UseLSCoreAuthUserPass<string>();
app.Run();
```

A request will succeed if it passes **either** API-key auth **or** JWT auth.

> **See also:** The [Sample.AuthCombined.Api](../src/SampleApps/AuthCombined/Sample.AuthCombined.Api/) sample application.

---

## Adding Validation

LSCore validation is built on [FluentValidation](https://docs.fluentvalidation.net/) with additional conventions. This mirrors the `Sample.Validation.Api` sample application.

### 1. Install packages

```bash
dotnet add package LSCore.Validation.Domain
dotnet add package LSCore.Exceptions.DependencyInjection
```

### 2. Define a request class

```csharp
public class UserRegisterRequest
{
    public string Username { get; set; }
    public string Password { get; set; }
}
```

### 3. Define validation codes (optional but recommended)

Use enums with `[LSCoreValidationMessage]` to keep error messages centralized:

```csharp
using LSCore.Validation.Contracts;

public enum UsersValidationCodes
{
    [LSCoreValidationMessage("Password must contain only letters and numbers.")]
    UVC_001,

    [LSCoreValidationMessage("Password must contains at least one number and one letter.")]
    UVC_002
}
```

### 4. Create a validator

Extend `LSCoreValidatorBase<T>`:

```csharp
using System.Text.RegularExpressions;
using FluentValidation;
using LSCore.Validation.Contracts;
using LSCore.Validation.Domain;

public class UserRegisterRequestValidator : LSCoreValidatorBase<UserRegisterRequest>
{
    const int MIN_PASSWORD_LENGTH = 8;
    const string PASSWORD_REGEX_ONLY_LETTERS_AND_NUMBERS = @"^[A-Za-z\d]+$";
    const string PASSWORD_REGEX_AT_LEAST_ONE_LETTER_AND_ONE_NUMBER = @"^(?=.*[A-Za-z])(?=.*\d).+$";
    const int MIN_USERNAME_LENGTH = 5;
    const int MAX_USERNAME_LENGTH = 20;

    public UserRegisterRequestValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty()
            .MinimumLength(MIN_USERNAME_LENGTH)
            .MaximumLength(MAX_USERNAME_LENGTH);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(MIN_PASSWORD_LENGTH)
            .Must(x => Regex.IsMatch(x, PASSWORD_REGEX_ONLY_LETTERS_AND_NUMBERS))
            .WithMessage(UsersValidationCodes.UVC_001.GetValidationMessage())
            .Must(x => Regex.IsMatch(x, PASSWORD_REGEX_AT_LEAST_ONE_LETTER_AND_ONE_NUMBER))
            .WithMessage(UsersValidationCodes.UVC_002.GetValidationMessage());
    }
}
```

### 5. Register with auto-DI and call `.Validate()`

```csharp
// Program.cs
using LSCore.DependencyInjection;
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.AddLSCoreDependencyInjection("Sample.Validation"); // auto-discovers validators
var app = builder.Build();
app.UseLSCoreDependencyInjection();
app.UseLSCoreExceptionsHandler();
app.MapControllers();
app.Run();
```

In the controller, call `request.Validate()` -- it throws an exception (handled by the exception middleware) if validation fails:

```csharp
using LSCore.Validation.Domain;
using Microsoft.AspNetCore.Mvc;

public class UsersController : ControllerBase
{
    [HttpPost]
    [Route("/register")]
    public IActionResult Register([FromBody] UserRegisterRequest request)
    {
        request.Validate();
        return Ok("User successfully registered!");
    }
}
```

> **See also:** The [Sample.Validation.Api](../src/SampleApps/Validation/Sample.Validation.Api/) and [Sample.ValidationWithRepository.Api](../src/SampleApps/Sample.ValidationWithRepository.Api/) sample applications.

---

## Validation with Repository Access

If your validator needs to query the database (e.g. to check for duplicate usernames), inject a repository into the validator constructor. This mirrors the `Sample.ValidationWithRepository.Api` sample application.

```csharp
using FluentValidation;
using LSCore.Validation.Domain;

public class UserRegisterRequestValidator : LSCoreValidatorBase<UserRegisterRequest>
{
    public UserRegisterRequestValidator(IUserRepository userRepository)
    {
        RuleFor(x => x.Username)
            .Must(username => userRepository.UsernameOccupied(username) == false)
            .WithMessage("User with given username already exists");
    }
}
```

> **Note:** Validators are typically registered as singleton or transient. If your repository depends on a scoped `DbContext`, use a factory pattern to create the context inside the validator.

---

## Object Mapping

LSCore provides a lightweight, convention-based mapper that resolves through DI. This mirrors the `Sample.Mapper.Api` sample application.

### 1. Install packages

```bash
dotnet add package LSCore.Mapper.Domain
dotnet add package LSCore.DependencyInjection
```

### 2. Define your entity and DTO

```csharp
// Entity
public class ProductEntity
{
    public long Id { get; set; }
    public string Name { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; }
}

// DTO
public class ProductDto
{
    public long Id { get; set; }
    public string Name { get; set; }
}
```

### 3. Create a mapper

Implement `ILSCoreMapper<TSource, TDestination>`:

```csharp
using LSCore.Mapper.Contracts;

public class ProductDtoMapper : ILSCoreMapper<ProductEntity, ProductDto>
{
    public ProductDto ToMapped(ProductEntity source) =>
        new() { Id = source.Id, Name = source.Name };
}
```

### 4. Use the mapper

The `ToMapped` and `ToMappedList` extension methods resolve the mapper through DI automatically:

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

### 5. Register with auto-DI

```csharp
using LSCore.DependencyInjection;
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.AddLSCoreDependencyInjection("Sample.Mapper"); // auto-discovers mappers, repos, managers
var app = builder.Build();
app.UseLSCoreDependencyInjection();
app.UseLSCoreExceptionsHandler();
app.MapControllers();
app.Run();
```

> **See also:** The [Sample.Mapper.Api](../src/SampleApps/Mapper/Sample.Mapper.Api/) sample application.

---

## Sorting and Pagination

LSCore provides a convention for sorted, paginated API responses. This mirrors the `Sample.SortAndPage.Api` sample application.

### 1. Install packages

```bash
dotnet add package LSCore.SortAndPage.Contracts
dotnet add package LSCore.SortAndPage.Domain
dotnet add package LSCore.Mapper.Domain
```

### 2. Define sort columns

```csharp
public enum ProductsSortColumn
{
    Id,
    Name
}
```

### 3. Define sort rules

Map each enum value to a property expression on your entity:

```csharp
using LSCore.SortAndPage.Contracts;

public static class SortColumnRules
{
    public static Dictionary<ProductsSortColumn, LSCoreSortRule<ProductEntity>>
        ProductsSortColumnCodesRules = new()
        {
            { ProductsSortColumn.Id, new LSCoreSortRule<ProductEntity>(x => x.Id) },
            { ProductsSortColumn.Name, new LSCoreSortRule<ProductEntity>(x => x.Name) }
        };
}
```

### 4. Create a sortable/pageable request

Extend `LSCoreSortableAndPageableRequest<TSortColumn>`:

```csharp
using LSCore.SortAndPage.Contracts;

public class ProductsGetAllRequest : LSCoreSortableAndPageableRequest<ProductsSortColumn>;
```

### 5. Apply sorting and pagination

Use `ToSortedAndPagedResponse` on an `IQueryable<T>`:

```csharp
using LSCore.Mapper.Domain;
using LSCore.SortAndPage.Domain;

public class ProductManager(IProductRepository productRepository) : IProductManager
{
    public LSCoreSortedAndPagedResponse<ProductDto> GetAll(ProductsGetAllRequest request) =>
        productRepository
            .GetAll()
            .ToSortedAndPagedResponse<ProductEntity, ProductsSortColumn, ProductDto>(
                request,
                SortColumnRules.ProductsSortColumnCodesRules,
                x => x.ToMapped<ProductEntity, ProductDto>()
            );
}
```

### 6. Expose via controller

```csharp
[HttpGet]
[Route("/products")]
public IActionResult GetAll([FromQuery] ProductsGetAllRequest request) =>
    Ok(productManager.GetAll(request));
```

> **See also:** The [Sample.SortAndPage.Api](../src/SampleApps/SortAndPage/Sample.SortAndPage.Api/) sample application.

---

## Auto-Discovery with `AddLSCoreDependencyInjection`

Several sample apps use `AddLSCoreDependencyInjection` to automatically discover and register:

- Services (interfaces to implementations)
- Validators
- Mappers

Pass your assembly name prefix so LSCore knows which assemblies to scan:

```csharp
builder.AddLSCoreDependencyInjection("MyApp");
```

Then activate it on the app:

```csharp
app.UseLSCoreDependencyInjection();
```

This removes the need to manually register every `IProductRepository -> ProductRepository` pair, every `ILSCoreMapper<,>` implementation, and every `LSCoreValidatorBase<>` subclass.

---

## Typical Full `Program.cs`

Here is a realistic `Program.cs` that combines authentication, validation, mapping, exception handling, and auto-DI:

```csharp
using LSCore.Auth.UserPass.Contracts;
using LSCore.Auth.UserPass.DependencyInjection;
using LSCore.Auth.Role.DependencyInjection;
using LSCore.DependencyInjection;
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();

// Auto-register services, validators, and mappers from assemblies matching the prefix
builder.AddLSCoreDependencyInjection("MyApp");

// Configure JWT authentication
builder.AddLSCoreAuthUserPass<string, AuthManager, UserRepository>(
    new LSCoreAuthUserPassConfiguration
    {
        SecurityKey = builder.Configuration["Auth:SecurityKey"]!,
        Issuer = builder.Configuration["Auth:Issuer"]!,
        Audience = builder.Configuration["Auth:Audience"]!
    }
);

// Configure role-based authorization
builder.AddLSCoreAuthRole<string, UserRole, UserRepository>();

var app = builder.Build();

// Exception handling middleware -- converts LSCore exceptions to HTTP responses
app.UseLSCoreExceptionsHandler();

// Activate auto-DI
app.UseLSCoreDependencyInjection();

// Auth middleware
app.UseLSCoreAuthUserPass<string>();
app.UseLSCoreAuthRole<string, UserRole>();

app.MapControllers();
app.Run();
```

---

## Custom API-Key Provider

Instead of hardcoding valid keys, you can implement `ILSCoreAuthKeyProvider` to load keys from a database, configuration, or any other source. This mirrors the `Sample.AuthKeyProvider.Api` sample application.

```csharp
using LSCore.Auth.Key.Contracts;

public class MyKeyProvider : ILSCoreAuthKeyProvider
{
    public bool IsValidKey(string key)
    {
        // Inject database, configuration, or any other source to validate keys
        return true;
    }
}
```

Register it with the generic overload:

```csharp
builder.AddLSCoreAuthKey<MyKeyProvider>(); // Registered as a scoped service
```

---

## Next Steps

- Browse the [sample applications guide](Samples.md) for detailed walkthroughs of each sample
- Explore the individual module documentation:
  - [Authentication](Auth-Overview.md)
  - [Validation](Validation.md)
  - [Mapper](Mapper.md)
  - [Sort and Page](Sort-and-Page.md)
  - [Repository](Repository.md)
  - [Exceptions](Exceptions.md)
  - [API Client](API-Client.md)
  - [Logging](Logging.md)
  - [Common](Common.md)
- Visit the [GitHub repository](https://github.com/LimitlessSoft/LSCore) for the latest source code
