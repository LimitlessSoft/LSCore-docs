---
layout: default
title: Samples
nav_order: 3
---

# LSCore Sample Applications

This guide describes each sample application in the `src/SampleApps/` directory, what it demonstrates, and how to run it. All samples target .NET 9 and follow the same project structure conventions.

---

## Running Any Sample

Every sample is a standalone ASP.NET Core Web API. To run one:

```bash
cd src/SampleApps/<SampleFolder>/<ProjectName>
dotnet run
```

The application will start on the port defined in its `Properties/launchSettings.json`. You can test endpoints using `curl`, Postman, or any HTTP client.

---

## Sample.AuthKey.Api

**Location:** `src/SampleApps/AuthKey/Sample.AuthKey.Api/`

**Purpose:** Demonstrates the simplest form of authentication -- protecting endpoints with static API keys.

**What it demonstrates:**
- Registering `LSCoreAuthKey` with a hardcoded list of valid keys
- Using the `[LSCoreAuth]` attribute to protect individual endpoints
- Leaving other endpoints publicly accessible

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Configures API-key auth with `AddLSCoreAuthKey` and a list of valid keys |
| `Controllers/ProductsController.cs` | Shows a public endpoint and a protected endpoint side by side |

**Program.cs setup:**

```csharp
using LSCore.Auth.Key.Contracts;
using LSCore.Auth.Key.DependencyInjection;
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.AddLSCoreAuthKey(
    new LSCoreAuthKeyConfiguration { ValidKeys = ["ThisIsFirstValidKey", "ThisIsSecondValidKey"] }
);
var app = builder.Build();
app.UseLSCoreExceptionsHandler();
app.UseLSCoreAuthKey();
app.MapControllers();
app.Run();
```

**Endpoints:**
- `GET /products` -- public, no authentication required
- `GET /products-auth` -- requires a valid API key

**Key pattern:** The `[LSCoreAuth]` attribute is opt-in per endpoint. Only decorated actions require authentication.

---

## Sample.AuthKeyProvider.Api

**Location:** `src/SampleApps/AuthKey/Sample.AuthKeyProvider.Api/`

**Purpose:** Demonstrates dynamic API-key validation using a custom provider instead of a static key list.

**What it demonstrates:**
- Implementing `ILSCoreAuthKeyProvider` to validate keys from any source (database, config, external service)
- Registering the provider with `AddLSCoreAuthKey<TProvider>()`

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Registers the custom key provider with the generic `AddLSCoreAuthKey<MyKeyProvider>()` overload |
| `MyKeyProvider.cs` | Custom implementation of `ILSCoreAuthKeyProvider` |
| `Controllers/ProductsController.cs` | Same public/protected endpoint pattern as `Sample.AuthKey.Api` |

**Custom key provider:**

```csharp
using LSCore.Auth.Key.Contracts;

public class MyKeyProvider : ILSCoreAuthKeyProvider
{
    public bool IsValidKey(string key)
    {
        // Inject database, configuration or any other way to load valid keys
        return true;
    }
}
```

**Key pattern:** Use the generic `AddLSCoreAuthKey<TProvider>()` to register a scoped key provider that can resolve dependencies from DI.

---

## Sample.AuthUserPass.Api

**Location:** `src/SampleApps/AuthPassword/Sample.AuthUserPass.Api/`

**Purpose:** Demonstrates username/password authentication with JWT tokens, including token generation and refresh token support.

**What it demonstrates:**
- Configuring JWT-based authentication with `AddLSCoreAuthUserPass`
- Implementing the user entity (`ILSCoreAuthUserPassEntity<string>`)
- Implementing the user repository (`ILSCoreAuthUserPassIdentityEntityRepository<string>`)
- Extending the base auth manager (`LSCoreAuthUserPassManager<string>`)
- Password hashing with `LSCoreAuthUserPassHelpers.HashPassword()`
- Using `[LSCoreAuth]` to protect endpoints requiring a valid JWT

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Configures JWT auth with security key, issuer, audience, and token skew |
| `UserEntity.cs` | User entity implementing `ILSCoreAuthUserPassEntity<string>` with `Identifier`, `Password`, `RefreshToken` |
| `UserRepository.cs` | In-memory repository with pre-seeded users and hashed passwords |
| `AuthManager.cs` | Minimal auth manager extending `LSCoreAuthUserPassManager<string>` |
| `LoginRequest.cs` | Simple DTO with `Username` and `Password` |
| `Controllers/UsersController.cs` | Login endpoint and protected/public data endpoints |

**Program.cs setup:**

```csharp
builder.AddLSCoreAuthUserPass<string, AuthManager, UserRepository>(
    new LSCoreAuthUserPassConfiguration
    {
        SecurityKey = "2e1cn102dvu10br02u109rc012c21ubrv0231rc10rvn1u2",
        Issuer = "Sample.AuthUserPass.Api",
        Audience = "Sample.AuthUserPass.Api",
        TokenSkew = TimeSpan.FromSeconds(30)
    }
);
```

**Endpoints:**
- `POST /login` -- authenticates with username/password, returns JWT tokens
- `GET /users` -- public
- `GET /users-auth` -- requires valid JWT

**Key pattern:** The three generic type parameters to `AddLSCoreAuthUserPass<TIdentifier, TManager, TRepository>()` allow full control over the identifier type, manager logic, and user lookup.

---

## Sample.AuthRole.Api

**Location:** `src/SampleApps/AuthRole/Sample.AuthRole.Api/`

**Purpose:** Demonstrates role-based authorization layered on top of username/password JWT auth.

**What it demonstrates:**
- Defining a role enum (`UserRole`)
- Implementing `ILSCoreAuthRoleEntity<string, UserRole>` on the user entity
- Implementing `ILSCoreAuthRoleIdentityEntityRepository<string, UserRole>` on the repository
- Using `[LSCoreAuthRole<UserRole>(UserRole.Administrator)]` to restrict endpoints to specific roles
- Using `[LSCoreAuth]` for "any authenticated user" access

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Registers both `AddLSCoreAuthUserPass` and `AddLSCoreAuthRole` |
| `Enums/UserRole.cs` | `User` and `Administrator` roles |
| `Entities/UserEntity.cs` | Implements both `ILSCoreAuthUserPassEntity` and `ILSCoreAuthRoleEntity` |
| `Repositories/UserRepository.cs` | Pre-seeds users; "SecondUser" has `Administrator` role |
| `Controllers/UsersController.cs` | Shows public, any-auth, and admin-only endpoints |

**Controller patterns:**

```csharp
// Any authenticated user
[LSCoreAuth]
[Route("/users-auth-any-role")]
public IActionResult GetAnyRole() => Ok("Here is some auth data");

// Only administrators
[LSCoreAuthRole<UserRole>(UserRole.Administrator)]
[Route("/users-auth-administrator-role")]
public IActionResult GetAdministratorRole() => Ok("Here is some auth data");
```

**Endpoints:**
- `POST /login` -- authenticate
- `GET /users` -- public
- `GET /users-auth-any-role` -- any authenticated user
- `GET /users-auth-administrator-role` -- administrators only

**Key pattern:** Role auth builds on UserPass auth. Register and use both middlewares in order.

---

## Sample.AuthPermission.Api

**Location:** `src/SampleApps/AuthPermission/Sample.AuthPermission.Api/`

**Purpose:** Demonstrates permission-based authorization, where users can have multiple granular permissions checked against endpoint requirements.

**What it demonstrates:**
- Defining a permission enum (`UserPermission`)
- Implementing `ILSCoreAuthPermissionEntity<string, UserPermission>` with a `Permissions` collection
- Implementing `ILSCoreAuthPermissionIdentityEntityRepository<string, UserPermission>`
- Requiring **all** listed permissions (default behavior)
- Requiring **any** of the listed permissions (pass `false` as first argument)

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Registers `AddLSCoreAuthUserPass` and `AddLSCoreAuthPermission` |
| `Enums/UserPermission.cs` | Three permissions: `Permission_One`, `Permission_Two`, `Permission_Three` |
| `Entities/UserEntity.cs` | Implements both UserPass and Permission entity interfaces |
| `Repositories/UserRepository.cs` | "FirstUser" has Permission_One and Permission_Two; others have none |
| `Interfaces/IUserRepository.cs` | Custom repository interface for app-specific user queries |
| `Controllers/UsersController.cs` | Demonstrates all-permissions, no-matching-permissions, and any-permission patterns |

**Controller patterns:**

```csharp
// Require ALL listed permissions
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
public IActionResult GetAnyRole() =>
    Ok("User must have any of the permissions to see this");
```

**Endpoints:**
- `POST /login` -- authenticate
- `GET /users` -- public
- `GET /users-auth-all-permissions` -- requires Permission_One AND Permission_Two
- `GET /users-auth-no-permission` -- requires all three permissions (will fail for seeded users)
- `GET /users-auth-any-permission` -- requires any of the three permissions

**Key pattern:** The first boolean argument to `[LSCoreAuthPermission]` controls AND vs OR logic. Omit it (or pass `true`) for ALL-required, pass `false` for ANY-required.

---

## Sample.AuthCombined.Api

**Location:** `src/SampleApps/AuthCombined/Sample.AuthCombined.Api/`

**Purpose:** Demonstrates combining API-key auth and username/password JWT auth on the same application, so requests can authenticate with either method.

**What it demonstrates:**
- Layering multiple auth strategies
- Using `BreakOnFailedAuth = false` on the key auth so it falls through to JWT auth on failure
- The order of `UseLSCoreAuthKey()` and `UseLSCoreAuthUserPass()` middleware

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Registers both key auth (with `BreakOnFailedAuth = false`) and UserPass auth |
| `AuthManager.cs` | Same minimal auth manager as the password sample |
| `UserRepository.cs` | Same in-memory user repository |
| `Controllers/UsersController.cs` | Login, public, and protected endpoints |

**Program.cs setup:**

```csharp
builder.AddLSCoreAuthKey(
    new LSCoreAuthKeyConfiguration
    {
        ValidKeys = ["ThisIsFirstValidKey", "ThisIsSecondValidKey"],
        BreakOnFailedAuth = false  // Falls through to next auth strategy
    }
);
builder.AddLSCoreAuthUserPass<string, AuthManager, UserRepository>(
    new LSCoreAuthUserPassConfiguration
    {
        SecurityKey = "2e1cn102dvu10br02u109rc012c21ubrv0231rc10rvn1u2",
        Audience = "Sample.AuthCombined.Api",
        Issuer = "Sample.AuthCombined.Api",
    }
);
```

**Endpoints:**
- `POST /login` -- authenticate with username/password
- `GET /users` -- public
- `GET /users-auth` -- requires either a valid API key OR a valid JWT

**Key pattern:** `BreakOnFailedAuth = false` is what enables fallthrough. Without it, a failed key auth would reject the request immediately.

---

## Sample.Validation.Api

**Location:** `src/SampleApps/Validation/Sample.Validation.Api/`

**Purpose:** Demonstrates request validation using LSCore's FluentValidation integration with enum-based validation codes.

**What it demonstrates:**
- Extending `LSCoreValidatorBase<T>` to define validation rules
- Using `[LSCoreValidationMessage]` attribute on enum values for centralized error messages
- Calling `.GetValidationMessage()` to retrieve the message from an enum value
- Calling `request.Validate()` in the controller to trigger validation
- Using `AddLSCoreDependencyInjection` to auto-discover validators

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Uses `AddLSCoreDependencyInjection("Sample.Validation")` to auto-register validators |
| `Requests/UserRegisterRequest.cs` | Simple request with `Username` and `Password` |
| `Enums/ValidationCodes/UsersValidationCodes.cs` | Enum with `[LSCoreValidationMessage]` attributes |
| `Validators/UserRegisterRequestValidator.cs` | Validator with rules for username length, password complexity |
| `Controllers/UsersController.cs` | Calls `request.Validate()` before processing |

**Validator example:**

```csharp
public class UserRegisterRequestValidator : LSCoreValidatorBase<UserRegisterRequest>
{
    public UserRegisterRequestValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty()
            .MinimumLength(5)
            .MaximumLength(20);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Must(x => Regex.IsMatch(x, @"^[A-Za-z\d]+$"))
            .WithMessage(UsersValidationCodes.UVC_001.GetValidationMessage())
            .Must(x => Regex.IsMatch(x, @"^(?=.*[A-Za-z])(?=.*\d).+$"))
            .WithMessage(UsersValidationCodes.UVC_002.GetValidationMessage());
    }
}
```

**Endpoints:**
- `GET /users` -- public
- `POST /register` -- validates the request body; returns errors if validation fails

**Key pattern:** `request.Validate()` is an extension method that resolves the correct validator from DI and throws a structured exception if validation fails. The exception middleware converts it into an appropriate HTTP response.

---

## Sample.ValidationWithRepository.Api

**Location:** `src/SampleApps/Sample.ValidationWithRepository.Api/`

**Purpose:** Demonstrates validators that depend on a repository to perform data-dependent validation (e.g. checking if a username is already taken).

**What it demonstrates:**
- Injecting dependencies into a validator constructor
- Querying a repository inside validation rules
- The interaction between DI scoping and validator lifetime

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Uses `AddLSCoreDependencyInjection("Sample.ValidationWithRepository")` |
| `Requests/UserRegisterRequest.cs` | Simple request with `Username` and `Password` |
| `Validators/UserRegisterRequestValidator.cs` | Injects `IUserRepository` and checks for duplicate usernames |
| `Entities/UserEntity.cs` | Simple user entity |
| `Interfaces/IUserRepository.cs` | Repository interface with `UsernameOccupied` and `Register` |
| `Repositories/UserRepository.cs` | In-memory repository tracking registered users |
| `Controllers/UsersController.cs` | Validates then registers the user |

**Validator with repository dependency:**

```csharp
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

**Endpoints:**
- `GET /users` -- public
- `POST /register` -- validates for duplicate username, then registers the user

**Key pattern:** Validators can take constructor dependencies just like any other service. The important caveat is that if your repository depends on a scoped `DbContext`, you should use a factory pattern since validators may have a different lifetime.

---

## Sample.Mapper.Api

**Location:** `src/SampleApps/Mapper/Sample.Mapper.Api/`

**Purpose:** Demonstrates LSCore's lightweight, convention-based object mapping between entities and DTOs.

**What it demonstrates:**
- Implementing `ILSCoreMapper<TSource, TDestination>`
- Using `.ToMapped<TSource, TDestination>()` for single objects
- Using `.ToMappedList<TSource, TDestination>()` for collections
- Auto-discovering mappers via `AddLSCoreDependencyInjection`

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Uses `AddLSCoreDependencyInjection("Sample.Mapper")` to auto-register everything |
| `Entities/ProductEntity.cs` | Full entity with `Id`, `Name`, `CreatedAt`, `IsActive` |
| `Dtos/ProductDto.cs` | Simplified DTO with only `Id` and `Name` |
| `Mappers/ProductDtoMapper.cs` | Maps `ProductEntity` to `ProductDto` |
| `Interfaces/IProductManager.cs` | Manager interface |
| `Interfaces/IProductRepository.cs` | Repository interface |
| `Managers/ProductManager.cs` | Uses `ToMapped` and `ToMappedList` extension methods |
| `Repositories/ProductRepository.cs` | In-memory product data; filters inactive products |
| `Controllers/ProductsControllers.cs` | GET all and GET by ID endpoints |

**Mapper implementation:**

```csharp
public class ProductDtoMapper : ILSCoreMapper<ProductEntity, ProductDto>
{
    public ProductDto ToMapped(ProductEntity source) =>
        new() { Id = source.Id, Name = source.Name };
}
```

**Usage in manager:**

```csharp
public ProductDto Get(int id) =>
    productRepository.Get(id).ToMapped<ProductEntity, ProductDto>();

public List<ProductDto> GetAll() =>
    productRepository.GetAll().ToMappedList<ProductEntity, ProductDto>();
```

**Endpoints:**
- `GET /products` -- returns all active products as DTOs
- `GET /products/{id}` -- returns a single product DTO; throws `LSCoreNotFoundException` if not found

**Key pattern:** Mappers are resolved from DI automatically by the `ToMapped` and `ToMappedList` extension methods. You never need to inject or reference the mapper directly.

---

## Sample.SortAndPage.Api

**Location:** `src/SampleApps/SortAndPage/Sample.SortAndPage.Api/`

**Purpose:** Demonstrates server-side sorting and pagination of `IQueryable` results using LSCore conventions.

**What it demonstrates:**
- Defining sortable columns as an enum
- Mapping sort columns to entity properties via `LSCoreSortRule<T>`
- Extending `LSCoreSortableAndPageableRequest<TSortColumn>` for request DTOs
- Using `ToSortedAndPagedResponse` to sort, paginate, and map in one step
- Returning `LSCoreSortedAndPagedResponse<TDto>` from the API

**Key files:**

| File | Role |
|---|---|
| `Program.cs` | Uses `AddLSCoreDependencyInjection("Sample.SortAndPage")` |
| `SortColumnCodes/ProductsSortColumn.cs` | Enum: `Id`, `Name` |
| `SortColumnRules.cs` | Static dictionary mapping enum values to `LSCoreSortRule<ProductEntity>` |
| `Requests/ProductsGetAllRequest.cs` | Extends `LSCoreSortableAndPageableRequest<ProductsSortColumn>` |
| `Entities/ProductEntity.cs` | Product entity with `Id`, `Name`, `CreatedAt`, `IsActive` |
| `Dtos/ProductDto.cs` | Simplified DTO |
| `Mappers/ProductDtoMapper.cs` | Entity-to-DTO mapper |
| `Interfaces/IProductRepository.cs` | Returns `IQueryable<ProductEntity>` |
| `Repositories/ProductRepository.cs` | Generates 100 random products at startup |
| `Managers/ProductManager.cs` | Applies sort, page, and map in one call |
| `Controllers/ProductsController.cs` | Accepts sort/page parameters from query string |

**Sort rules definition:**

```csharp
public static Dictionary<ProductsSortColumn, LSCoreSortRule<ProductEntity>>
    ProductsSortColumnCodesRules = new()
    {
        { ProductsSortColumn.Id, new LSCoreSortRule<ProductEntity>(x => x.Id) },
        { ProductsSortColumn.Name, new LSCoreSortRule<ProductEntity>(x => x.Name) }
    };
```

**Manager usage:**

```csharp
public LSCoreSortedAndPagedResponse<ProductDto> GetAll(ProductsGetAllRequest request) =>
    productRepository
        .GetAll()
        .ToSortedAndPagedResponse<ProductEntity, ProductsSortColumn, ProductDto>(
            request,
            SortColumnRules.ProductsSortColumnCodesRules,
            x => x.ToMapped<ProductEntity, ProductDto>()
        );
```

**Endpoints:**
- `GET /products?SortColumn=Id&SortDirection=Asc&Page=1&PageSize=10` -- returns a sorted, paginated response

**Key pattern:** The repository returns `IQueryable<T>` and the sorting/pagination is applied at the `IQueryable` level, so it works with EF Core and database-level queries. The `ToSortedAndPagedResponse` extension handles sorting, pagination, and mapping in a single composable call.

---

## Summary Table

| Sample | Auth Type | Key LSCore Modules | Primary Pattern |
|---|---|---|---|
| Sample.AuthKey.Api | API Key (static list) | Auth.Key | `[LSCoreAuth]` attribute on endpoints |
| Sample.AuthKeyProvider.Api | API Key (custom provider) | Auth.Key | `ILSCoreAuthKeyProvider` for dynamic key validation |
| Sample.AuthUserPass.Api | Username/Password (JWT) | Auth.UserPass | Entity + repository + manager pattern for JWT auth |
| Sample.AuthRole.Api | JWT + Role-based | Auth.UserPass, Auth.Role | `[LSCoreAuthRole<TRole>]` attribute |
| Sample.AuthPermission.Api | JWT + Permission-based | Auth.UserPass, Auth.Permission | `[LSCoreAuthPermission<TPermission>]` with AND/OR logic |
| Sample.AuthCombined.Api | API Key + JWT (layered) | Auth.Key, Auth.UserPass | `BreakOnFailedAuth = false` for auth fallthrough |
| Sample.Validation.Api | N/A | Validation, DependencyInjection | `LSCoreValidatorBase<T>`, `.Validate()`, enum codes |
| Sample.ValidationWithRepository.Api | N/A | Validation, DependencyInjection | Validator with repository injection |
| Sample.Mapper.Api | N/A | Mapper, DependencyInjection | `ILSCoreMapper<,>`, `.ToMapped()`, `.ToMappedList()` |
| Sample.SortAndPage.Api | N/A | SortAndPage, Mapper, DependencyInjection | `ToSortedAndPagedResponse` on `IQueryable<T>` |

---

## Common Patterns Across Samples

### Exception Handling Middleware

Every sample registers the exception handler:

```csharp
app.UseLSCoreExceptionsHandler();
```

This converts LSCore exceptions (e.g. `LSCoreNotFoundException`) into proper HTTP responses automatically.

### Convention-Based DI

Samples that use mapping, validation, or custom services use:

```csharp
builder.AddLSCoreDependencyInjection("AssemblyPrefix");
app.UseLSCoreDependencyInjection();
```

This scans assemblies whose names start with the given prefix and auto-registers services, mappers, and validators.

### Controller Primary Constructors

All sample controllers use C# primary constructor syntax for dependency injection:

```csharp
public class UsersController(ILSCoreAuthUserPassManager<string> authPasswordManager)
    : ControllerBase
{
    // authPasswordManager is available directly
}
```

### Builder / App Pattern

Every sample follows the same two-phase pattern:

1. **Builder phase** -- register services with `builder.AddLSCore*()` methods
2. **App phase** -- activate middleware with `app.UseLSCore*()` methods
