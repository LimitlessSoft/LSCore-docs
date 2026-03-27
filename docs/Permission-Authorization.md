---
layout: default
title: Permission Authorization
parent: Auth Overview
nav_order: 4
---

# Permission-Based Authorization

Permission-based authorization restricts access to endpoints based on a set of granular permissions assigned to each user. Permissions are defined as a C# enum. Each user entity holds a collection of permissions, and endpoints declare which permissions are required -- with the option to require all of them or just one.

Permission authorization is an **authorization** layer -- it requires the user to already be **authenticated** (typically via UserPass/JWT). Key-authenticated requests bypass permission checks entirely.

## How It Works

1. You define your permissions as an enum.
2. Your user entity implements `ILSCoreAuthPermissionEntity<TEntityIdentifier, TPermission>`, exposing a `Permissions` collection.
3. You decorate endpoints with `LSCoreAuthPermissionAttribute<TPermission>`, listing the required permissions and optionally specifying whether all are required.
4. The `LSCoreAuthPermissionMiddleware` intercepts requests:
   - If the endpoint has no `LSCoreAuthPermissionAttribute`, the request passes through.
   - If the caller is Key-authenticated (`LSCoreAuthEntityType.Key`), the request passes through.
   - If the caller is not authenticated, `LSCoreUnauthenticatedException` is thrown (401).
   - Otherwise, the middleware calls `ILSCoreAuthPermissionManager.HasPermission()` to check the user's permissions.
   - If the check fails, `LSCoreForbiddenException` is thrown (403).

## Defining Permissions

Define your permissions as a standard C# enum:

```csharp
public enum AppPermission
{
    ReadUsers,
    CreateUsers,
    UpdateUsers,
    DeleteUsers,
    ManageSettings,
    ViewReports
}
```

## Implementing the User Entity

Your user entity must implement `ILSCoreAuthPermissionEntity<TEntityIdentifier, TPermission>`:

```csharp
public interface ILSCoreAuthPermissionEntity<out TEntityIdentifier, TPermission>
    : ILSCoreAuthEntity<TEntityIdentifier>
    where TPermission : Enum
{
    ICollection<TPermission> Permissions { get; set; }
}
```

### Example

```csharp
using LSCore.Auth.Permission.Contracts;

public class User : ILSCoreAuthPermissionEntity<Guid, AppPermission>
{
    public Guid Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public ICollection<AppPermission> Permissions { get; set; } = new List<AppPermission>();

    // ILSCoreAuthEntity<Guid>
    public Guid Identifier => Id;
}
```

If you are using multiple auth modules, your entity can implement multiple interfaces:

```csharp
public class User
    : ILSCoreAuthUserPassEntity<Guid>,
      ILSCoreAuthPermissionEntity<Guid, AppPermission>
{
    public Guid Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string? RefreshToken { get; set; }
    public ICollection<AppPermission> Permissions { get; set; } = new List<AppPermission>();

    public Guid Identifier => Id;
}
```

## Implementing the Repository

Your repository must implement `ILSCoreAuthPermissionIdentityEntityRepository<TEntityIdentifier, TPermission>`:

```csharp
public interface ILSCoreAuthPermissionIdentityEntityRepository<TEntityIdentifier, TPermission>
    where TPermission : Enum
{
    ILSCoreAuthPermissionEntity<TEntityIdentifier, TPermission>? GetOrDefault(
        TEntityIdentifier identifier
    );
}
```

### Example

```csharp
using LSCore.Auth.Permission.Contracts;

public class UserPermissionRepository
    : ILSCoreAuthPermissionIdentityEntityRepository<Guid, AppPermission>
{
    private readonly AppDbContext _db;

    public UserPermissionRepository(AppDbContext db)
    {
        _db = db;
    }

    public ILSCoreAuthPermissionEntity<Guid, AppPermission>? GetOrDefault(Guid identifier)
    {
        return _db.Users
            .Include(u => u.Permissions)
            .FirstOrDefault(u => u.Id == identifier);
    }
}
```

## Using LSCoreAuthPermissionAttribute

The `LSCoreAuthPermissionAttribute<TPermission>` is a generic attribute that extends `LSCoreAuthAttribute`. It has two constructors:

### Require ALL Permissions (Default)

```csharp
// User must have BOTH ReadUsers AND ViewReports
[LSCoreAuthPermissionAttribute<AppPermission>(AppPermission.ReadUsers, AppPermission.ViewReports)]
app.MapGet("/admin/user-reports", () => { /* ... */ });
```

By default, `RequireAll` is `true`, so the user must have every listed permission.

### Require ANY Permission

```csharp
// User needs at least ONE of: UpdateUsers OR ManageSettings
[LSCoreAuthPermissionAttribute<AppPermission>(false, AppPermission.UpdateUsers, AppPermission.ManageSettings)]
app.MapPut("/settings", () => { /* ... */ });
```

Pass `false` as the first argument to switch to "any" mode.

### Validation

At least one permission must be specified. Passing an empty permission list throws `ArgumentException` at attribute construction time.

## DI Registration

### Standard Registration (Built-In Manager)

The simplest setup uses the built-in `LSCoreAuthPermissionManager<TAuthEntityIdentifier, TPermission>`:

```csharp
builder.AddLSCoreAuthPermission<
    Guid,                        // TAuthEntityIdentifier
    AppPermission,               // TPermission (your enum)
    UserPermissionRepository     // your repository
>();
```

### Custom Manager Registration

If you need custom permission-checking logic, provide your own manager:

```csharp
builder.AddLSCoreAuthPermission<
    Guid,                        // TAuthEntityIdentifier
    AppPermission,               // TPermission
    CustomPermissionManager,     // your ILSCoreAuthPermissionManager implementation
    UserPermissionRepository     // your repository
>();
```

## Middleware Setup

```csharp
var app = builder.Build();

// Authentication must come before permission authorization
app.UseLSCoreAuthUserPass<Guid>();

// Permission authorization middleware
app.UseLSCoreAuthPermission<Guid, AppPermission>();

app.Run();
```

## Complete Example

```csharp
using LSCore.Auth.Contracts;
using LSCore.Auth.Permission.Contracts;
using LSCore.Auth.Permission.DependencyInjection;
using LSCore.Auth.UserPass.Contracts;
using LSCore.Auth.UserPass.DependencyInjection;
using LSCore.Auth.UserPass.Domain;

public enum AppPermission
{
    ReadUsers,
    CreateUsers,
    UpdateUsers,
    DeleteUsers,
    ManageSettings
}

var builder = WebApplication.CreateBuilder(args);

// Register UserPass auth
builder.AddLSCoreAuthUserPass<
    Guid,
    LSCoreAuthUserPassManager<Guid>,
    UserRepository
>(new LSCoreAuthUserPassConfiguration
{
    SecurityKey = "a-very-long-secret-key-at-least-32-characters",
    Issuer = "MyApp",
    Audience = "MyAppUsers"
});

// Register Permission auth
builder.AddLSCoreAuthPermission<Guid, AppPermission, UserPermissionRepository>();

var app = builder.Build();

app.UseLSCoreAuthUserPass<Guid>();
app.UseLSCoreAuthPermission<Guid, AppPermission>();

// Public
app.MapGet("/", () => "Hello");

// Any authenticated user
app.MapGet("/profile", [LSCoreAuthAttribute] () => "Authenticated");

// Requires ReadUsers permission
app.MapGet("/users",
    [LSCoreAuthPermissionAttribute<AppPermission>(AppPermission.ReadUsers)] () =>
{
    return Results.Ok();
});

// Requires BOTH CreateUsers AND ManageSettings
app.MapPost("/users",
    [LSCoreAuthPermissionAttribute<AppPermission>(AppPermission.CreateUsers, AppPermission.ManageSettings)] () =>
{
    return Results.Ok();
});

// Requires EITHER UpdateUsers OR ManageSettings
app.MapPut("/users/{id}",
    [LSCoreAuthPermissionAttribute<AppPermission>(false, AppPermission.UpdateUsers, AppPermission.ManageSettings)] (Guid id) =>
{
    return Results.Ok();
});

app.Run();
```

## Permission Check Logic

The built-in `LSCoreAuthPermissionManager` implements the following logic:

- **`requireAll = true`**: Returns `true` only if the user has **every** listed permission. Short-circuits to `false` on the first missing permission.
- **`requireAll = false`**: Returns `true` if the user has **at least one** of the listed permissions. Short-circuits to `true` on the first match.
- If the user is not found in the repository, returns `false` regardless of mode.
