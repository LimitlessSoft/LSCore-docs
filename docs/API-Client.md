---
layout: default
title: API Client
nav_order: 10
---

# LSCore API Client

The LSCore API Client module provides a base class and supporting infrastructure for building typed REST API clients that communicate with microservices. It handles base URL configuration, API key authentication via the `X-LS-Key` header, HTTP status code mapping to LSCore exceptions, and query string generation from objects.

## NuGet Packages

| Package | Description |
|---------|-------------|
| `LSCore.ApiClient.Rest` | Core API client base class, configuration interface, and HTTP extensions. |
| `LSCore.ApiClient.Rest.DependencyInjection` | `WebApplicationBuilder` extension for registering API clients in the DI container. |

Both packages target **.NET 9.0**.

### Dependencies

`LSCore.ApiClient.Rest` depends on:

- `LSCore.Auth.Key.Contracts` -- provides the `LSCoreAuthKeyHeaders.KeyCustomHeader` constant (`"X-LS-Key"`).
- `LSCore.Exceptions` -- provides the exception types thrown by `HandleStatusCode`.

`LSCore.ApiClient.Rest.DependencyInjection` depends on:

- `LSCore.ApiClient.Rest`
- `Microsoft.AspNetCore.App` (framework reference)

---

## Configuration

### ILSCoreApiClientRestConfiguration

The configuration contract that every API client requires:

```csharp
public interface ILSCoreApiClientRestConfiguration
{
    string BaseUrl { get; set; }
    string? LSCoreApiKey { get; set; }
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `BaseUrl` | `string` | Yes | The base URL of the target microservice (e.g. `"https://api.example.com"`). Set as `HttpClient.BaseAddress`. |
| `LSCoreApiKey` | `string?` | No | When provided, the client adds an `X-LS-Key` header to every request for API key authentication. |

### LSCoreApiClientRestConfiguration\<TClient\>

A generic concrete implementation of `ILSCoreApiClientRestConfiguration` that is parameterized by your client type. This allows multiple API clients in the same application, each with their own configuration instance resolved from DI.

```csharp
public class LSCoreApiClientRestConfiguration<TClient> : ILSCoreApiClientRestConfiguration
    where TClient : LSCoreApiClient
{
    public string BaseUrl { get; set; }
    public string? LSCoreApiKey { get; set; }
}
```

---

## LSCoreApiClient

`LSCoreApiClient` is an abstract base class that your concrete API client must inherit. It creates an `HttpClient`, sets its `BaseAddress` from configuration, optionally attaches the API key header, and provides a `HandleStatusCode` method for uniform error handling.

```csharp
public abstract class LSCoreApiClient
{
    protected readonly HttpClient _httpClient = new();

    public LSCoreApiClient(ILSCoreApiClientRestConfiguration configuration)
    {
        _httpClient.BaseAddress = new Uri(configuration.BaseUrl);
        if (!string.IsNullOrWhiteSpace(configuration.LSCoreApiKey))
            _httpClient.DefaultRequestHeaders.Add(
                LSCoreAuthKeyHeaders.KeyCustomHeader,
                configuration.LSCoreApiKey
            );
    }

    protected void HandleStatusCode(HttpResponseMessage? response) { /* ... */ }
}
```

### Creating a Concrete API Client

Inherit from `LSCoreApiClient` and implement your service-specific methods. Use `_httpClient` for requests and call `HandleStatusCode` to translate HTTP errors into LSCore exceptions.

```csharp
using System.Net.Http.Json;
using LSCore.ApiClient.Rest;

public class WeatherApiClient : LSCoreApiClient
{
    public WeatherApiClient(LSCoreApiClientRestConfiguration<WeatherApiClient> config)
        : base(config) { }

    public async Task<WeatherForecast?> GetForecastAsync(string city)
    {
        var response = await _httpClient.GetAsync($"/api/forecast/{city}");
        HandleStatusCode(response);
        return await response.Content.ReadFromJsonAsync<WeatherForecast>();
    }

    public async Task CreateForecastAsync(WeatherForecast forecast)
    {
        var response = await _httpClient.PostAsJsonAsync("/api/forecast", forecast);
        HandleStatusCode(response);
    }

    public async Task UpdateForecastAsync(int id, WeatherForecast forecast)
    {
        var response = await _httpClient.PutAsJsonAsync($"/api/forecast/{id}", forecast);
        HandleStatusCode(response);
    }

    public async Task DeleteForecastAsync(int id)
    {
        var response = await _httpClient.DeleteAsync($"/api/forecast/{id}");
        HandleStatusCode(response);
    }
}
```

### HandleStatusCode

Call `HandleStatusCode` after every HTTP call to map the response status to the appropriate LSCore exception:

| HTTP Status | Behavior |
|-------------|----------|
| `200 OK` | Returns normally (no exception). |
| `400 Bad Request` | Throws `LSCoreBadRequestException` with message `"Microservice API returned bad request."` |
| `401 Unauthorized` | Throws `LSCoreUnauthenticatedException`. |
| `403 Forbidden` | Throws `LSCoreForbiddenException`. |
| `404 Not Found` | Throws `LSCoreNotFoundException`. |
| `null` response | Throws `LSCoreBadRequestException` with message `"Response is null."` |
| Any other status | Throws `LSCoreBadRequestException` with message `"Microservice API returned unhandled exception."` |

---

## HttpClient Extensions

The `HttpClientExtensions` class provides a convenience method for sending GET requests with query parameters built from an object's properties.

### GetAsJsonAsync

```csharp
public static Task<HttpResponseMessage> GetAsJsonAsync(
    this HttpClient client,
    string requestUri,
    object obj
)
```

Converts the public properties of `obj` into URL-encoded query parameters and appends them to `requestUri`, then issues a GET request.

- Null-valued properties are excluded.
- Enumerable properties (except `string`) are expanded into repeated query parameters (e.g. `Ids=1&Ids=2&Ids=3`).
- All values are URL-encoded via `HttpUtility.UrlEncode`.

**Example:**

```csharp
var filter = new
{
    City = "New York",
    MinTemp = 20,
    Tags = new[] { "sunny", "warm" }
};

// Produces: GET /api/forecast?City=New+York&MinTemp=20&Tags=sunny&Tags=warm
var response = await _httpClient.GetAsJsonAsync("/api/forecast", filter);
HandleStatusCode(response);
```

---

## Dependency Injection Registration

The `LSCore.ApiClient.Rest.DependencyInjection` package provides a `WebApplicationBuilder` extension method for registering API clients.

### AddLSCoreApiClientRest\<TClient\>

```csharp
public static void AddLSCoreApiClientRest<TClient>(
    this WebApplicationBuilder builder,
    LSCoreApiClientRestConfiguration<TClient> configuration
) where TClient : LSCoreApiClient
```

This method:

1. Registers the `LSCoreApiClientRestConfiguration<TClient>` as a **singleton** in the DI container.
2. Registers `TClient` as a **transient** service.

If `configuration` is `null`, it throws `ArgumentNullException`.

### Full Registration Example

```csharp
using LSCore.ApiClient.Rest;
using LSCore.ApiClient.Rest.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Register the WeatherApiClient with its configuration
builder.AddLSCoreApiClientRest(new LSCoreApiClientRestConfiguration<WeatherApiClient>
{
    BaseUrl = "https://weather-service.internal",
    LSCoreApiKey = "my-secret-api-key"
});

var app = builder.Build();
```

After registration, inject the client into controllers or services:

```csharp
public class WeatherController : ControllerBase
{
    private readonly WeatherApiClient _weatherClient;

    public WeatherController(WeatherApiClient weatherClient)
    {
        _weatherClient = weatherClient;
    }

    [HttpGet("{city}")]
    public async Task<IActionResult> GetForecast(string city)
    {
        var forecast = await _weatherClient.GetForecastAsync(city);
        return Ok(forecast);
    }
}
```

---

## Configuring Authentication

The API client supports authentication via the LSCore Auth Key system. When `LSCoreApiKey` is set in the configuration, the `X-LS-Key` header is automatically added to the `HttpClient`'s default request headers. This means every request sent through the client includes the key without any additional code.

```csharp
// With authentication
builder.AddLSCoreApiClientRest(new LSCoreApiClientRestConfiguration<MyServiceClient>
{
    BaseUrl = "https://my-service.internal",
    LSCoreApiKey = "your-api-key-here"
});

// Without authentication (LSCoreApiKey omitted or null)
builder.AddLSCoreApiClientRest(new LSCoreApiClientRestConfiguration<PublicServiceClient>
{
    BaseUrl = "https://public-service.example.com"
});
```

---

## Registering Multiple Clients

Because `LSCoreApiClientRestConfiguration<TClient>` is generic over the client type, you can register multiple API clients in the same application, each with independent configuration:

```csharp
builder.AddLSCoreApiClientRest(new LSCoreApiClientRestConfiguration<WeatherApiClient>
{
    BaseUrl = "https://weather-service.internal",
    LSCoreApiKey = "weather-key"
});

builder.AddLSCoreApiClientRest(new LSCoreApiClientRestConfiguration<BillingApiClient>
{
    BaseUrl = "https://billing-service.internal",
    LSCoreApiKey = "billing-key"
});
```

Each client resolves its own configuration from DI because `LSCoreApiClientRestConfiguration<WeatherApiClient>` and `LSCoreApiClientRestConfiguration<BillingApiClient>` are distinct types.
