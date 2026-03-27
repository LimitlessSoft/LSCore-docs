---
layout: default
title: Logging
nav_order: 12
---

# LSCore Logging

The LSCore Logging module provides a single extension method that wires up the standard console and debug logging providers for an ASP.NET Core application. It acts as a thin convenience wrapper over the built-in Microsoft logging infrastructure so that all LSCore-based projects configure logging in a consistent way.

## NuGet Package

| Package | Description |
|---|---|
| `LSCore.Logging` | Configures console and debug logging providers on a `WebApplicationBuilder`. Depends on `Microsoft.AspNetCore.App` (framework reference). |

The package targets **.NET 9.0** (version 9.1.4.1 at time of writing). It has no external NuGet dependencies beyond the ASP.NET Core shared framework.

---

## What It Provides

A single extension method on `WebApplicationBuilder`:

```csharp
namespace LSCore.Logging;

public static class Extensions
{
    /// <summary>
    /// Configures the logging system to include console and debug providers.
    /// </summary>
    public static ILoggingBuilder AddLSCoreLogging(this WebApplicationBuilder builder)
    {
        builder.Logging.AddConsole();
        builder.Logging.AddDebug();
        return builder.Logging;
    }
}
```

Calling `AddLSCoreLogging()` registers two logging providers:

- **Console** -- writes structured log output to `stdout`, suitable for container environments and local development.
- **Debug** -- writes log output to the attached debugger's output window (e.g., Visual Studio Output pane).

The method returns the `ILoggingBuilder` so you can chain additional logging configuration if needed.

---

## How to Set Up Logging

### 1. Install the package

Add a reference to `LSCore.Logging` in your ASP.NET Core project.

### 2. Call the extension method

In your `Program.cs`, call `AddLSCoreLogging()` on the `WebApplicationBuilder`:

```csharp
using LSCore.Logging;

var builder = WebApplication.CreateBuilder(args);

builder.AddLSCoreLogging();

// ... other service registrations ...

var app = builder.Build();
app.Run();
```

### 3. (Optional) Chain additional configuration

Because `AddLSCoreLogging()` returns the `ILoggingBuilder`, you can continue configuring logging inline:

```csharp
builder.AddLSCoreLogging()
    .SetMinimumLevel(LogLevel.Warning)
    .AddFilter("Microsoft.EntityFrameworkCore", LogLevel.Error);
```

---

## Configuration

Log levels and filters can be controlled through standard ASP.NET Core mechanisms:

- **appsettings.json** -- set per-category minimum levels under the `"Logging"` section.
- **Environment variables** -- override settings for deployed environments.
- **ILoggingBuilder chaining** -- call `.SetMinimumLevel()`, `.AddFilter()`, or register additional providers on the returned builder.

Example `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

Because `AddLSCoreLogging()` uses the standard Microsoft logging system, all existing documentation on `ILogger<T>`, logging scopes, and structured logging applies without modification.

---

## Code Examples

### Basic setup in Program.cs

```csharp
using LSCore.Logging;

var builder = WebApplication.CreateBuilder(args);

builder.AddLSCoreLogging();
builder.Services.AddControllers();

var app = builder.Build();
app.MapControllers();
app.Run();
```

### Using ILogger in a service

Once logging is configured, inject and use `ILogger<T>` as you normally would in any ASP.NET Core application:

```csharp
using Microsoft.Extensions.Logging;

public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public void ProcessOrder(int orderId)
    {
        _logger.LogInformation("Processing order {OrderId}", orderId);
        // ... business logic ...
        _logger.LogInformation("Order {OrderId} processed successfully", orderId);
    }
}
```
