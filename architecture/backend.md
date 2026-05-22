# Backend Reusability Between Applications

## 1. The Most Common Way: Consuming the Backend API

Your backend acts as a REST API/server, and the other program makes HTTP requests to it.

### Example

### Backend Endpoints

```http
GET /api/users
POST /api/login
```

The other project consumes this using `HttpClient`.

### Example in C#

```csharp
using System.Net.Http;
using System.Net.Http.Json;

HttpClient client = new HttpClient();

var users = await client.GetFromJsonAsync<List<User>>(
    "https://localhost:5001/api/users"
);
```

### Why This Is the Most Professional Approach

- It separates projects
- You can reuse the backend
- It works for web, mobile, desktop, etc.
- Avoids duplicating logic

---

# 2. Share Logic with a Shared Library/Class

If you want to reuse internal code and avoid HTTP requests, you can move the logic to a project like:

```text
MyApp.Core
```

and then reference it in both programs.

## Example Structure

```text
Solution
│
├── BackendAPI
├── DesktopApp
└── SharedServices
```

## Example in SharedServices

```csharp
public class UserService
{
    public string GetRole()
    {
        return "Admin";
    }
}
```

Then both projects use:

```csharp
using SharedServices;
```

### This Is Useful When

- Both projects are in the same solution
- You want to share validations
- Business rules
- Helpers
- JWT logic
- Utilities

---

# 3. Using Services with Dependency Injection

In ASP.NET, services are typically registered like this:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

And then used like this:

```csharp
public class HomeController
{
    private readonly IUserService _userService;

    public HomeController(IUserService userService)
    {
        _userService = userService;
    }
}
```

This allows ASP.NET to automatically inject the service wherever it is needed.

---

# Recommended Architecture

A common ASP.NET project structure looks like this:

```text
Controllers
Services
Repositories
Models
DTOs
```

## Responsibilities

- Controllers → Handle HTTP requests
- Services → Business logic
- Repositories → Database access
- Models → Data representation
- DTOs → Data transfer objects

# Architecture

## Backends

### 1. Main Backend — .NET

Used by:

- Front Admin
- Front Employee
- Front Client

Responsibilities:

- Authentication
- Users
- Metrics
- Events
- Heavy business logic
- Redis
- PostgreSQL

Technologies:

- ASP.NET Core
- PostgreSQL
- Redis

---

### 2. Secondary Backend — Laravel

Completely decoupled. Ideal for:

- Microservices
- External APIs
- Specific modules
- Integrations
- Technical proofs of concept
- CMS
- Notifications

Technologies:

- Laravel
- PHP

---

## Frontends

Strongly recommended: **React**

With multiple panels to manage, React is a great fit thanks to:

- Reusable components
- Routing
- State management
- API integration
- Dashboards
- Scalability 
