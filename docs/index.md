---
layout: default
title: Home
nav_order: 1
permalink: /
---

# LSCore Documentation

LSCore is a collection of free and open-source .NET 9.0 libraries that simplify building ASP.NET Core APIs. Its core principles are **simplicity** and **abstraction**: features are divided into independent packages so you include only what you need.

All packages are published to [NuGet](https://www.nuget.org/profiles/LimitlessSoft) and share a synchronized version (currently **9.1.4.1**).

---

## Quick Start

1. Install the packages you need from NuGet (see module listing below).
2. Configure services in `Program.cs` using the `Add*()` extension methods.
3. Activate middleware with the `Use*()` extension methods after building the app.

Minimal example with exception handling and dependency injection:

```csharp
using LSCore.DependencyInjection;
using LSCore.Exceptions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

builder.AddLSCoreDependencyInjection("MyProject");

var app = builder.Build();

app.UseLSCoreDependencyInjection();
app.UseLSCoreExceptionsHandler();

app.Run();
```

For hands-on learners, explore the `src/SampleApps` directory which contains working implementations of all features. See the [Samples](Samples.md) page for details.

---

## Modules

### Authentication and Authorization

| Module | Packages | Documentation |
|---|---|---|
| Auth (shared contracts) | `LSCore.Auth.Contracts` | [Auth Overview](Auth-Overview.md) |
| Auth -- API Key | `LSCore.Auth.Key.Contracts`, `LSCore.Auth.Key.DependencyInjection` | [Key Auth](Key-Authentication.md) |
| Auth -- Username/Password (JWT) | `LSCore.Auth.UserPass.Contracts`, `LSCore.Auth.UserPass.Domain`, `LSCore.Auth.UserPass.DependencyInjection` | [Auth Overview](Auth-Overview.md) |
| Auth -- Role-Based | `LSCore.Auth.Role.Contracts`, `LSCore.Auth.Role.Domain`, `LSCore.Auth.Role.DependencyInjection` | [Auth Overview](Auth-Overview.md) |
| Auth -- Permission-Based | `LSCore.Auth.Permission.Contracts`, `LSCore.Auth.Permission.Domain`, `LSCore.Auth.Permission.DependencyInjection` | [Auth Overview](Auth-Overview.md) |

### Core Infrastructure

| Module | Packages | Documentation |
|---|---|---|
| Dependency Injection | `LSCore.DependencyInjection` | Convention-based assembly scanner. Auto-registers services, mappers, and validators. |
| Exception Handling | `LSCore.Exceptions`, `LSCore.Exceptions.DependencyInjection` | [Exceptions](Exceptions.md) |
| Logging | `LSCore.Logging` | [Logging](Logging.md) |

### Data and Mapping

| Module | Packages | Documentation |
|---|---|---|
| Repository | `LSCore.Repository.Contracts`, `LSCore.Repository` | [Repository](Repository.md) |
| Mapper | `LSCore.Mapper.Contracts`, `LSCore.Mapper.Domain` | [Mapper](Mapper.md) |
| Sort and Page | `LSCore.SortAndPage.Contracts`, `LSCore.SortAndPage.Domain` | [Sort and Page](Sort-and-Page.md) |

### Validation

| Module | Packages | Documentation |
|---|---|---|
| Validation | `LSCore.Validation.Contracts`, `LSCore.Validation.Domain` | [Validation](Validation.md) |

### API Client

| Module | Packages | Documentation |
|---|---|---|
| REST API Client | `LSCore.ApiClient.Rest`, `LSCore.ApiClient.Rest.DependencyInjection` | [API Client](API-Client.md) |

### Utilities

| Module | Packages | Documentation |
|---|---|---|
| Common Contracts | `LSCore.Common.Contracts` | [Common](Common.md) |
| Common Extensions | `LSCore.Common.Extensions` | [Common](Common.md) |

---

## All NuGet Packages

| Package | Description |
|---|---|
| `LSCore.ApiClient.Rest` | Abstract base class for REST API clients with status-code mapping |
| `LSCore.ApiClient.Rest.DependencyInjection` | DI registration for the REST API client |
| `LSCore.Auth.Contracts` | Shared authentication interfaces and types |
| `LSCore.Auth.Key.Contracts` | API-key authentication interfaces |
| `LSCore.Auth.Key.DependencyInjection` | API-key authentication middleware and DI wiring |
| `LSCore.Auth.Permission.Contracts` | Permission-based authorization interfaces |
| `LSCore.Auth.Permission.Domain` | Permission-based authorization implementation |
| `LSCore.Auth.Permission.DependencyInjection` | Permission-based authorization middleware and DI wiring |
| `LSCore.Auth.Role.Contracts` | Role-based authorization interfaces |
| `LSCore.Auth.Role.Domain` | Role-based authorization implementation |
| `LSCore.Auth.Role.DependencyInjection` | Role-based authorization middleware and DI wiring |
| `LSCore.Auth.UserPass.Contracts` | Username/password (JWT) authentication interfaces |
| `LSCore.Auth.UserPass.Domain` | JWT authentication with BCrypt password hashing |
| `LSCore.Auth.UserPass.DependencyInjection` | JWT authentication middleware and DI wiring |
| `LSCore.Common.Contracts` | Shared data types (e.g., `LSCoreIdRequest`) |
| `LSCore.Common.Extensions` | Utility extension methods |
| `LSCore.DependencyInjection` | Convention-based assembly scanner for automatic service registration |
| `LSCore.Exceptions` | Exception types mapping to HTTP status codes (400, 401, 403, 404, 500) |
| `LSCore.Exceptions.DependencyInjection` | Exception-handling middleware for ASP.NET Core |
| `LSCore.Logging` | Console and debug logging configuration |
| `LSCore.Mapper.Contracts` | `ILSCoreMapper<TSource, TDestination>` interface |
| `LSCore.Mapper.Domain` | `ToMapped<>()` extension methods for object mapping |
| `LSCore.Repository.Contracts` | Repository interfaces and `LSCoreEntity` base class |
| `LSCore.Repository` | Generic repository with CRUD, soft-delete, and audit fields |
| `LSCore.SortAndPage.Contracts` | Sorting and pagination request/response models |
| `LSCore.SortAndPage.Domain` | `ToSortedAndPagedResponse<>()` extension on `IQueryable<T>` |
| `LSCore.Validation.Contracts` | `ILSCoreValidator<T>` interface and validation message utilities |
| `LSCore.Validation.Domain` | FluentValidation-based validators with automatic `Validate<T>()` extension |

---

## Links

- **GitHub Repository**: [https://github.com/LimitlessSoft/LSCore](https://github.com/LimitlessSoft/LSCore)
- **Discord**: [https://discord.gg/PTWERkBV](https://discord.gg/PTWERkBV)
- **NuGet Packages**: [https://www.nuget.org/profiles/LimitlessSoft](https://www.nuget.org/profiles/LimitlessSoft)
- **Issues**: [https://github.com/LimitlessSoft/LSCore/issues](https://github.com/LimitlessSoft/LSCore/issues)
- **Feature Requests**: [https://github.com/LimitlessSoft/LSCore/discussions/categories/request-feature](https://github.com/LimitlessSoft/LSCore/discussions/categories/request-feature)
- **Postman Collection**: [LSCore.postman_collection.json](https://github.com/user-attachments/files/19439535/LSCore.postman_collection.json)
