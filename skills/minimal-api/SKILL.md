---
name: minimal-api
description: Build a new HTTP API with ASP.NET Core Minimal APIs. Use when creating a new API, or adding endpoints to an API built with this skill - endpoints (MapGet, MapPost, MapGroup), route handlers, parameter binding, pagination, TypedResults and Results<...> responses, request validation, problem details and error handling, OpenAPI metadata, endpoint authorization, CORS, health check endpoints (liveness, readiness), forwarded headers behind a proxy, or the API's Program.cs. Not for existing APIs built with controllers or other conventions. Doesn't cover authentication setup, rate limiting, output caching, API versioning, or how the application layers are organized (clean-architecture skill).
---

# Minimal APIs

Aim for endpoints that are **small, typed, and self-describing**: each feature maps its own group of endpoints, endpoint methods delegate the work to a service, responses are typed so OpenAPI describes them without extra metadata, invalid input is rejected before the endpoint runs, and errors are returned as problem details.

## 1. Scope

This skill is for **new HTTP APIs**. Don't apply it to an existing API built with controllers or other conventions, and don't use it to migrate one.

Read the target framework first: some features below depend on it (marked "from .NET 10").

The examples call an `IOrderService` and use request and response records. When the solution follows the clean-architecture skill, the service is a use-case handler from `Application` (`CreateOrderHandler`) and the records come from `Application`; everything else in this skill is unchanged.

## 2. Program.cs

`Program.cs` registers services, builds the middleware pipeline, and maps one line per feature. It contains no endpoint methods. `DatabaseHealthCheck`, `IOrderService`, and `OrderService` are placeholders for the app's own types.

```csharp
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.AspNetCore.HttpOverrides;
using MyApp.Api.Features.Orders;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddProblemDetails();
builder.Services.AddValidation(); // from .NET 10
builder.Services.AddOpenApi();
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database", tags: ["ready"]); // or registered by the layer that owns the dependency
// Behind a reverse proxy only (section 10). Remove when clients reach the app directly.
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    // The proxy's network. Fully qualified: Microsoft.AspNetCore.HttpOverrides also has an IPNetwork type.
    options.KnownIPNetworks.Add(System.Net.IPNetwork.Parse("10.0.0.0/8")); // from .NET 10; KnownNetworks before
});
builder.Services.AddAuthentication(); // required: add the scheme (JWT bearer, Entra ID), or protected endpoints return 500 (section 9)
builder.Services.AddAuthorization();

builder.Services.AddScoped<IOrderService, OrderService>(); // or the layers' registration methods

var app = builder.Build();

app.UseForwardedHeaders();
app.UseExceptionHandler();
app.UseStatusCodePages();
app.UseAuthentication();
app.UseAuthorization();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = _ => false })
    .ShortCircuit();
app.MapHealthChecks("/health/ready", new HealthCheckOptions { Predicate = check => check.Tags.Contains("ready") })
    .ShortCircuit();

app.MapOrderEndpoints();

app.Run();
```

Middleware order:
1. `UseForwardedHeaders()`: first, so every later middleware sees the original scheme and client IP.
2. `UseExceptionHandler()` and `UseStatusCodePages()`: before anything that can fail.
3. `UseCors()`, when the API has a CORS policy (section 9).
4. `UseAuthentication()` then `UseAuthorization()`.
5. Endpoint mapping.

## 3. Organizing endpoints

One static class per feature, with an extension method that maps the feature's group:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;

namespace MyApp.Api.Features.Orders;

public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapGet("/", ListOrders)
            .WithName("ListOrders")
            .WithSummary("List orders");

        group.MapGet("/{id:guid}", GetOrder)
            .WithName("GetOrder")
            .WithSummary("Get an order by ID");

        group.MapPost("/", CreateOrder)
            .WithName("CreateOrder")
            .WithSummary("Create an order")
            .ProducesValidationProblem();

        return app;
    }

    public static async Task<Ok<PagedResponse<OrderResponse>>> ListOrders(
        [AsParameters] ListOrdersQuery query, IOrderService orders, CancellationToken cancellationToken)
        => TypedResults.Ok(await orders.ListAsync(query, cancellationToken));

    public static async Task<Results<Ok<OrderResponse>, NotFound>> GetOrder(
        Guid id, IOrderService orders, CancellationToken cancellationToken)
        => await orders.FindAsync(id, cancellationToken) is { } order
            ? TypedResults.Ok(order)
            : TypedResults.NotFound();

    public static async Task<CreatedAtRoute<OrderResponse>> CreateOrder(
        CreateOrderRequest request, IOrderService orders, CancellationToken cancellationToken)
    {
        var order = await orders.CreateAsync(request, cancellationToken);
        return TypedResults.CreatedAtRoute(order, "GetOrder", new { id = order.Id });
    }
}
```

- Place the file with the feature (`Features/Orders/OrderEndpoints.cs`).
- Call each `Map<Feature>Endpoints()` explicitly from `Program.cs`. Don't discover endpoint classes by reflection: it hides what is mapped and isn't compatible with trimming or Native AOT.
- Put the shared prefix, tags, and authorization on the group, not on each endpoint.
- Write endpoint methods as named static methods, not long lambdas: they are readable, and tests can call them and check the returned type.
- Keep endpoint methods thin: bind the input, call one service method, map its outcome to an HTTP result. Business rules and data access live behind the service.
- Name every endpoint that another endpoint links to (`WithName`), and give each endpoint a summary. Endpoint names must be unique in the whole app: start them with the feature (`GetOrder`, `GetCustomer`), not a generic verb (`GetById`).
- Routes: plural nouns for collections (`/orders`), route constraints on IDs (`{id:guid}`, `{id:int}`), no verbs in paths.

## 4. Binding

| Source | How it binds |
| --- | --- |
| Route | Parameter with the route value's name: `Guid id` for `{id:guid}` |
| Query string | Simple parameters not in the route: required if non-nullable, optional if nullable or with a default |
| Several query values | `[AsParameters]` on a record: `ListOrdersQuery` |
| Header | `[FromHeader(Name = "X-Correlation-Id")] string? correlationId` |
| Form or file upload | `[FromForm]` record, `IFormFile`, `IFormFileCollection`: antiforgery applies (below) |
| Body (JSON) | One complex parameter (a request record) on POST, PUT, PATCH |
| Services | Registered services bind from dependency injection without an attribute |

- Endpoints that bind form data or files require antiforgery validation; without antiforgery services, calling them fails.
  - API authenticated with bearer tokens or API keys: add `.DisableAntiforgery()` to these endpoints. A browser can't send the token on its own, so cross-site request forgery doesn't apply.
  - API authenticated with cookies: keep the protection (`builder.Services.AddAntiforgery()`), and the client sends the antiforgery token with the form.
- Use request and response records. Never bind to or return entities: it exposes internal fields, allows over-posting, and couples the API contract to the database.
- Add a `CancellationToken` parameter to every async endpoint method and pass it down: it is canceled when the client disconnects.
- Paginate every list endpoint: an unbounded list gets slower and heavier as the data grows.

  ```csharp
  public sealed record ListOrdersQuery(int Page = 1, int PageSize = 20);

  public sealed record PagedResponse<T>(IReadOnlyList<T> Items, int Page, int PageSize, int TotalCount);
  ```

  Enforce the limits behind the endpoint, whatever the client sends: `Page` at least 1, `PageSize` clamped (`Math.Clamp(query.PageSize, 1, 100)`). Sort by a stable key before paging, or pages can overlap.
- Use `System.Text.Json`, the default. Configure it with `builder.Services.ConfigureHttpJsonOptions(...)` (`AddJsonOptions` configures MVC only). Don't add Newtonsoft.Json.

## 5. Responses

- Return `TypedResults`, not `Results`: the concrete result types describe the responses in OpenAPI and can be checked in tests.
- When an endpoint method returns more than one result type, declare them with `Results<T1, T2, ...>`: the compiler rejects a result that isn't declared.
- Don't add `.Produces<T>()` for responses already in the return type. Add `.ProducesProblem(...)` or `.ProducesValidationProblem()` only for responses produced outside the endpoint method (validation, exception handler).
- `201 Created`: return `TypedResults.CreatedAtRoute(value, "<endpoint name>", routeValues)`, or `TypedResults.Created(...)` with a relative location.
- Expected outcomes (not found, conflict) are results, not exceptions: the service returns a value (`null`, a result type) and the endpoint method maps it to `TypedResults.NotFound()`, `TypedResults.Conflict(...)`, or `TypedResults.Problem(...)`.

## 6. Validation

From .NET 10, use the built-in validation:

```csharp
builder.Services.AddValidation();
```

```csharp
using System.ComponentModel.DataAnnotations;

public sealed record CreateOrderRequest(
    [Required, StringLength(50)] string CustomerId,
    [Range(1, 1000)] int Quantity);
```

- Invalid query, header, or body values return `400` with validation problem details before the endpoint method runs.
- Use DataAnnotations attributes for single values and `IValidatableObject` for rules across several properties.
- `AddValidation()` discovers the endpoint parameter types and the types they contain, including records declared in other projects (a class library, or `Application` with the clean-architecture skill). Don't add `[ValidatableType]` or another `AddValidation()` for them.
- Test one invalid request per endpoint: missing validation produces no error or log, and the test is what proves it runs.
- Use `.DisableValidation()` only on an endpoint that validates its input another way.
- Before .NET 10, validate in an endpoint filter on the group and return `TypedResults.ValidationProblem(errors)`.
- Validation checks the shape of the input. Rules that need data (unique name, existing customer) belong in the service.

## 7. Errors

```csharp
builder.Services.AddProblemDetails();

app.UseExceptionHandler();
app.UseStatusCodePages();
```

- Unhandled exceptions return `500` problem details without a stack trace; error status codes without a body (`404`, `405`) also get problem details.
- Don't catch exceptions in endpoint methods to return `500` yourself: the exception handler does it and logs the exception.
- Don't return exception messages or stack traces to clients outside Development.
- To customize problem details for all responses (for example, add a trace ID), configure `AddProblemDetails(options => options.CustomizeProblemDetails = ...)` instead of writing a custom middleware.

## 8. OpenAPI

- Use the built-in OpenAPI support: `builder.Services.AddOpenApi()` and `app.MapOpenApi()`. Don't add Swashbuckle.
- Map the document in Development only (`if (app.Environment.IsDevelopment())`): production doesn't expose its API description.
- Describe endpoints with metadata (`WithSummary`, `WithDescription`, `WithTags`, `WithName`), not code comments only.
- Don't use `WithOpenApi()`: it is obsolete from .NET 10 (`ASPDEPR002`) and doesn't feed the built-in generator. For per-endpoint changes to the document, use `AddOpenApiOperationTransformer(...)`.
- A UI for the document (Swagger UI, Scalar) is a development tool: map it in Development only too.

## 9. Authorization

Authentication setup (JWT bearer, Microsoft Entra ID) is outside this skill. On the endpoint side:

- `RequireAuthorization()` needs an authentication scheme. With `AddAuthentication()` alone, every protected endpoint returns `500` (`No authenticationScheme was specified`): configure the scheme before mapping protected groups.
- Tests of protected endpoints, invalid-request tests included, need an authenticated client: authorization runs before validation. With `WebApplicationFactory`, register a test authentication scheme in `ConfigureTestServices`.
- Require authorization on each group: `.RequireAuthorization()` or `.RequireAuthorization("<policy>")`.
- Mark anonymous endpoints explicitly with `.AllowAnonymous()` inside a protected group, so the exception is visible.
- Define policies in `AddAuthorization(options => ...)` or `AddAuthorizationBuilder()`, not as role strings repeated on endpoints.
- Health endpoints are mapped outside the API groups and stay anonymous; don't expose details in their response.

**CORS** (only when a browser app on another origin calls the API; server-to-server calls don't need it)
- Define one named policy with the exact origins, from configuration: `AddCors(options => options.AddPolicy("<name>", policy => policy.WithOrigins(origins)...))`.
- Apply it with `.RequireCors("<name>")` on the groups that browsers call, and call `app.UseCors()` before authentication.
- Never combine `AllowAnyOrigin()` with credentials, and don't use `AllowAnyOrigin()` on an API that requires authentication.
- Allow only the methods and headers the browser app uses.

## 10. Hosting concerns

**Health checks**
- `/health/live`: no checks (`Predicate = _ => false`). It reports that the process responds; the orchestrator restarts the container when it fails.
- `/health/ready`: only checks tagged `ready` (database, required dependencies). The orchestrator stops sending traffic while it fails, without a restart.
- Register each dependency check where the dependency is configured: with the clean-architecture skill, the database check is added in `AddInfrastructure()`, next to the `DbContext`. `Program.cs` only calls `AddHealthChecks()` and maps the endpoints.
- Don't put dependency checks in liveness: a database outage would restart every instance.
- Add `.ShortCircuit()` to both: the endpoint runs as soon as routing matches it and skips the rest of the pipeline (logging, authentication), which matters for probes sent every few seconds. A short-circuited endpoint can't use `RequireAuthorization()` or `RequireCors()`.
- Don't route `/health/*` through the public ingress: only the orchestrator needs them. If the platform can't restrict the path, serve them on a dedicated port: make Kestrel listen on both ports (`ASPNETCORE_HTTP_PORTS=8080;8081`), expose only `8080` through the ingress, and replace the two `MapHealthChecks` calls with the health check middleware bound to the port:

  ```csharp
  app.UseHealthChecks("/health/live", 8081, new HealthCheckOptions { Predicate = _ => false });
  app.UseHealthChecks("/health/ready", 8081, new HealthCheckOptions { Predicate = check => check.Tags.Contains("ready") });
  ```

  The middleware checks the port the connection actually arrived on and ends the request itself. Don't use `.RequireHost("*:8081")` for this: it matches the `Host` header, which the client controls, so a request through the public port with `Host: x:8081` would reach the endpoint.
- Point the orchestrator probes at these paths (the docker skill doesn't add a `HEALTHCHECK`).

**Forwarded headers** (behind a reverse proxy, ingress, or load balancer that terminates TLS)
- Enable `XForwardedFor | XForwardedProto` and call `UseForwardedHeaders()` first in the pipeline. Without it, the app sees `http` and the proxy's IP: redirects, authentication callbacks, and generated links break.
- Trust only the proxy: add its addresses to `KnownProxies`, or its network to `KnownIPNetworks` (from .NET 10; `KnownNetworks` before).
- `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true` enables the same headers but trusts every sender: use it only when the app can't be reached except through the proxy.
- Never enable forwarded headers on an app exposed directly to clients: they could forge their IP and scheme.
- Behind a TLS-terminating proxy, don't add `UseHttpsRedirection()` without forwarded headers: it redirects in a loop.

**Logging**
- Keep the default console logger: in a container, logs go to stdout and the platform collects them. Don't write log files.
- Use the JSON formatter for structured logs in containers: `Logging__Console__FormatterName=json`.
- Don't log request bodies, tokens, or personal data.

## 11. Anti-patterns

| Avoid | Do instead |
| --- | --- |
| Endpoint methods written inline in `Program.cs` | `Map<Feature>Endpoints()` extension per feature |
| Reflection-based endpoint discovery | Explicit `app.Map<Feature>Endpoints()` calls |
| `Results.Ok(...)` with an `IResult` return type | `TypedResults` with `Results<...>` |
| `.Produces<T>()` repeating the return type | Rely on `TypedResults` metadata |
| Returning or binding entities | Request and response records |
| Business logic and queries in endpoint methods | A service called by the endpoint method |
| List endpoint returning every row | Page and size, size capped behind the endpoint |
| Generic endpoint names (`GetById`) reused across features | Unique names starting with the feature |
| Async endpoint method without `CancellationToken` | Add it and pass it down |
| Manual validation at the top of each endpoint method | `AddValidation()` (from .NET 10) or an endpoint filter |
| `try/catch` returning `500` in endpoint methods | `AddProblemDetails()` + `UseExceptionHandler()` |
| Exceptions for not found or conflict | Values from the service, mapped to `TypedResults` |
| Swashbuckle | `AddOpenApi()` / `MapOpenApi()` |
| `WithOpenApi()` | Remove it; `AddOpenApiOperationTransformer` for tweaks |
| OpenAPI document or UI mapped in production | Map in Development only |
| `RequireAuthorization()` repeated on each endpoint | On the group, `AllowAnonymous()` for exceptions |
| `AddAuthentication()` without a scheme behind `RequireAuthorization()` | Configure the scheme; a test scheme in endpoint tests |
| `AllowAnyOrigin()` on an authenticated API | Named CORS policy with explicit origins |
| Dependency checks in the liveness endpoint | Liveness without checks, dependencies in readiness |
| Health endpoints reachable through the public ingress | Internal only, or a dedicated port; `.ShortCircuit()` |
| `.RequireHost("*:<port>")` to restrict health endpoints to a port | `UseHealthChecks(path, port, options)`, which checks the real port |
| `.DisableAntiforgery()` on a cookie-authenticated API | Keep antiforgery; disable it only with bearer tokens or API keys |
| `UseForwardedHeaders()` after authentication, or trusting every proxy | First in the pipeline, known proxies only |
| Newtonsoft.Json | `System.Text.Json` with `ConfigureHttpJsonOptions` |

## Review checklist

- [ ] Applied to a new API only.
- [ ] `Program.cs` has no endpoint methods; one `Map<Feature>Endpoints()` call per feature.
- [ ] Each feature uses a `MapGroup` with prefix, tags, and authorization; an authentication scheme is configured, and endpoint tests use an authenticated client.
- [ ] Endpoint methods are named static methods that call one service method, with `CancellationToken`, using request and response records.
- [ ] List endpoints paginated, page size capped behind the endpoint.
- [ ] Endpoint names unique across the app; no `WithOpenApi()`.
- [ ] Responses use `TypedResults` and `Results<...>`; no redundant `Produces`.
- [ ] Validation runs before the endpoint method, proven by one invalid-request test per endpoint.
- [ ] `AddProblemDetails()`, `UseExceptionHandler()`, `UseStatusCodePages()`; no stack traces in responses.
- [ ] OpenAPI built in, mapped in Development only; endpoints have names and summaries.
- [ ] CORS only for browser clients: named policy, explicit origins, `UseCors()` before authentication.
- [ ] `/health/live` without checks, `/health/ready` with dependency checks, both `.ShortCircuit()` and not exposed publicly (dedicated port through `UseHealthChecks(path, port, ...)`, never `RequireHost`).
- [ ] Form and file endpoints: antiforgery kept for cookie authentication, `.DisableAntiforgery()` only with bearer tokens or API keys.
- [ ] Forwarded headers first in the pipeline, trusting only known proxies, only behind a proxy.
- [ ] Logs to stdout through the console logger.
