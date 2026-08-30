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

Both packages target **.NET 9.0** (version 9.1.5 at time of writing).

## Exception Types

All exception types live in the `LSCore.Exceptions` namespace, extend `System.Exception`, and offer two constructors: a parameterless constructor and one that accepts a `string message`.

| Exception Class | HTTP Status Code | Typical Use |
|---|---|---|
| `LSCoreBadRequestException` | 400 Bad Request | The client sent invalid or malformed input. |
| `LSCoreUnauthenticatedException` | 401 Unauthorized | The request lacks valid authentication credentials. |
| `LSCoreForbiddenException` | 403 Forbidden | The authenticated user does not have permission to access the requested resource. |
| `LSCoreNotFoundException` | 404 Not Found | The requested resource does not exist. |
| `LSCoreInternalException` | 500 Internal Server Error | An unexpected server-side error occurred. Handled exactly like any unrecognized exception. |

### Detailed Behavior Notes

- The four **4xx** exceptions write their `Message` to the response body, so you can surface the reason to the caller.
- The body is sent with a `Content-Type`: `application/json; charset=utf-8` when the message is a JSON object or array, `text/plain; charset=utf-8` otherwise. This is what makes validation errors parseable -- `Validate()` throws its FluentValidation errors as a serialized JSON array, and typed HTTP clients dispatch on the content type.
- An empty message produces an empty body and no `Content-Type`.
- **LSCoreInternalException** (and any other unhandled exception) results in a 500 with an **empty body**. The message is logged via `ILogger` at the `Error` level but never returned, so internal detail cannot leak to the caller. The 4xx paths are not logged -- they are expected outcomes, not failures.
- An `AggregateException` with a single inner exception (from `Task.WhenAll`, `.Result` or `.Wait()`) and a `TargetInvocationException` (from reflection) are unwrapped first, so a wrapped `LSCoreNotFoundException` still returns 404. An `AggregateException` carrying several exceptions has no single status code to map to and becomes a 500.
- If the client disconnects mid-request, the resulting `OperationCanceledException` is not treated as an error: it is logged at `Debug` and the response is left alone.
- If an exception is thrown **after** the response has started, the status code can no longer be changed. The exception is logged and rethrown, which aborts the connection so the client sees a truncated response instead of an apparently successful one.

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

1. Returns immediately (logging at `Debug`) if the client has disconnected, or rethrows if the response has already started.
2. Unwraps single-exception `AggregateException`s and `TargetInvocationException`s.
3. Matches the exception type using a `switch`.
4. Sets `context.Response.StatusCode` to the corresponding HTTP status code.
5. For the four 4xx types, writes `exception.Message` to the response body with `application/json` or `text/plain` depending on whether the message is JSON.
6. For `LSCoreInternalException` and any other exception, logs at `Error` level and returns 500 with an empty body.
