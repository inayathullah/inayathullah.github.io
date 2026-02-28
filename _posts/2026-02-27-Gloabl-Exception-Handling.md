---
layout: post
title: ASP.NET Global Exception Handling
categories: ASP.NET
tags: [ASP.NET, Exception, C#]
---


### How Exceptions Are Normally Handled
Let's look at the code below.

```csharp
[ApiController]
public class EmployeeController : ControllerBase
{
    private readonly IEmployeeService _employeeService;
    private readonly ILogger<EmployeeController> _logger;
    public EmployeeController(
        IEmployeeService employeeService,
        ILogger<EmployeeController> logger)
    {
        _employeeService = employeeService;
        _logger = logger;
    }
    public async Task<IActionResult> Post(
            [FromBody] CreateEmployeeRequest request,
            CancellationToken token = default)
    {
        try
        {
            var result = await _employeeService
                    .CreateAsync(request, token);
            if (result.IsSuccess()){
                return Created();
            }
            return BadRequest();
        }
        catch(Exception e)
        {
            _logger.LogError(e, "Failed to process request");
            return StatusCode(StatusCodes.Status500InternalServerError);
        }
    }
}
```

The `catch` block is where exception handling is currently implemented. When an exception occurs, we log it and return ``500 Internal Server Error``.

### What is the problem in this approach?
- Every method in the controller has to handle exceptions.
- This is bad practice and violates the DRY principle.
- If we need to change logic in the `catch` block globally, we have to repeat that change in every method, which is not ideal.
- Debugging becomes painful because we need to trace log messages across the codebase to find which method emitted them.


### How to Implement Centralized Global Exception Handling

### Middlewares
Middleware is a piece of software that is assembled into an ASP.NET app pipeline to handle requests and responses.

Each middleware:

- Chooses whether to pass the request to the next middleware in the pipeline.
- Can perform work before and after the next middleware in the pipeline.
- The order of middleware is also important ([Middleware Order](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0#middleware-order)).

Why middleware is useful:

- Useful for short-circuiting a response when a request fails to meet criteria.
- A centralized place to inspect input and act accordingly.
- A centralized place to handle cross-cutting concerns like logging, exception handling, and authorization.
- A centralized place to override output/response.


#### Let's Build a GlobalExceptionHandler Middleware
<img class="zoomable" src="/assets/globalexceptionhandler/middleware.png" alt="Middleware">

<div class="code-header">GlobalExceptionHandler.cs</div>
```csharp
internal class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(
        ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger
            .LogError(exception,
                 "Exception occurred: {Message}", exception.Message);

        var problemDetails = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "Server error"
        };

        httpContext.Response.StatusCode = problemDetails.Status.Value;

        await httpContext
            .Response
            .WriteAsJsonAsync(problemDetails, cancellationToken);

        return true;
    }
}
```
Let's hook the middleware into the ASP.NET pipeline.

<div class="code-header">Program.cs</div>
```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
....
....
...
app.UseExceptionHandler();
....
....
```

Let's remove the `try-catch` block from the controller method. Exception handling is no longer the controller's concern; it becomes the application's concern.

```csharp
[ApiController]
public class EmployeeController : ControllerBase
{
    private readonly IEmployeeService _employeeService;
    public EmployeeController(
        IEmployeeService employeeService)
    {
        _employeeService = employeeService;
    }
    public async Task<IActionResult> Post(
            [FromBody] CreateEmployeeRequest request,
            CancellationToken token = default)
    {
        var result = await _employeeService
                    .CreateAsync(request, token);
        if (result.IsSuccess()){
            return Created();
        }
        return BadRequest();
    }
}
```

The controller is now clean and focused. Exceptions are handled by middleware. If an exception occurs, the middleware logs it and returns ``500 Internal Server Error`` with problem details.
