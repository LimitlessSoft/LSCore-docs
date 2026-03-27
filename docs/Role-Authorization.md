---
layout: default
title: Role Authorization
parent: Auth Overview
nav_order: 3
---

# Role-Based Authorization

Role-based authorization restricts access to endpoints based on the authenticated user's role. Roles are defined as a C# enum. Each user entity has a single role, and endpoints specify which roles are permitted.

Role authorization is an **authorization** layer -- it requires the user to already be **authenticated** (typically via UserPass/JWT). Key-authenticated requests bypass role checks entirely.

## How It Works

1. You define your roles as an enum.
2. Your user entity implements `ILSCoreAuthRoleEntity<TEntityIdentifier, TRole>`, exposing a `Role` property.
3. You decorate endpoints with `LSCoreAuthRoleAttribute<TRole>`, listing the allowed roles.
4. The `LSCoreAuthRoleMiddleware` intercepts requests:
   - If the endpoint has no `LSCoreAuthRoleAttribute`, the request passes through.
   - If the caller is Key-authenticated (`LSCoreAuthEntityType.Key`), the request passes through.
   - If the caller is not authenticated, `LSCoreUnauthenticatedException` is thrown (401).
   - Otherwise, the middleware calls `ILSCoreAuthRoleManager.IsInRole()` to check if the user's role matches any of the required roles.
   - If the role check fails, `LSCoreForbiddenException` is thrown (403).

## Defining Roles

Define your roles as a standard C# enum:

```csharp
public enum AppRole
{
    Member,
    Moderator,
    Admin
}
```

## Implementing the User Entity

Your user entity must implement `ILSCoreAuthRoleEntity<TEntityIdentifier, TRole>`:

```csharp
public interface ILSCoreAuthRoleEntity<out TEntityIdentifier, TRole>
    : ILSCoreAuthEntity<TEntityIdentifier>
    where TRole : Enum
{
    TRole Role { get; set; }
}
```

### Example

```csharp
using LSCore.Auth.Role.Contracts;

public class User : ILSCoreAuthRoleEntity<Guid, AppRole>
{
    public Guid Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public AppRole Role { get; set; } = AppRole.Member;

    // ILSCoreAuthEntity<Guid>
    public Guid Identifier => Id;
}
```

Note: If you are also using UserPass auth, your entity can implement both interfaces:

```csharp
public class User
    : ILSCoreAuthUserPassEntity<Guid>,
      ILSCoreAuthRoleEntity<Guid, AppRole>
{
    public Guid Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string? RefreshToken { get; set; }
    public AppRole Role { get; set; } = AppRole.Member;

    public Guid Identifier => Id;
}
```

## Implementing the Repository

Your repository must implement `ILSCoreAuthRoleIdentityEntityRepository<TEntityIdentifier, TRole>`:

```csharp
public interface ILSCoreAuthRoleIdentityEntityRepository<TAuthEntityIdentifier, TRole>
    where TRole : Enum
{
    ILSCoreAuthRoleEntity<TAuthEntityIdentifier, TRole>? GetOrDefault(
        TAuthEntityIdentifier identifier
    );
}
```

### Example

```csharp
using LSCore.Auth.Role.Contracts;

public class UserRoleRepository
    : ILSCoreAuthRoleIdentityEntityRepository<Guid, AppRole>
{
    private readonly AppDbContext _db;

    public UserRoleRepository(AppDbContext db)
    {
        _db = db;
    }

    public ILSCoreAuthRoleEntity<Guid, AppRole>? GetOrDefault(Guid identifier)
    {
        return _db.Users.FirstOrDefault(u => u.Id == identifier);
    }
}
```

## Using LSCoreAuthRoleAttribute

The `LSCoreAuthRoleAttribute<TRole>` is a generic attribute that extends `LSCoreAuthAttribute`. Apply it to endpoints to specify which roles are allowed:

```csharp
// Allow only Admin
[LSCoreAuthRoleAttribute<AppRole>(AppRole.Admin)]
app.MapDelete("/users/{id}", (Guid id) => { /* ... */ });

// Allow Admin or Moderator
[LSCoreAuthRoleAttribute<AppRole>(AppRole.Admin, AppRole.Moderator)]
app.MapPut("/posts/{id}", (Guid id) => { /* ... */ });
```

The attribute accepts `params TRole[] roles`, so you can specify multiple allowed roles. The user needs to have **any one** of the listed roles to pass the check.

## DI Registration

### Standard Registration (Built-In Manager)

The simplest setup uses the built-in `LSCoreAuthRoleManager<TAuthEntityIdentifier, TRole>`:

```csharp
builder.AddLSCoreAuthRole<
    Guid,                  // TAuthEntityIdentifier
    AppRole,               // TRole (your enum)
    UserRoleRepository     // your repository
>();
```

### Custom Manager Registration

If you need custom role-checking logic, provide your own manager implementation:

```csharp
builder.AddLSCoreAuthRole<
    Guid,                  // TAuthEntityIdentifier
    AppRole,               // TRole
    CustomRoleManager,     // your custom ILSCoreAuthRoleManager implementation
    UserRoleRepository     // your repository
>();
```

## Middleware Setup

```csharp
var app = builder.Build();

// Authentication must come before role authorization
app.UseLSCoreAuthUserPass<Guid>();

// Role authorization middleware
app.UseLSCoreAuthRole<Guid, AppRole>();

app.Run();
```

## Complete Example

```csharp
using LSCore.Auth.Contracts;
using LSCore.Auth.Role.Contracts;
using LSCore.Auth.Role.DependencyInjection;
using LSCore.Auth.UserPass.Contracts;
using LSCore.Auth.UserPass.DependencyInjection;
using LSCore.Auth.UserPass.Domain;

public enum AppRole
{
    Member,
    Moderator,
    Admin
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

// Register Role auth
builder.AddLSCoreAuthRole<Guid, AppRole, UserRoleRepository>();

var app = builder.Build();

app.UseLSCoreAuthUserPass<Guid>();
app.UseLSCoreAuthRole<Guid, AppRole>();

// Public endpoint
app.MapGet("/", () => "Hello");

// Any authenticated user
app.MapGet("/profile", [LSCoreAuthAttribute] (
    LSCoreAuthContextEntity<Guid> ctx
) =>
{
    return Results.Ok(new { UserId = ctx.Identifier });
});

// Admin only
app.MapDelete("/users/{id}", [LSCoreAuthRoleAttribute<AppRole>(AppRole.Admin)] (Guid id) =>
{
    return Results.Ok();
});

// Admin or Moderator
app.MapPut("/posts/{id}/approve",
    [LSCoreAuthRoleAttribute<AppRole>(AppRole.Admin, AppRole.Moderator)] (Guid id) =>
{
    return Results.Ok();
});

app.Run();
```
