---
layout: default
title: UserPass Authentication
parent: Auth Overview
nav_order: 2
---

# Username/Password Authentication (UserPass)

UserPass authentication provides credential-based authentication using JWT (JSON Web Tokens). Users authenticate with an identifier (e.g., email, username) and a password. On success, the system returns a pair of JWT tokens (access + refresh). Subsequent requests are authenticated by passing the access token in the standard `Authorization: Bearer <token>` header.

## How It Works

1. The caller sends credentials (identifier + password) to your login endpoint.
2. Your endpoint calls `ILSCoreAuthUserPassManager<TEntityIdentifier>.Authenticate()`.
3. The manager looks up the user via `ILSCoreAuthUserPassIdentityEntityRepository`, verifies the BCrypt-hashed password, generates JWT tokens, and stores the refresh token.
4. The caller receives an `LSCoreJwt` containing `AccessToken` and `RefreshToken`.
5. On subsequent requests, the caller includes `Authorization: Bearer <AccessToken>`.
6. ASP.NET Core's built-in JWT Bearer authentication validates the token.
7. The `LSCoreAuthUserPassMiddleware` reads the validated JWT claims and populates `LSCoreAuthContextEntity<TEntityIdentifier>` with the user's identifier and entity type.

## Configuration

### LSCoreAuthUserPassConfiguration

Extends `LSCoreAuthConfiguration` with JWT-specific settings:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `SecurityKey` | `string` | *required* | The symmetric key used to sign and validate JWTs. Must be long enough for HMAC-SHA256. |
| `Issuer` | `string` | *required* | The JWT issuer claim (`iss`). Typically your application name or URL. |
| `Audience` | `string` | *required* | The JWT audience claim (`aud`). Typically your API URL or client identifier. |
| `AccessTokenExpirationMinutes` | `int` | `30` | How long access tokens are valid, in minutes. |
| `RefreshTokenExpirationDays` | `int` | `7` | How long refresh tokens are valid, in days. |
| `TokenSkew` | `TimeSpan` | `5 minutes` | Clock skew tolerance for token validation. |
| `PasswordHashWorkFactor` | `int` | `12` | BCrypt work factor used by `LSCoreAuthUserPassHelpers.HashPassword`. Each increment doubles hashing and verification time. |
| `AllowLegacyBCryptHashes` | `bool` | `false` | When `true`, passwords stored as plain `BCrypt.HashPassword` hashes are accepted on login (and logged as a warning) so an existing user table can be migrated. See [Password Hashing](#password-hashing). |
| `AuthAll` | `bool` | `false` | When `true`, every request requires authentication. When `false`, only endpoints with `[LSCoreAuthAttribute]` are checked. |
| `BreakOnFailedAuth` | `bool` | `true` | When `true`, unauthenticated requests to protected endpoints throw `LSCoreUnauthenticatedException`. |

## Implementing the User Entity

Your user entity must implement `ILSCoreAuthUserPassEntity<TEntityIdentifier>`:

```csharp
public interface ILSCoreAuthUserPassEntity<out TEntityIdentifier>
    : ILSCoreAuthEntity<TEntityIdentifier>
{
    public string? RefreshToken { get; set; }
    public string Password { get; set; }
}
```

### Example

```csharp
using LSCore.Auth.UserPass.Contracts;

public class User : ILSCoreAuthUserPassEntity<Guid>
{
    public Guid Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string? RefreshToken { get; set; }

    // ILSCoreAuthEntity<Guid>
    public Guid Identifier => Id;
}
```

## Implementing the Repository

Your repository must implement `ILSCoreAuthUserPassIdentityEntityRepository<TEntityIdentifier>`:

```csharp
public interface ILSCoreAuthUserPassIdentityEntityRepository<TEntityIdentifier>
{
    ILSCoreAuthUserPassEntity<TEntityIdentifier>? GetOrDefault(TEntityIdentifier identifier);
    void SetRefreshToken(TEntityIdentifier entityIdentifier, string refreshToken);
}
```

### Example

```csharp
using LSCore.Auth.UserPass.Contracts;

public class UserRepository : ILSCoreAuthUserPassIdentityEntityRepository<Guid>
{
    private readonly AppDbContext _db;

    public UserRepository(AppDbContext db)
    {
        _db = db;
    }

    public ILSCoreAuthUserPassEntity<Guid>? GetOrDefault(Guid identifier)
    {
        return _db.Users.FirstOrDefault(u => u.Id == identifier);
    }

    public void SetRefreshToken(Guid entityIdentifier, string refreshToken)
    {
        var user = _db.Users.First(u => u.Id == entityIdentifier);
        user.RefreshToken = refreshToken;
        _db.SaveChanges();
    }
}
```

## JWT Handling

### Token Structure

Each JWT contains the following claims:

| Claim | Description |
|-------|-------------|
| `sub` | The entity identifier (e.g., user ID) |
| `jti` | A unique token identifier (GUID) |
| `ls:identifier` | The entity identifier (custom claim used by LSCore to populate the auth context) |

Tokens are signed with HMAC-SHA256 using the `SecurityKey` from configuration.

### Password Hashing

Passwords are hashed using BCrypt **Enhanced** hashing with a work factor of 12. Use the provided helper to hash passwords before storing them:

```csharp
using LSCore.Auth.UserPass.Domain;

var hashedPassword = LSCoreAuthUserPassHelpers.HashPassword("raw-password");
```

To use a different work factor, pass it explicitly or pass your configuration:

```csharp
LSCoreAuthUserPassHelpers.HashPassword("raw-password", 14);
LSCoreAuthUserPassHelpers.HashPassword("raw-password", configuration);
```

Raising the work factor is safe for existing users: BCrypt stores the cost inside the hash, so hashes written with an older, lower work factor keep verifying.

> **⚠️ Hashes must come from `LSCoreAuthUserPassHelpers.HashPassword`, not from `BCrypt.HashPassword`.**
>
> LSCore uses BCrypt's `Enhanced*` methods, which pre-hash the password with SHA-384. A hash from plain `BCrypt.HashPassword` looks identical (`$2a$NN$...`) but will **never** verify against `EnhancedVerify`:
>
> ```csharp
> BCrypt.Net.BCrypt.EnhancedVerify(pw, LSCoreAuthUserPassHelpers.HashPassword(pw)); // true
> BCrypt.Net.BCrypt.EnhancedVerify(pw, BCrypt.Net.BCrypt.HashPassword(pw));         // false
> ```

#### Migrating a table hashed with plain BCrypt

If your users' passwords were hashed with plain `BCrypt.HashPassword`, every login fails with `LSCoreForbiddenException` -- the same result as a wrong password. `Authenticate` detects this case: when `EnhancedVerify` fails it retries with plain `BCrypt.Verify`, and if that succeeds it logs a warning naming the user and explaining the mismatch.

The login is still rejected unless you opt in:

```csharp
new LSCoreAuthUserPassConfiguration
{
    // ...
    AllowLegacyBCryptHashes = true
}
```

With the option on, plain BCrypt hashes are accepted and each such login is logged as a warning. Rehash those passwords with `LSCoreAuthUserPassHelpers.HashPassword` as users log in, then turn the option back off.

### Auth Context

After middleware execution, the authenticated user's identity is available through `LSCoreAuthContextEntity<TEntityIdentifier>`, which is registered as a scoped service:

```csharp
public class LSCoreAuthContextEntity<TEntityIdentifier>
{
    public bool IsAuthenticated => Type != null && Identifier != null;
    public bool NotAuthenticated => !IsAuthenticated;
    public LSCoreAuthEntityType? Type { get; set; }
    public TEntityIdentifier? Identifier { get; set; }
}
```

For JWT-authenticated users, `Type` is set to `LSCoreAuthEntityType.User`.

## DI Registration

Register UserPass auth in `Program.cs`:

```csharp
builder.AddLSCoreAuthUserPass<
    Guid,                   // TEntityIdentifier -- your entity's ID type
    AuthManager,            // TAuthPasswordManager -- your manager (or use the built-in one)
    UserRepository          // TLSCoreIdentityRepository -- your repository
>(new LSCoreAuthUserPassConfiguration
{
    SecurityKey = "your-256-bit-secret-key-here-min-32-chars",
    Issuer = "MyApp",
    Audience = "MyAppUsers",
    AccessTokenExpirationMinutes = 60,
    RefreshTokenExpirationDays = 30
});
```

### Using the Built-In Manager

If the default authentication flow (BCrypt verification, JWT generation) is sufficient, use the built-in `LSCoreAuthUserPassManager<TEntityIdentifier>`:

```csharp
using LSCore.Auth.UserPass.Domain;

builder.AddLSCoreAuthUserPass<
    Guid,
    LSCoreAuthUserPassManager<Guid>,
    UserRepository
>(new LSCoreAuthUserPassConfiguration
{
    SecurityKey = "your-256-bit-secret-key-here-min-32-chars",
    Issuer = "MyApp",
    Audience = "MyAppUsers"
});
```

## Middleware Setup

```csharp
var app = builder.Build();

// UserPass middleware (adds ASP.NET Authentication + custom context population)
app.UseLSCoreAuthUserPass<Guid>();

app.Run();
```

The `UseLSCoreAuthUserPass<TEntityIdentifier>()` call registers two middlewares in sequence:
1. `app.UseAuthentication()` -- ASP.NET Core's built-in JWT Bearer validation
2. `LSCoreAuthUserPassMiddleware<TEntityIdentifier>` -- Reads JWT claims and populates `LSCoreAuthContextEntity`

## Complete Example

### Program.cs

```csharp
using LSCore.Auth.Contracts;
using LSCore.Auth.UserPass.Contracts;
using LSCore.Auth.UserPass.DependencyInjection;
using LSCore.Auth.UserPass.Domain;

var builder = WebApplication.CreateBuilder(args);

builder.AddLSCoreAuthUserPass<
    Guid,
    LSCoreAuthUserPassManager<Guid>,
    UserRepository
>(new LSCoreAuthUserPassConfiguration
{
    SecurityKey = "a-very-long-secret-key-at-least-32-characters",
    Issuer = "MyApplication",
    Audience = "MyApplicationUsers",
    AccessTokenExpirationMinutes = 60,
    RefreshTokenExpirationDays = 14
});

var app = builder.Build();

app.UseLSCoreAuthUserPass<Guid>();

// Login endpoint (public)
app.MapPost("/login", (
    LoginRequest request,
    ILSCoreAuthUserPassManager<Guid> authManager
) =>
{
    var jwt = authManager.Authenticate(request.UserId, request.Password);
    return Results.Ok(jwt);
});

// Protected endpoint
app.MapGet("/me", [LSCoreAuthAttribute] (
    LSCoreAuthContextEntity<Guid> authContext
) =>
{
    return Results.Ok(new { UserId = authContext.Identifier });
});

app.Run();

record LoginRequest(Guid UserId, string Password);
```

### Registering a User (Password Hashing)

```csharp
using LSCore.Auth.UserPass.Domain;

var user = new User
{
    Id = Guid.NewGuid(),
    Email = "user@example.com",
    Password = LSCoreAuthUserPassHelpers.HashPassword("my-secure-password")
};
dbContext.Users.Add(user);
dbContext.SaveChanges();
```
