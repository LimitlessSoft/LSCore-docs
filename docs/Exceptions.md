---
layout: default
title: Exceptions
nav_order: 7
---

# LSCore Exceptions

The LSCore Exceptions module provides a structured set of exception types that map directly to HTTP status codes. When paired with the exception-handling middleware, thrown exceptions are automatically translated into the appropriate HTTP responses, removing the need for repetitive status-code logic in controllers and services.

## Why Exceptions Over Wrapper Classes

There are two common approaches to error handling in APIs:

| Approach | Speed | Complexity | Safety |
|---|---|---|---|
| Wrapper classes | Faster (1-5 ns/request) | Complex | Possible leakage when any part of code throws an unexpected exception |
| Exceptions (LSCore approach) | Slower (1-5 ns/request) | Simple to implement | All unhandled exceptions are converted to 500 without any message leakage |

LSCore uses the exception-based approach because the performance difference is negligible while the simplicity and safety benefits are significant. You throw from anywhere, and the middleware translates it to the correct HTTP response.

## NuGet Packages

| Package | Description |
|---|---|
| `LSCore.Exceptions` | Core exception classes. No ASP.NET Core dependency. |
| `LSCore.Exceptions.DependencyInjection` | ASP.NET Core middleware that catches LSCore exceptions and sets the HTTP response status code. Depends on `LSCore.Exceptions` and the `Microsoft.AspNetCore.App` framework reference. |

Both packages target **.NET 9.0** (version 9.1.4.1 at time of writing).

## Exception Types

All exception types live in the `LSCore.Exceptions` namespace, extend `System.Exception`, and offer two constructors: a parameterless constructor and one that accepts a `string message`.

| Exception Class | HTTP Status Code | Typical Use |
|---|---|---|
| `LSCoreBadRequestException` | 400 Bad Request | The client sent invalid or malformed input. The exception message is written to the response body so the caller knows what was wrong. |
| `LSCoreUnauthenticatedException` | 401 Unauthorized | The request lacks valid authentication credentials. |
| `LSCoreForbiddenException` | 403 Forbidden | The authenticated user does not have permission to access the requested resource. |
| `LSCoreNotFoundException` | 404 Not Found | The requested resource does not exist. |
| `LSCoreInternalException` | 500 Internal Server Error | An unexpected server-side error occurred. Handled by the middleware's default branch, same as any unrecognized exception. |

### Detailed Behavior Notes

- **LSCoreBadRequestException** is the only exception whose `Message` is written back to the HTTP response body. This lets you surface validation details to the caller.
- **LSCoreUnauthenticatedException** and **LSCoreForbiddenException** set the status code only; no body is written.
- **LSCoreNotFoundException** sets the status code only; no body is written.
- **LSCoreInternalException** (and any other unhandled exception) results in a 500 status code. The exception is also logged via `ILogger` at the `Error` level.

## Setting Up the Middleware

### 1. Install both packages

Add references to `LSCore.Exceptions` and `LSCore.Exceptions.DependencyInjection` in your project.

### 2. Register the middleware

In your `Program.cs` (or wherever you configure the ASP.NET Core pipeline), call the extension method on `WebApplication`:

```csharp
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
// ... service registration ...

var app = builder.Build();

app.UseLSCoreExceptionsHandler();

// ... other middleware, routing, etc. ...
app.Run();
```

`UseLSCoreExceptionsHandler()` inserts `LSCoreExceptionsHandleMiddleware` into the request pipeline. Place it early enough to catch exceptions from downstream middleware and endpoints.

## Code Examples

### Throwing exceptions in a service

```csharp
using LSCore.Exceptions;

public class OrderService
{
    public Order GetOrder(int id)
    {
        var order = _repository.Find(id);

        if (order is null)
            throw new LSCoreNotFoundException("Order not found.");

        return order;
    }

    public void PlaceOrder(OrderRequest request)
    {
        if (string.IsNullOrWhiteSpace(request.ProductName))
            throw new LSCoreBadRequestException("ProductName is required.");

        // ... business logic ...
    }
}
```

### Guarding authorization

```csharp
using LSCore.Exceptions;

public class DocumentService
{
    public Document GetDocument(int documentId, User currentUser)
    {
        if (currentUser is null)
            throw new LSCoreUnauthenticatedException();

        var document = _repository.Find(documentId);

        if (document is null)
            throw new LSCoreNotFoundException();

        if (document.OwnerId != currentUser.Id)
            throw new LSCoreForbiddenException();

        return document;
    }
}
```

### Signaling an internal failure

```csharp
using LSCore.Exceptions;

public class PaymentService
{
    public void ProcessPayment(PaymentRequest request)
    {
        var result = _gateway.Charge(request);

        if (!result.Success)
            throw new LSCoreInternalException("Payment gateway returned an error.");
    }
}
```

## How the Middleware Handles Each Exception

When an exception propagates out of the request pipeline, `LSCoreExceptionsHandleMiddleware` catches it and performs the following:

1. Matches the exception type using a `switch` expression.
2. Sets `context.Response.StatusCode` to the corresponding HTTP status code.
3. For `LSCoreBadRequestException` only, writes `exception.Message` to the response body.
4. For any exception that does not match a known LSCore type (the `default` branch), logs the exception at `Error` level and returns 500.
