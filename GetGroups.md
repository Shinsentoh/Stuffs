Great — now we’re moving into a **clean, production-grade pattern** 💪
We’ll combine:

* ✅ **FluentValidation** → request validation
* ✅ **FluentResults** → unified success/error handling
* ✅ **Centralized error mapping (no try/catch in service)**
* ✅ Clean separation (Application vs Technical)

---

# 🧩 Architecture Overview

```
[ Controller ]
     ↓
[ GetGroupService ]
     ↓
[ IActiveDirectoryService ]
     ↓
[ Active Directory ]

Validation → FluentValidation
Errors     → FluentResults (configured centrally)
```

---

# 📦 1. NuGet Packages

```bash
dotnet add package FluentResults
dotnet add package FluentValidation
```

---

# 🧠 2. Domain Model

```csharp
public class Group
{
    public string Name { get; set; }
    public string DistinguishedName { get; set; }
}
```

---

# 📥 3. Request / Response (Result-based)

```csharp
using FluentResults;

public enum ItemType
{
    Gpo,
    Computer,
    Group,
    User
}

public class GetGroupsRequest
{
    public string SearchTerm { get; set; }
    public ItemType ItemType { get; set; }
}

// No response class needed — FluentResults handles it
// Result<List<Group>>
```

---

# ✅ 4. FluentValidation

```csharp
using FluentValidation;

public class GetGroupsRequestValidator : AbstractValidator<GetGroupsRequest>
{
    public GetGroupsRequestValidator()
    {
        RuleFor(x => x.SearchTerm)
            .NotEmpty()
            .MinimumLength(2);

        RuleFor(x => x.ItemType)
            .IsInEnum();
    }
}
```

---

# ⚙️ 5. Technical Service (NO try/catch)

👉 Let exceptions bubble — we’ll convert them globally

```csharp
using System.DirectoryServices;

public interface IActiveDirectoryService
{
    SearchResultCollection Search(
        string ldapFilter,
        string[] propertiesToLoad,
        SearchScope scope);
}
```

```csharp
public class ActiveDirectoryService : IActiveDirectoryService
{
    private readonly string _ldapPath;

    public ActiveDirectoryService(string ldapPath)
    {
        _ldapPath = ldapPath;
    }

    public SearchResultCollection Search(
        string ldapFilter,
        string[] propertiesToLoad,
        SearchScope scope)
    {
        using var entry = new DirectoryEntry(_ldapPath);

        var searcher = new DirectorySearcher(entry)
        {
            Filter = ldapFilter,
            SearchScope = scope
        };

        foreach (var prop in propertiesToLoad)
        {
            searcher.PropertiesToLoad.Add(prop);
        }

        return searcher.FindAll(); // exceptions bubble
    }
}
```

---

# 🧠 6. FluentResults Configuration (🔥 KEY PART)

👉 This replaces try/catch in your service

```csharp
using FluentResults;

public static class ResultConfig
{
    public static void Configure()
    {
        Result.Setup(cfg =>
        {
            cfg.DefaultTryCatchHandler = ex => MapException(ex);
        });
    }

    private static IError MapException(Exception ex)
    {
        return ex switch
        {
            System.Runtime.InteropServices.COMException =>
                new Error("Active Directory connection error"),

            UnauthorizedAccessException =>
                new Error("Access denied to Active Directory"),

            _ => new Error("Unexpected error").CausedBy(ex)
        };
    }
}
```

👉 Call this once at startup:

```csharp
ResultConfig.Configure();
```

---

# 🧠 7. Application Service (CLEAN ✅)

👉 No try/catch
👉 Validation + orchestration only

```csharp
using FluentResults;
using FluentValidation;
using System.DirectoryServices;

public class GetGroupService
{
    private readonly IActiveDirectoryService _adService;
    private readonly IValidator<GetGroupsRequest> _validator;

    public GetGroupService(
        IActiveDirectoryService adService,
        IValidator<GetGroupsRequest> validator)
    {
        _adService = adService;
        _validator = validator;
    }

    public Result<List<Group>> Execute(GetGroupsRequest request)
    {
        // ✅ 1. Validation
        var validation = _validator.Validate(request);

        if (!validation.IsValid)
        {
            return Result.Fail(
                validation.Errors.Select(e => new Error(e.ErrorMessage)));
        }

        // ✅ 2. Wrap execution with FluentResults
        return Result.Try(() =>
        {
            var filter = BuildFilter(request);

            var results = _adService.Search(
                filter,
                new[] { "cn", "distinguishedName" },
                SearchScope.Subtree);

            return Map(results);
        });
    }

    private string BuildFilter(GetGroupsRequest request)
    {
        return request.ItemType switch
        {
            ItemType.Group => $"(&(objectClass=group)(cn=*{request.SearchTerm}*))",
            ItemType.User => $"(&(objectClass=user)(cn=*{request.SearchTerm}*))",
            ItemType.Computer => $"(&(objectClass=computer)(cn=*{request.SearchTerm}*))",
            ItemType.Gpo => $"(&(objectClass=groupPolicyContainer)(displayName=*{request.SearchTerm}*))",
            _ => throw new ArgumentOutOfRangeException()
        };
    }

    private List<Group> Map(SearchResultCollection results)
    {
        var list = new List<Group>();

        foreach (SearchResult r in results)
        {
            list.Add(new Group
            {
                Name = Get(r, "cn"),
                DistinguishedName = Get(r, "distinguishedName")
            });
        }

        return list;
    }

    private string Get(SearchResult r, string name)
    {
        return r.Properties.Contains(name)
            ? r.Properties[name][0]?.ToString()
            : null;
    }
}
```

---

# 🚀 Usage

```csharp
var result = service.Execute(new GetGroupsRequest
{
    SearchTerm = "Admins",
    ItemType = ItemType.Group
});

if (result.IsSuccess)
{
    foreach (var g in result.Value)
        Console.WriteLine(g.Name);
}
else
{
    foreach (var error in result.Errors)
        Console.WriteLine(error.Message);
}
```

---

# 🔥 What You Achieved

### ❌ Before

* try/catch everywhere
* mixed responsibilities

### ✅ Now

* 🧼 Clean service (no exception handling)
* 🎯 Centralized error mapping
* 🧪 Fully testable
* 🔄 Consistent error model
* ⚡ Ready for pipelines / middleware

---

# 🧠 Optional Next Step (Highly Recommended)

I can extend this into:

### 🔥 Pipeline Behavior (like MediatR without MediatR)

* Validation auto-executed
* Result wrapping automatic
* Logging + metrics

### 🔥 Source Generator Mapper integration

* Replace `Map()` with generated code

### 🔥 Result<T> → HTTP mapping (Minimal API / Controllers)

---

Just tell me 👍

-----------------------
Autre demande
-----------------------

Perfect — this is the missing piece to make your stack **fully clean end-to-end** 🔥

We’ll map:

```csharp
Result<T>  →  IActionResult
```

in a **centralized, reusable way** for controllers.

---

# 🧩 Goal

Instead of this ❌:

```csharp
if (result.IsSuccess)
    return Ok(result.Value);
else
    return BadRequest(result.Errors);
```

You get this ✅:

```csharp
return result.ToActionResult();
```

---

# 🏗️ 1. Define Error Types (IMPORTANT)

👉 This lets us map errors → HTTP status codes cleanly

```csharp
using FluentResults;

public class NotFoundError : Error
{
    public NotFoundError(string message) : base(message) { }
}

public class ValidationError : Error
{
    public ValidationError(string message) : base(message) { }
}

public class UnauthorizedError : Error
{
    public UnauthorizedError(string message) : base(message) { }
}
```

---

# 🔁 2. Result → HTTP Mapper (Extension)

```csharp
using FluentResults;
using Microsoft.AspNetCore.Mvc;

public static class ResultExtensions
{
    public static IActionResult ToActionResult<T>(this Result<T> result)
    {
        if (result.IsSuccess)
        {
            return new OkObjectResult(result.Value);
        }

        var (status, errors) = MapErrors(result.Errors);

        return new ObjectResult(new
        {
            Errors = errors
        })
        {
            StatusCode = status
        };
    }

    private static (int status, List<string> errors) MapErrors(List<IError> errors)
    {
        // Default
        var status = StatusCodes.Status500InternalServerError;

        if (errors.Any(e => e is ValidationError))
            status = StatusCodes.Status400BadRequest;

        else if (errors.Any(e => e is NotFoundError))
            status = StatusCodes.Status404NotFound;

        else if (errors.Any(e => e is UnauthorizedError))
            status = StatusCodes.Status401Unauthorized;

        var messages = errors.Select(e => e.Message).ToList();

        return (status, messages);
    }
}
```

---

# 🧠 3. Plug FluentValidation → FluentResults

Update your service validation:

```csharp
if (!validation.IsValid)
{
    return Result.Fail(
        validation.Errors.Select(e => new ValidationError(e.ErrorMessage)));
}
```

---

# ⚙️ 4. Update Exception Mapping (GLOBAL)

```csharp
private static IError MapException(Exception ex)
{
    return ex switch
    {
        UnauthorizedAccessException =>
            new UnauthorizedError("Access denied"),

        KeyNotFoundException =>
            new NotFoundError("Resource not found"),

        _ => new Error("Unexpected error").CausedBy(ex)
    };
}
```

---

# 🎮 5. Controller Example

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/groups")]
public class GroupsController : ControllerBase
{
    private readonly GetGroupService _service;

    public GroupsController(GetGroupService service)
    {
        _service = service;
    }

    [HttpGet]
    public IActionResult Get([FromQuery] GetGroupsRequest request)
    {
        var result = _service.Execute(request);

        return result.ToActionResult(); // 🔥 clean
    }
}
```

---

# 📤 Example Responses

## ✅ Success

```json
{
  "value": [
    {
      "name": "Admins",
      "distinguishedName": "CN=Admins,DC=example,DC=com"
    }
  ]
}
```

---

## ❌ Validation Error (400)

```json
{
  "errors": [
    "SearchTerm must not be empty"
  ]
}
```

---

## ❌ Not Found (404)

```json
{
  "errors": [
    "Group not found"
  ]
}
```

---

## ❌ Server Error (500)

```json
{
  "errors": [
    "Unexpected error"
  ]
}
```

---

# 🧼 Optional: Cleaner Response Shape

If you don’t want `"value"` wrapper:

```csharp
if (result.IsSuccess)
{
    return new OkObjectResult(result.Value); // already clean
}
```

---

# 🔥 Advanced Upgrade (Recommended)

## 1. ProblemDetails (RFC 7807)

```csharp
return new ObjectResult(new ProblemDetails
{
    Title = "Request failed",
    Detail = string.Join(", ", errors),
    Status = status
})
{
    StatusCode = status
};
```

---

## 2. Base Controller (even cleaner)

```csharp
public abstract class ApiControllerBase : ControllerBase
{
    protected IActionResult Handle<T>(Result<T> result)
        => result.ToActionResult();
}
```

Usage:

```csharp
return Handle(_service.Execute(request));
```

---

# 🧠 Final Architecture

```id="c76cz3"
[ Controller ]
    ↓ (ToActionResult)
[ Result<T> ]
    ↓
[ GetGroupService ]
    ↓
[ ActiveDirectoryService ]
```

---

# 🚀 What You Now Have

* ✅ No try/catch in controllers
* ✅ No if/else clutter
* ✅ Centralized HTTP mapping
* ✅ Strongly typed errors
* ✅ Clean, testable architecture

---

# 👉 If you want next level

I can extend this to:

* 🔥 Minimal APIs version (`IResult`)
* 🔥 Global ASP.NET Core filter (no extension needed)
* 🔥 Full Clean Architecture template
* 🔥 Logging + tracing integration
* 🔥 OpenAPI/Swagger error typing

Just tell me 👍
