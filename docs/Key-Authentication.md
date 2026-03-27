---
layout: default
title: Key Authentication
parent: Auth Overview
nav_order: 1
---

# Key-Based Authentication

Key-based authentication validates requests by checking an API key sent in a custom HTTP header. It is the simplest auth mechanism in LSCore and is well-suited for machine-to-machine communication or service integrations where a shared secret is acceptable.

## How It Works

1. The caller includes a key in the `X-LS-Key` HTTP header.
2. The `LSCoreAuthKeyMiddleware` intercepts the request.
3. If the endpoint requires authentication (via `LSCoreAuthAttribute` or the `AuthAll` configuration flag), the middleware validates the key using your `ILSCoreAuthKeyProvider` implementation.
4. On success, the request's `ClaimsPrincipal` is set to an authenticated identity of type `"ApiKey"`.
5. On failure, an `LSCoreUnauthenticatedException` is thrown (returns HTTP 401).

Key-authenticated requests are treated as `LSCoreAuthEntityType.Key`. This means they **bypass role and permission authorization checks entirely** -- the Role and Permission middlewares will pass them through without verification.

## Configuration

### LSCoreAuthKeyConfiguration

Extends `LSCoreAuthConfiguration` and provides:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `AuthAll` | `bool` | `false` | When `true`, every request requires key authentication. When `false`, only endpoints decorated with `[LSCoreAuthAttribute]` are checked. |
| `BreakOnFailedAuth` | `bool` | `true` | When `true`, missing or invalid keys throw `LSCoreUnauthenticatedException`. When `false`, unauthenticated requests are allowed through. |
| `ValidKeys` | `HashSet<string>?` | `null` | **Obsolete.** A static set of valid keys. Use `ILSCoreAuthKeyProvider` instead. |

### Headers

The key is read from the custom HTTP header defined in `LSCoreAuthKeyHeaders`:

```csharp
public static class LSCoreAuthKeyHeaders
{
    public const string KeyCustomHeader = "X-LS-Key";
}
```

Callers must include:
```
X-LS-Key: <your-api-key>
```

## Implementing ILSCoreAuthKeyProvider

To validate keys, implement the `ILSCoreAuthKeyProvider` interface:

```csharp
public interface ILSCoreAuthKeyProvider
{
    bool IsValidKey(string key);
}
```

### Example: Database-Backed Key Provider

```csharp
using LSCore.Auth.Key.Contracts;

public class DatabaseKeyProvider : ILSCoreAuthKeyProvider
{
    private readonly IApiKeyRepository _repository;

    public DatabaseKeyProvider(IApiKeyRepository repository)
    {
        _repository = repository;
    }

    public bool IsValidKey(string key)
    {
        return _repository.Exists(key);
    }
}
```

### Example: Configuration-Backed Key Provider

```csharp
using LSCore.Auth.Key.Contracts;

public class ConfigKeyProvider : ILSCoreAuthKeyProvider
{
    private readonly HashSet<string> _validKeys;

    public ConfigKeyProvider(IConfiguration configuration)
    {
        _validKeys = configuration
            .GetSection("Auth:ApiKeys")
            .Get<HashSet<string>>() ?? new HashSet<string>();
    }

    public bool IsValidKey(string key)
    {
        return _validKeys.Contains(key);
    }
}
```

## DI Registration

Register Key auth in `Program.cs` using the extension methods on `WebApplicationBuilder`:

### Recommended: Generic Provider Registration

```csharp
// With default configuration (AuthAll = false, BreakOnFailedAuth = true)
builder.AddLSCoreAuthKey<DatabaseKeyProvider>();

// With custom configuration
builder.AddLSCoreAuthKey<DatabaseKeyProvider>(new LSCoreAuthKeyConfiguration
{
    AuthAll = true,
    BreakOnFailedAuth = true
});
```

### Obsolete: Static Key Set

The following registration method is obsolete and exists only for backward compatibility. It uses a `DummyLSCoreAuthKeyProvider` internally and relies on the `ValidKeys` property:

```csharp
// Do not use -- prefer the generic provider approach above
#pragma warning disable CS0618
builder.AddLSCoreAuthKey(new LSCoreAuthKeyConfiguration
{
    ValidKeys = new HashSet<string> { "key-1", "key-2" }
});
#pragma warning restore CS0618
```

## Middleware Setup

After registering services, add the middleware to the application pipeline:

```csharp
var app = builder.Build();
app.UseLSCoreAuthKey();
// ... other middleware
app.Run();
```

The Key middleware should be placed **before** UserPass middleware in the pipeline. If a request is already authenticated (e.g., by a JWT), the Key middleware skips validation and passes the request through.

## Complete Example

```csharp
using LSCore.Auth.Key.Contracts;
using LSCore.Auth.Key.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Register key auth with your provider
builder.AddLSCoreAuthKey<DatabaseKeyProvider>();

var app = builder.Build();

// Add key auth middleware
app.UseLSCoreAuthKey();

app.MapGet("/public", () => "No auth needed");

app.MapGet("/secure", [LSCoreAuthAttribute] () => "Key required");

app.Run();
```

### Sending a Request

```bash
# Authenticated request
curl -H "X-LS-Key: my-valid-key" https://localhost:5001/secure

# Unauthenticated request (returns 401)
curl https://localhost:5001/secure
```
