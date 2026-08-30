---
layout: default
title: Validation
nav_order: 6
---

# LSCore Validation

The LSCore Validation module provides a structured, convention-based validation system built on top of [FluentValidation](https://docs.fluentvalidation.net/). It integrates with the LSCore dependency injection system for automatic registration and offers a simple `Validate()` extension method that can be called on any object.

When validation fails, the module throws an `LSCoreBadRequestException` containing a JSON-serialized list of validation errors, which the LSCore exception handling middleware automatically converts into an HTTP 400 response.

## NuGet Packages

The validation system is split into two packages:

| Package | Description | Version |
|---------|-------------|---------|
| `LSCore.Validation.Contracts` | Interfaces and attributes -- no external dependencies | 10.0.0 |
| `LSCore.Validation.Domain` | Base class, extension methods, and FluentValidation integration | 10.0.0 |

Both target **.NET 10.0**.

### Dependencies

`LSCore.Validation.Domain` depends on:

- `LSCore.Validation.Contracts`
- `LSCore.DependencyInjection`
- `LSCore.Exceptions`
- `FluentValidation` 12.1.1
- `Newtonsoft.Json` 13.0.4

`LSCore.Validation.Contracts` has no external package dependencies.

> **FluentValidation 12 is required from LSCore 10.0.0.** `LSCoreValidatorBase<T>` publicly derives
> from `AbstractValidator<T>`, so the FluentValidation major version is part of this package's API
> surface -- referencing `LSCore.Validation.Domain` 10.0.0 pulls your project onto FluentValidation
> 12. The serialized validation error payload is unchanged from LSCore 9.x. Projects that must stay
> on FluentValidation 11 should stay on the LSCore `9.1.x` line.

---

## Core Concepts

### ILSCoreValidator Interface

The marker interface that identifies a class as an LSCore validator. It carries a generic type constraint limiting validation to reference types.

```csharp
namespace LSCore.Validation.Contracts;

public interface ILSCoreValidator<TRequest>
    where TRequest : class;
```

This interface is used by the DI scanning system to discover validators at startup. You do not interact with it directly -- it is implemented by `LSCoreValidatorBase<T>`.

### LSCoreValidatorBase

The abstract base class for all validators. It inherits from FluentValidation's `AbstractValidator<TRequest>` and implements `ILSCoreValidator<TRequest>`.

```csharp
namespace LSCore.Validation.Domain;

public class LSCoreValidatorBase<TRequest> : AbstractValidator<TRequest>, ILSCoreValidator<TRequest>
    where TRequest : class;
```

All custom validators must inherit from this class. This gives you access to the full FluentValidation rule API while ensuring the validator is automatically discovered and registered by the LSCore DI system.

### LSCoreValidationMessageAttribute

A custom attribute for attaching human-readable validation messages to enum values. It extends `System.ComponentModel.DescriptionAttribute`.

```csharp
namespace LSCore.Validation.Contracts;

public class LSCoreValidationMessageAttribute(string description)
    : DescriptionAttribute(description);
```

Use it on enum members to define reusable validation messages:

```csharp
public enum UserValidationMessages
{
    [LSCoreValidationMessage("Username is required.")]
    UsernameRequired,

    [LSCoreValidationMessage("Username must be between {0} and {1} characters.")]
    UsernameLengthInvalid,

    [LSCoreValidationMessage("Email address is not valid.")]
    EmailInvalid
}
```

### Validation Message Extension Methods

Two extension methods on `System.Enum` retrieve the message text from the attribute:

```csharp
// Returns the attribute description, or the enum name if no attribute is present.
string message = UserValidationMessages.UsernameRequired.GetValidationMessage();
// Result: "Username is required."

// Supports string.Format-style arguments for parameterized messages.
string message = UserValidationMessages.UsernameLengthInvalid.GetValidationMessage(3, 50);
// Result: "Username must be between 3 and 50 characters."
```

---

## How to Create a Custom Validator

### Step 1: Define Your Request Model

```csharp
public class CreateUserRequest
{
    public string Username { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
}
```

### Step 2: Define Validation Messages (Optional)

```csharp
using LSCore.Validation.Contracts;

public enum CreateUserValidationMessages
{
    [LSCoreValidationMessage("Username is required.")]
    UsernameRequired,

    [LSCoreValidationMessage("Username must be between {0} and {1} characters.")]
    UsernameLengthInvalid,

    [LSCoreValidationMessage("A valid email address is required.")]
    EmailRequired,

    [LSCoreValidationMessage("Age must be at least {0}.")]
    AgeTooLow
}
```

### Step 3: Create the Validator

Inherit from `LSCoreValidatorBase<T>` and define rules in the constructor using FluentValidation syntax:

```csharp
using FluentValidation;
using LSCore.Validation.Contracts;
using LSCore.Validation.Domain;

public class CreateUserRequestValidator : LSCoreValidatorBase<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty()
            .WithMessage(CreateUserValidationMessages.UsernameRequired.GetValidationMessage())
            .Length(3, 50)
            .WithMessage(CreateUserValidationMessages.UsernameLengthInvalid.GetValidationMessage(3, 50));

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .WithMessage(CreateUserValidationMessages.EmailRequired.GetValidationMessage());

        RuleFor(x => x.Age)
            .GreaterThanOrEqualTo(18)
            .WithMessage(CreateUserValidationMessages.AgeTooLow.GetValidationMessage(18));
    }
}
```

### Step 4: Validate Objects

Call the `Validate()` extension method on any object that has a registered validator:

```csharp
using LSCore.Validation.Domain;

var request = new CreateUserRequest
{
    Username = "",
    Email = "not-an-email",
    Age = 12
};

request.Validate(); // Throws LSCoreBadRequestException if invalid
```

If validation fails, an `LSCoreBadRequestException` is thrown with a JSON-serialized array of `FluentValidation.Results.ValidationFailure` objects as the exception message.

---

## DI Registration

Validators are automatically registered when you use the LSCore dependency injection system. No manual registration is required.

### Setup in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register all LSCore services, including validators.
// Pass the root namespace of your project so the scanner knows which assemblies to inspect.
builder.AddLSCoreDependencyInjection("MyProject");

var app = builder.Build();

// Required: makes the service provider available for the Validate() extension method.
app.UseLSCoreDependencyInjection();

// Optional but recommended: converts LSCoreBadRequestException into HTTP 400 responses.
app.UseLSCoreExceptionsHandler();

app.Run();
```

### How Auto-Registration Works

During startup, `AddLSCoreDependencyInjection` scans all assemblies whose names start with the provided project root name. For each non-abstract, non-generic class it finds, it checks whether the class's inheritance chain contains `LSCoreValidatorBase<T>`. If it does, the class is registered as a **singleton** in the DI container, keyed to its specific `LSCoreValidatorBase<TRequest>` base type.

This means a validator like `CreateUserRequestValidator : LSCoreValidatorBase<CreateUserRequest>` is registered as `LSCoreValidatorBase<CreateUserRequest>` in the service collection.

### Disabling Auto-Registration

If you need to disable automatic validator scanning (for example, to register validators manually), use the scanning options:

```csharp
builder.AddLSCoreDependencyInjection("MyProject", options =>
{
    options.Scan.DisableLSCoreValidators();
});
```

---

## Integration with Other LSCore Modules

### LSCore.Exceptions

When validation fails, the `Validate()` extension method throws `LSCoreBadRequestException`. This exception is part of the `LSCore.Exceptions` package and can be automatically handled by the exception middleware.

The exception middleware (`LSCoreExceptionsHandleMiddleware`) catches `LSCoreBadRequestException` and returns:
- HTTP status code **400** (Bad Request)
- The exception message (the JSON validation errors) written to the response body

### LSCore.DependencyInjection

The validation module relies on `LSCore.DependencyInjection` for two things:
1. **Automatic registration** of validators via assembly scanning at startup.
2. **Runtime resolution** of validators via `Container.ServiceProvider` when `Validate()` is called.

The call to `app.UseLSCoreDependencyInjection()` is **required** for the `Validate()` extension method to work. Without it, `Container.ServiceProvider` is null and `Validate()` throws an exception.

---

## Complete Working Example

```csharp
// --- CreateUserRequest.cs ---
public class CreateUserRequest
{
    public string Username { get; set; }
    public string Email { get; set; }
}

// --- CreateUserValidationMessages.cs ---
using LSCore.Validation.Contracts;

public enum CreateUserValidationMessages
{
    [LSCoreValidationMessage("Username is required.")]
    UsernameRequired,

    [LSCoreValidationMessage("Email must be a valid email address.")]
    EmailInvalid
}

// --- CreateUserRequestValidator.cs ---
using FluentValidation;
using LSCore.Validation.Contracts;
using LSCore.Validation.Domain;

public class CreateUserRequestValidator : LSCoreValidatorBase<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty()
            .WithMessage(CreateUserValidationMessages.UsernameRequired.GetValidationMessage());

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .WithMessage(CreateUserValidationMessages.EmailInvalid.GetValidationMessage());
    }
}

// --- In your service or handler ---
using LSCore.Validation.Domain;

public class UserService
{
    public void CreateUser(CreateUserRequest request)
    {
        // Throws LSCoreBadRequestException if validation fails.
        request.Validate();

        // Proceed with user creation...
    }
}

// --- Program.cs ---
var builder = WebApplication.CreateBuilder(args);
builder.AddLSCoreDependencyInjection("MyProject");

var app = builder.Build();
app.UseLSCoreDependencyInjection();
app.UseLSCoreExceptionsHandler();

app.Run();
```

## Error Response Format

When validation fails, the HTTP response body contains a JSON array of validation failure objects:

```json
[
  {
    "PropertyName": "Username",
    "ErrorMessage": "Username is required.",
    "AttemptedValue": "",
    "Severity": 0,
    "ErrorCode": "NotEmptyValidator"
  },
  {
    "PropertyName": "Email",
    "ErrorMessage": "Email must be a valid email address.",
    "AttemptedValue": "not-an-email",
    "Severity": 0,
    "ErrorCode": "EmailValidator"
  }
]
```

This structure comes from FluentValidation's `ValidationFailure` class serialized via `Newtonsoft.Json`.
