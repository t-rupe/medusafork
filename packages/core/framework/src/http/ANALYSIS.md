# HTTP Layer Analysis - MedusaJS Framework

## Purpose
Express-based HTTP layer providing file-based routing, middleware pipeline, authentication, and request handling for API endpoints.

## Architecture

### Core Components

**1. Express Initialization** (`express-loader.ts`)
- Configures Express app with session management (Redis/DynamoDB)
- Sets up HTTP logging (Morgan + Winston)
- Cookie parsing, compression, static file serving
- Trust proxy configuration for production

```typescript
// Session storage: Redis or DynamoDB
sessionOpts.store = new RedisStore({
  client: redisClient,
  prefix: `sess:`
})
```

**2. Route Discovery** (`ApiLoader` in `router.ts`)
- Scans directories for `route.ts` files
- Discovers middleware via file loaders
- Sorts routes by specificity (static → params → regex → wildcard)
- Caches route matches for performance

**3. File-Based Routing Pattern**
```
/api/admin/products/route.ts       → GET /admin/products
/api/admin/products/[id]/route.ts  → GET /admin/products/:id
```

Route files export named HTTP methods:
```typescript
export const GET = async (
  req: AuthenticatedMedusaRequest<ParamsType>,
  res: MedusaResponse<ResponseType>
) => {
  // Handler logic
}
```

**4. Request Pipeline**
```
Request → Logging → Cookie Parser → Session → Body Parser →
CORS → Auth → Route-Specific Middleware → Handler → Error Handler
```

**5. Authentication** (`authenticate-middleware.ts`)
Three auth strategies:
- **Session**: Express session with Redis/DynamoDB
- **Bearer**: JWT token verification
- **API Key**: Basic auth with `sk_` prefix tokens

```typescript
authenticate(
  actorType: "user" | "customer" | string[],
  authType: ["bearer", "session", "api-key"],
  { allowUnauthenticated?: boolean }
)
```

**6. CORS Handling**
Per-namespace CORS policies:
- `/admin` - Admin dashboard CORS
- `/store` - Storefront CORS
- `/auth` - Auth endpoints CORS

Routes can opt-out via `shouldAppendAdminCors: false`

**7. Request Object Extensions** (`types.ts`)
```typescript
interface MedusaRequest extends Express.Request {
  scope: MedusaContainer        // DI container
  validatedBody: Body           // Zod-validated body
  validatedQuery: QueryFields   // Validated query params
  filterableFields: QueryFields // Parsed filters
  queryConfig: {                // Remote query config
    fields: string[]
    pagination: { skip, take, order }
  }
  auth_context?: AuthContext    // Authenticated user info
}
```

**8. Body Parser Configuration**
Routes specify body parser config via middleware:
```typescript
{
  matcher: "/admin/products",
  methods: ["POST"],
  bodyParser: {
    sizeLimit: "5mb",
    preserveRawBody: true
  }
}
```

**9. Route Registration Flow**
```
RoutesLoader.scanDir() → Find route.ts files →
RoutesSorter.sort() → Order by specificity →
RoutesFinder.add() → Build regex matchers →
Express.METHOD(path, handler) → Register
```

**10. Middleware Types**
- **Global**: Applied to all routes via `USE`
- **Route-specific**: Defined in middleware config
- **Namespace**: Applied to `/admin`, `/store`, `/auth`

See: `middleware-file-loader.ts`, `define-middlewares.ts`

## Key Features

### 1. Query Config Builder
Transforms query params to structured config:
```typescript
?fields=id,name&limit=10&offset=0
// Becomes:
req.queryConfig = {
  fields: ["id", "name"],
  pagination: { skip: 0, take: 10 }
}
```
See: `utils/get-query-config.ts`

### 2. Entity Refetching
Helper to fetch entities from any module:
```typescript
await refetchEntity({
  entity: "product",
  idOrFilter: req.params.id,
  scope: req.scope,
  fields: req.queryConfig.fields
})
```
See: `utils/refetch-entities.ts`

### 3. Validation Integration
Zod schemas for body/query validation:
```typescript
validateBody(schema: ZodObject) // Middleware
validateQuery(schema: ZodObject) // Middleware
```
See: `utils/validate-body.ts`, `utils/validate-query.ts`

### 4. Error Handling
Centralized error handler with exception formatting:
- Transforms errors to consistent JSON format
- Logs with request context
- Returns appropriate HTTP status codes

See: `middlewares/error-handler.ts`, `middlewares/exception-formatter.ts`

## .NET Mapping with FastEndpoints

### 1. Replace Express with FastEndpoints
**Medusa**: Express router with manual route registration
**FastEndpoints**: Auto-discovery endpoints via assembly scanning

```csharp
// FastEndpoints equivalent
public class GetProductsEndpoint : Endpoint<GetProductsRequest, GetProductsResponse>
{
    public override void Configure()
    {
        Get("/admin/products");
        Policies("AdminOnly");
    }

    public override async Task HandleAsync(GetProductsRequest req, CancellationToken ct)
    {
        // Handler logic
    }
}
```

### 2. Dependency Injection
**Medusa**: Awilix container in `req.scope`
**FastEndpoints**: Built-in ASP.NET Core DI

```csharp
public override async Task HandleAsync(...)
{
    var service = Resolve<IProductService>(); // FastEndpoints
    // or constructor injection
}
```

### 3. Authentication
**Medusa**: Custom middleware with session/JWT/API key
**.NET**: ASP.NET Core Authentication middleware

```csharp
builder.Services.AddAuthentication()
    .AddJwtBearer()
    .AddCookie("Session");

// In endpoint:
public override void Configure()
{
    AuthSchemes("Bearer", "Session");
}
```

### 4. Request Validation
**Medusa**: Zod schemas
**FastEndpoints**: FluentValidation

```csharp
public class GetProductsValidator : Validator<GetProductsRequest>
{
    public GetProductsValidator()
    {
        RuleFor(x => x.Limit).InclusiveBetween(1, 100);
    }
}
```

### 5. Route Discovery
**Medusa**: File-system scanning for `route.ts`
**FastEndpoints**: Assembly scanning for `Endpoint<,>` classes

Both use convention-based discovery, no manual registration needed.

### 6. Middleware Pipeline
**Medusa**: Express middleware chain
**.NET**: ASP.NET Core middleware pipeline

```csharp
app.UseAuthentication();
app.UseAuthorization();
app.UseFastEndpoints(c => {
    c.Endpoints.RoutePrefix = "api";
});
```

### 7. CORS
**Medusa**: Per-namespace CORS
**.NET**: CORS policy per endpoint

```csharp
public override void Configure()
{
    Policies("AdminCorsPolicy");
}
```

### 8. Query Config Pattern (REPR/VSA)
**Medusa**: `req.queryConfig` with fields/pagination
**FastEndpoints**: Query objects with mapping

```csharp
public class GetProductsRequest
{
    public string[] Fields { get; set; }
    public int Skip { get; set; }
    public int Take { get; set; }
}
```

### 9. Entity Refetching Pattern
**Medusa**: `refetchEntity()` helper
**.NET**: Repository pattern or Mediator

```csharp
// Using Mediator (REPR pattern)
var product = await SendAsync(new GetProductQuery
{
    Id = req.Id,
    Fields = req.Fields
});
```

### 10. Error Handling
**Medusa**: Global error middleware
**FastEndpoints**: Global exception handler

```csharp
app.UseExceptionHandler("/error");
// or FastEndpoints error response
await SendErrorsAsync(400, ct);
```

## Implementation Recommendations

### .NET Project Structure
```
/Endpoints
  /Admin
    /Products
      GetProducts.cs           // Endpoint
      GetProductsRequest.cs    // Request DTO
      GetProductsResponse.cs   // Response DTO
      GetProductsValidator.cs  // Validator
  /Store
    /Products
      ...
```

### Key .NET Packages
- **FastEndpoints** - Endpoint routing
- **FluentValidation** - Request validation
- **Microsoft.AspNetCore.Authentication.JwtBearer** - JWT auth
- **Swashbuckle.AspNetCore** - OpenAPI/Swagger
- **MediatR** (optional) - REPR pattern

### FastEndpoints Configuration
```csharp
builder.Services.AddFastEndpoints();
builder.Services.AddAuthentication(...)
    .AddJwtBearer()
    .AddCookie();

app.UseAuthentication();
app.UseAuthorization();
app.UseFastEndpoints(c => {
    c.Endpoints.RoutePrefix = "api";
    c.Endpoints.Configurator = ep => {
        // Global endpoint config
    };
});
```

### Request Context (Similar to MedusaRequest)
```csharp
public class MedusaContext
{
    public required IServiceProvider Scope { get; init; }
    public required ClaimsPrincipal User { get; init; }
    public QueryConfig QueryConfig { get; set; }
    public Dictionary<string, object> FilterableFields { get; set; }
}

// Access via HttpContext.Items or custom base endpoint
```

### Session Management
**Medusa**: Redis/DynamoDB sessions
**.NET**: StackExchange.Redis with distributed cache

```csharp
builder.Services.AddStackExchangeRedisCache(options => {
    options.Configuration = "localhost:6379";
});
builder.Services.AddSession(options => {
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});
```

## Key Differences

| Aspect | MedusaJS | .NET FastEndpoints |
|--------|----------|-------------------|
| Routing | File-based (`route.ts`) | Class-based (`Endpoint<,>`) |
| DI Container | Awilix (manual) | Built-in (automatic) |
| Validation | Zod | FluentValidation |
| Auth | Custom middleware | ASP.NET Core Identity |
| Middleware | Express chain | ASP.NET Core pipeline |
| Route Discovery | FS scanning | Assembly scanning |
| Request Type | Extended Express.Request | Strongly-typed DTO |

## References

**Core Files:**
- `framework/src/http/router.ts` - Route registration
- `framework/src/http/express-loader.ts` - Express setup
- `framework/src/http/routes-finder.ts` - Route matching
- `framework/src/http/middlewares/authenticate-middleware.ts` - Auth
- `framework/src/http/types.ts` - Type definitions

**Example Routes:**
- `medusa/src/api/admin/notifications/route.ts`
- `medusa/src/api/admin/notifications/[id]/route.ts`
