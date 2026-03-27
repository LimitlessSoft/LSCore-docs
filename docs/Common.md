---
layout: default
title: Common
nav_order: 11
---

# LSCore Common

The LSCore Common module provides shared contracts and utility extensions used across the LSCore ecosystem. It is split into two independent NuGet packages so consumers can depend on only what they need.

## NuGet Packages

| Package | Description |
|---|---|
| `LSCore.Common.Contracts` | Shared request/response contracts that have not yet been promoted to their own domain-specific packages. |
| `LSCore.Common.Extensions` | General-purpose extension methods and supporting types that have not yet been promoted to their own domain-specific packages. |

Both packages target **.NET 9.0** (version 9.1.4.1 at time of writing). Neither package has external NuGet dependencies.

---

## LSCore.Common.Contracts

### LSCoreIdRequest

A simple request DTO used throughout the framework wherever an operation needs a single `long` identifier.

```csharp
namespace LSCore.Common.Contracts;

public class LSCoreIdRequest
{
    public long Id { get; set; }

    public LSCoreIdRequest() { }

    public LSCoreIdRequest(long id) => Id = id;
}
```

**Usage example -- passing an ID to a service method:**

```csharp
using LSCore.Common.Contracts;

public class UserController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetUser(long id)
    {
        var request = new LSCoreIdRequest(id);
        var user = _userService.GetById(request);
        return Ok(user);
    }
}
```

The parameterless constructor supports model binding from JSON request bodies, while the parameterized constructor is convenient for in-code construction.

---

## LSCore.Common.Extensions

### DefaultValueType

An enum that controls what `GetDescriptionOrDefault` returns when a `[Description]` attribute is not found on an enum member.

```csharp
namespace LSCore.Common.Extensions;

public enum DefaultValueType
{
    Null,   // Return null
    Empty,  // Return string.Empty
    Self    // Return the enum member's name (value.ToString())
}
```

### EnumExtensions

Extension methods for extracting `[Description]` attribute values from enum members.

#### GetDescription

Returns the `System.ComponentModel.DescriptionAttribute` value for the given enum member. Throws `NullReferenceException` if no `[Description]` attribute is present.

```csharp
public static string GetDescription(this Enum value)
```

#### GetDescriptionOrDefault

Returns the `[Description]` value if present. When the attribute is missing, the behavior is determined by the `defaultValueType` parameter:

```csharp
public static string? GetDescriptionOrDefault(
    this Enum value,
    DefaultValueType defaultValueType
)
```

| `DefaultValueType` | Behavior when `[Description]` is absent |
|---|---|
| `Null` | Returns `null` |
| `Empty` | Returns `string.Empty` |
| `Self` | Returns `value.ToString()` (the enum member name) |

**Usage example:**

```csharp
using System.ComponentModel;
using LSCore.Common.Extensions;

public enum OrderStatus
{
    [Description("Pending approval")]
    Pending,

    [Description("Order shipped")]
    Shipped,

    Cancelled   // No Description attribute
}

// Returns "Pending approval"
var desc = OrderStatus.Pending.GetDescription();

// Returns "Order shipped"
var desc2 = OrderStatus.Shipped.GetDescriptionOrDefault(DefaultValueType.Null);

// Returns null (no attribute, fallback is Null)
var desc3 = OrderStatus.Cancelled.GetDescriptionOrDefault(DefaultValueType.Null);

// Returns "" (no attribute, fallback is Empty)
var desc4 = OrderStatus.Cancelled.GetDescriptionOrDefault(DefaultValueType.Empty);

// Returns "Cancelled" (no attribute, fallback is Self)
var desc5 = OrderStatus.Cancelled.GetDescriptionOrDefault(DefaultValueType.Self);

// Throws NullReferenceException (no attribute, GetDescription is strict)
var desc6 = OrderStatus.Cancelled.GetDescription();
```
