# Sprint 2 — Gateway Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the YARP-based API Gateway as the single front door for all backend services. Cookie session validation against Redis (issued by Identity), JWT issuance for downstream services (Cookie→JWT hybrid), tier-based rate limiting, OWASP security headers, Polly resilience, URL versioning (`/api/v1/...`), and observability.

**Architecture:** YARP reverse proxy (.NET 10) running as ASP.NET Core service. Cookie auth scheme decrypts `skillvio.sid` (Redis-backed DataProtection key ring shared with Identity service). On valid cookie, gateway issues a short-lived JWT (HS256, 5-min TTL) and forwards request to downstream service with `Authorization: Bearer ...`. Rate limiter partitions by user-or-IP per-policy. WebSocket pass-through for `/api/v1/ws/*` (needed by Sprint 3 SignalR).

**Tech Stack:** .NET 10 LTS, ASP.NET Core 10, YARP 2.x, Cookies + DataProtection.StackExchangeRedis, System.IdentityModel.Tokens.Jwt, Polly v8, ASP.NET Core RateLimiter, Serilog → Loki, OpenTelemetry → Tempo, Prometheus, xUnit + Testcontainers.

**Spec:** `docs/superpowers/specs/2026-05-04-sprint-2-gateway-design.md`

**Effort:** ~10 work-days, 13 tasks (T1-T13). Solo dev.

**Repo:** `skillvio-gateway` (skeleton on `main`, remote `git@github.com:skillvio/skillvio-gateway.git`).

**Dependencies:**
- ✅ Sprint 1 Identity Service v0.1.0-sprint1 (cookie scheme, DataProtection key ring in Redis, user/role state)
- ✅ skillvio-shared-contracts v0.1.0 (events, not directly consumed but on classpath)
- ✅ skillvio-infra docker-compose (Postgres + Redis + RabbitMQ + observability)

---

## Pre-Flight Checklist

Before T1, verify:

- [ ] Sprint 1 Identity Service is running locally (`dotnet run --project src/Skillvio.Identity.Api` from skillvio-identity-service repo) on port `5169` (or similar)
- [ ] Redis is running (used for shared DataProtection keys + rate limiter backing if needed): `cd skillvio-infra && docker compose up -d redis postgres`
- [ ] `psql -h localhost -p 55432 -U skillvio -d gateway_db -c '\l'` — gateway has its own DB? **Per spec**, gateway uses Postgres only for `compliance_audit` rows it owns; otherwise stateless. We'll add `gateway_db` to the infra `init-databases.sh` if not present.
- [ ] Confirm Identity DataProtection key ring location: `skillvio-identity:dataprotection-keys` (Redis key prefix). Gateway will read from same prefix to decrypt cookies.

---

## Task 1 — Solution Skeleton (.NET 10 + YARP)

**Spec ref:** §4 Mimari, §8 S2.T1
**Estimated:** 0.5 day
**Files:**
- Create: `skillvio-gateway.sln`, `Directory.Build.props`, `Directory.Packages.props`, `global.json`, `nuget.config`, `.editorconfig`, `.gitignore`
- Create: `src/Skillvio.Gateway/Skillvio.Gateway.csproj` (single project — gateway is a focused web app, no Clean Arch split needed for routing)
- Create: `src/Skillvio.Gateway/Program.cs`
- Create: `tests/Skillvio.Gateway.IntegrationTests/Skillvio.Gateway.IntegrationTests.csproj`
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Verify .NET 10 SDK + remote**

```bash
cd /Users/gokhannihal/skillvio/skillvio-gateway
export PATH="$HOME/.dotnet:$PATH"
dotnet --version
git status
```

- [ ] **Step 2: Write `global.json`** (pin SDK 10.0.203 like Sprint 1)

- [ ] **Step 3: Write `Directory.Build.props`** (same as identity-service: net10.0, Nullable, ImplicitUsings, TreatWarningsAsErrors)

- [ ] **Step 4: Write `Directory.Packages.props`**

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- YARP -->
    <PackageVersion Include="Yarp.ReverseProxy" Version="2.2.0" />
    <!-- Auth -->
    <PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="10.0.0" />
    <PackageVersion Include="Microsoft.IdentityModel.Tokens" Version="8.1.0" />
    <PackageVersion Include="System.IdentityModel.Tokens.Jwt" Version="8.1.0" />
    <!-- DataProtection (shared with Identity) -->
    <PackageVersion Include="Microsoft.AspNetCore.DataProtection.StackExchangeRedis" Version="10.0.7" />
    <PackageVersion Include="StackExchange.Redis" Version="2.8.16" />
    <!-- EF Core (compliance_audit only) -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.0.7" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.7" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.7" />
    <PackageVersion Include="EFCore.NamingConventions" Version="10.0.1" />
    <!-- Resilience -->
    <PackageVersion Include="Microsoft.Extensions.Http.Resilience" Version="9.0.0" />
    <PackageVersion Include="Polly" Version="8.4.2" />
    <!-- Observability -->
    <PackageVersion Include="Serilog.AspNetCore" Version="8.0.3" />
    <PackageVersion Include="Serilog.Sinks.Grafana.Loki" Version="8.3.0" />
    <PackageVersion Include="OpenTelemetry.Extensions.Hosting" Version="1.15.3" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.10.1" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.Http" Version="1.10.0" />
    <PackageVersion Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.15.3" />
    <PackageVersion Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" Version="1.10.0-beta.1" />
    <!-- Health -->
    <PackageVersion Include="AspNetCore.HealthChecks.NpgSql" Version="9.0.0" />
    <PackageVersion Include="AspNetCore.HealthChecks.Redis" Version="9.0.0" />
    <PackageVersion Include="AspNetCore.HealthChecks.Uris" Version="9.0.0" />
    <!-- Test -->
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
    <PackageVersion Include="xunit" Version="2.9.3" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="3.1.4" />
    <PackageVersion Include="FluentAssertions" Version="6.12.1" />
    <PackageVersion Include="Microsoft.AspNetCore.Mvc.Testing" Version="10.0.0" />
    <PackageVersion Include="Testcontainers.PostgreSql" Version="3.10.0" />
    <PackageVersion Include="Testcontainers.Redis" Version="3.10.0" />
    <PackageVersion Include="NSubstitute" Version="5.1.0" />
  </ItemGroup>
</Project>
```

- [ ] **Step 5: Copy `.editorconfig` + `.gitignore` from skillvio-identity-service** for consistency

- [ ] **Step 6: Create solution + Gateway project + test project**

```bash
dotnet new sln -n skillvio-gateway --format sln
mkdir -p src/Skillvio.Gateway tests/Skillvio.Gateway.IntegrationTests
cd src/Skillvio.Gateway && dotnet new web -f net10.0 --force && cd ../..
cd tests/Skillvio.Gateway.IntegrationTests && dotnet new xunit -f net10.0 --force && rm UnitTest1.cs && cd ../..
dotnet sln add src/Skillvio.Gateway/Skillvio.Gateway.csproj tests/Skillvio.Gateway.IntegrationTests/Skillvio.Gateway.IntegrationTests.csproj
dotnet add tests/Skillvio.Gateway.IntegrationTests reference src/Skillvio.Gateway
```

- [ ] **Step 7: Add baseline PackageReferences** to `Skillvio.Gateway.csproj`:

```xml
<ItemGroup>
  <PackageReference Include="Yarp.ReverseProxy" />
  <PackageReference Include="Serilog.AspNetCore" />
</ItemGroup>
```

To test project add: `Microsoft.NET.Test.Sdk`, `xunit`, `xunit.runner.visualstudio`, `FluentAssertions`, `Microsoft.AspNetCore.Mvc.Testing`, `Testcontainers.Redis`.

- [ ] **Step 8: Minimal `Program.cs`**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy().LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
var app = builder.Build();
app.MapGet("/", () => "Skillvio Gateway");
app.MapReverseProxy();
app.Run();
```

- [ ] **Step 9: Build**

```bash
dotnet build
```

Expected: 0/0.

- [ ] **Step 10: Write CI workflow** (Postgres + Redis services, dotnet test, same pattern as identity)

- [ ] **Step 11: Commit + push**

```bash
git add -A
git commit -m "chore(s2-t1): scaffold YARP gateway solution"
git push -u origin main
```

---

## Task 2 — YARP Route Configuration

**Spec ref:** §5 YARP Configuration, §8 S2.T2
**Estimated:** 0.5 day
**Files:**
- Create: `src/Skillvio.Gateway/appsettings.json` (YARP routes/clusters)
- Create: `src/Skillvio.Gateway/appsettings.Development.json` (dev overrides)

- [ ] **Step 1: Write YARP config in `appsettings.json`**

```json
{
  "ReverseProxy": {
    "Routes": {
      "auth-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/auth/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/v1/auth/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "users-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/users/{**rest}" },
        "AuthorizationPolicy": "RequireAuthenticated"
      },
      "certifications-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/certifications/{**rest}" },
        "AuthorizationPolicy": "RequireAuthenticated"
      },
      "onboarding-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/onboarding/{**rest}" },
        "AuthorizationPolicy": "RequireAuthenticated"
      },
      "admin-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/admin/{**rest}" },
        "AuthorizationPolicy": "RequireAdmin"
      },
      "mfa-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/mfa/{**rest}" },
        "AuthorizationPolicy": "RequireAuthenticated"
      },
      "learning-v1": {
        "ClusterId": "learning",
        "Match": { "Path": "/api/v1/learning/{**rest}" }
      },
      "me-v1": {
        "ClusterId": "learning",
        "Match": { "Path": "/api/v1/me/{**rest}" },
        "AuthorizationPolicy": "RequireAuthenticated"
      },
      "attempts-v1": {
        "ClusterId": "learning",
        "Match": { "Path": "/api/v1/attempts/{**rest}" },
        "AuthorizationPolicy": "RequireAuthenticated",
        "RateLimiterPolicy": "LessonSubmit"
      },
      "exam-access-v1": {
        "ClusterId": "learning",
        "Match": { "Path": "/api/v1/exam-access/{**rest}" },
        "AuthorizationPolicy": "RequireAuthenticated"
      },
      "ws-v1": {
        "ClusterId": "learning",
        "Match": { "Path": "/api/v1/ws/{**rest}" },
        "AuthorizationPolicy": "RequireAuthenticated"
      }
    },
    "Clusters": {
      "identity": {
        "Destinations": {
          "i1": { "Address": "http://identity-api:8080/" }
        },
        "HealthCheck": {
          "Active": { "Enabled": true, "Interval": "00:00:15", "Path": "/health/ready" },
          "Passive": { "Enabled": true, "Policy": "TransportFailureRate", "ReactivationPeriod": "00:00:30" }
        },
        "HttpRequest": { "ActivityTimeout": "00:00:30" }
      },
      "learning": {
        "Destinations": {
          "l1": { "Address": "http://learning-api:8080/" }
        },
        "HealthCheck": {
          "Active": { "Enabled": true, "Interval": "00:00:15", "Path": "/health/ready" }
        },
        "HttpRequest": { "ActivityTimeout": "00:00:30" }
      }
    }
  }
}
```

- [ ] **Step 2: Dev overrides** in `appsettings.Development.json`:

```json
{
  "ReverseProxy": {
    "Clusters": {
      "identity": {
        "Destinations": {
          "i1": { "Address": "http://localhost:5169/" }
        }
      },
      "learning": {
        "Destinations": {
          "l1": { "Address": "http://localhost:5170/" }
        }
      }
    }
  }
}
```

- [ ] **Step 3: Commit**

```bash
git commit -am "feat(s2-t2): YARP route config (11 routes, 2 clusters, health checks)"
git push
```

---

## Task 3 — Cookie Auth + Redis Session Validation

**Spec ref:** §6 Middleware Pipeline (cookie auth), §8 S2.T3
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Gateway/Auth/CookieAuthConfig.cs`
- Create: `src/Skillvio.Gateway/Auth/SkillvioCookieAuthHandler.cs` (custom handler if needed for cross-service decryption)
- Modify: `src/Skillvio.Gateway/Program.cs`
- Create: `tests/Skillvio.Gateway.IntegrationTests/Auth/CookieValidationTests.cs`

- [ ] **Step 1: Add packages**

```bash
dotnet add src/Skillvio.Gateway package Microsoft.AspNetCore.Authentication.Cookies
dotnet add src/Skillvio.Gateway package Microsoft.AspNetCore.DataProtection.StackExchangeRedis
dotnet add src/Skillvio.Gateway package StackExchange.Redis
```

- [ ] **Step 2: Configure Cookie auth + shared DataProtection**

```csharp
// src/Skillvio.Gateway/Auth/CookieAuthConfig.cs
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.DataProtection;
using StackExchange.Redis;

namespace Skillvio.Gateway.Auth;

public static class CookieAuthConfig
{
    public const string SchemeName = "Skillvio.Cookies";  // MUST match Identity's scheme

    public static IServiceCollection AddSkillvioGatewayAuth(this IServiceCollection services, IConfiguration config)
    {
        var redisCs = config.GetConnectionString("Redis") ?? "localhost:6379";
        var redis = ConnectionMultiplexer.Connect(redisCs);
        services.AddSingleton<IConnectionMultiplexer>(redis);

        services.AddDataProtection()
            .PersistKeysToStackExchangeRedis(redis, "skillvio-identity:dataprotection-keys")
            .SetApplicationName("skillvio");

        services.AddAuthentication(SchemeName)
            .AddCookie(SchemeName, opt =>
            {
                opt.Cookie.Name = "skillvio.sid";
                opt.Cookie.HttpOnly = true;
                opt.Cookie.SecurePolicy = CookieSecurePolicy.Always;
                opt.Cookie.SameSite = SameSiteMode.Lax;
                opt.Cookie.Domain = config["Cookies:Domain"];
                opt.SlidingExpiration = false;
                opt.Events.OnRedirectToLogin = ctx => { ctx.Response.StatusCode = 401; return Task.CompletedTask; };
                opt.Events.OnRedirectToAccessDenied = ctx => { ctx.Response.StatusCode = 403; return Task.CompletedTask; };
            });

        services.AddAuthorization(opt =>
        {
            opt.AddPolicy("RequireAuthenticated", p => p.RequireAuthenticatedUser());
            opt.AddPolicy("RequireAdmin", p => p.RequireAuthenticatedUser().RequireRole("admin"));
        });

        return services;
    }
}
```

- [ ] **Step 3: Wire in Program.cs**

```csharp
builder.Services.AddSkillvioGatewayAuth(builder.Configuration);
// ...
app.UseAuthentication();
app.UseAuthorization();
app.MapReverseProxy();
```

- [ ] **Step 4: Cross-service cookie test**

The critical test: a cookie issued by Identity service must be validated by Gateway. Both share same Redis backplane. This depends on:
1. Same `ApplicationName` (`"skillvio"`)
2. Same Redis key prefix (`"skillvio-identity:dataprotection-keys"`)
3. Same cookie scheme name (`"Skillvio.Cookies"`)

```csharp
[Fact]
public async Task Cookie_IssuedByIdentity_ValidatedByGateway()
{
    // Arrange: spin up Identity test factory + Gateway test factory, share Redis container
    using var identity = new IdentityFactory(redis);
    using var gateway = new GatewayFactory(redis);

    // Issue cookie via identity
    var loginResp = await identity.Client.PostAsJsonAsync("/api/v1/auth/login", new { ... });
    var cookies = loginResp.Headers.GetValues("Set-Cookie").ToList();

    // Forward to gateway
    var gwReq = new HttpRequestMessage(HttpMethod.Get, "/api/v1/users/me");
    gwReq.Headers.Add("Cookie", string.Join("; ", cookies.Select(ExtractNameValue)));
    var gwResp = await gateway.Client.SendAsync(gwReq);

    gwResp.StatusCode.Should().Be(HttpStatusCode.OK);  // 401 if cross-service decrypt fails
}
```

- [ ] **Step 5: Commit**

```bash
git commit -am "feat(s2-t3): cookie auth + Redis DataProtection backplane (shared with Identity)"
git push
```

---

## Task 4 — JWT Issuer Middleware (Cookie → JWT Hybrid)

**Spec ref:** §3 Soru2 (D), §6 JWT Issuer middleware, §8 S2.T4
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Gateway/Auth/IGatewayJwtIssuer.cs`
- Create: `src/Skillvio.Gateway/Auth/GatewayJwtIssuer.cs`
- Create: `src/Skillvio.Gateway/Middleware/JwtIssuerMiddleware.cs`
- Modify: `src/Skillvio.Gateway/Program.cs`
- Create: `tests/Skillvio.Gateway.IntegrationTests/Auth/JwtIssuerTests.cs`

- [ ] **Step 1: Add packages**

```bash
dotnet add src/Skillvio.Gateway package System.IdentityModel.Tokens.Jwt
```

- [ ] **Step 2: Implement IGatewayJwtIssuer**

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.IdentityModel.Tokens;

namespace Skillvio.Gateway.Auth;

public interface IGatewayJwtIssuer
{
    string IssueForDownstream(ClaimsPrincipal principal, TimeSpan? lifetime = null);
}

public class GatewayJwtIssuer(IConfiguration config) : IGatewayJwtIssuer
{
    public string IssueForDownstream(ClaimsPrincipal principal, TimeSpan? lifetime = null)
    {
        var key = config["Jwt:SigningKey"]
            ?? throw new InvalidOperationException("Jwt:SigningKey not configured");
        var keyBytes = Encoding.UTF8.GetBytes(key);
        if (keyBytes.Length < 32) throw new InvalidOperationException("Jwt key must be ≥ 32 bytes");

        var creds = new SigningCredentials(new SymmetricSecurityKey(keyBytes), SecurityAlgorithms.HmacSha256);
        var jti = Guid.NewGuid().ToString();
        var sid = principal.FindFirstValue(ClaimTypes.NameIdentifier) ?? throw new InvalidOperationException("No sub claim");

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, sid),
            new(JwtRegisteredClaimNames.Jti, jti),
            new("sid", sid)  // session id alias
        };
        // Pass through roles + plan + locale
        claims.AddRange(principal.FindAll(ClaimTypes.Role).Select(c => new Claim("role", c.Value)));
        var plan = principal.FindFirstValue("plan");
        if (plan is not null) claims.Add(new Claim("plan", plan));

        var token = new JwtSecurityToken(
            issuer: "skillvio-gateway",
            audience: "skillvio-services",
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: DateTime.UtcNow.Add(lifetime ?? TimeSpan.FromMinutes(5)),
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

- [ ] **Step 3: JwtIssuerMiddleware**

Issues a JWT after authentication and adds `Authorization: Bearer ...` header for downstream proxy. Runs after `UseAuthentication` but before `MapReverseProxy`.

```csharp
public class JwtIssuerMiddleware(RequestDelegate next, IGatewayJwtIssuer issuer)
{
    public async Task InvokeAsync(HttpContext ctx)
    {
        if (ctx.User.Identity?.IsAuthenticated == true)
        {
            var jwt = issuer.IssueForDownstream(ctx.User);
            ctx.Request.Headers.Authorization = $"Bearer {jwt}";
            // Strip the cookie before forwarding (downstream doesn't need it)
            ctx.Request.Headers.Remove("Cookie");
            // Pass through user_id header for services that don't validate JWT yet
            var userId = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier);
            if (userId is not null) ctx.Request.Headers["X-User-Id"] = userId;
            var roles = ctx.User.FindAll(ClaimTypes.Role).Select(r => r.Value).ToArray();
            if (roles.Length > 0) ctx.Request.Headers["X-User-Role"] = string.Join(",", roles);
            var plan = ctx.User.FindFirstValue("plan");
            if (plan is not null) ctx.Request.Headers["X-User-Plan"] = plan;
        }
        await next(ctx);
    }
}
```

- [ ] **Step 4: Register middleware**

```csharp
builder.Services.AddSingleton<IGatewayJwtIssuer, GatewayJwtIssuer>();

// in pipeline:
app.UseAuthentication();
app.UseAuthorization();
app.UseMiddleware<JwtIssuerMiddleware>();
app.MapReverseProxy();
```

- [ ] **Step 5: Test JWT generation**

```csharp
[Fact]
public void Issue_GeneratesValidJwt_WithExpectedClaims()
{
    var issuer = new GatewayJwtIssuer(BuildConfig("test-signing-key-32-chars-long-x"));
    var principal = BuildPrincipal(userId: "abc", roles: ["user"], plan: "free");

    var token = issuer.IssueForDownstream(principal);

    var parsed = new JwtSecurityTokenHandler().ReadJwtToken(token);
    parsed.Claims.Should().Contain(c => c.Type == "sub" && c.Value == "abc");
    parsed.Claims.Should().Contain(c => c.Type == "role" && c.Value == "user");
    parsed.Claims.Should().Contain(c => c.Type == "plan" && c.Value == "free");
    parsed.ValidTo.Should().BeAfter(DateTime.UtcNow.AddMinutes(4));
}
```

- [ ] **Step 6: Commit**

```bash
git commit -am "feat(s2-t4): JWT issuer middleware (Cookie → JWT hybrid for downstream services)"
git push
```

---

## Task 5 — Rate Limiter (Tier-Based Policies)

**Spec ref:** §3 Soru3 (C+E hibrit), §8 S2.T5
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Gateway/RateLimiting/RateLimitPolicies.cs`
- Modify: `Program.cs`
- Create: `tests/Skillvio.Gateway.IntegrationTests/RateLimiting/RateLimitTests.cs`

- [ ] **Step 1: Define policies extension**

```csharp
public static class RateLimitPolicies
{
    public static IServiceCollection AddSkillvioRateLimiting(this IServiceCollection services, IConfiguration config)
    {
        services.AddRateLimiter(opt =>
        {
            opt.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
            opt.OnRejected = async (ctx, ct) =>
            {
                ctx.HttpContext.Response.StatusCode = 429;
                ctx.HttpContext.Response.Headers["Retry-After"] = "60";
                await ctx.HttpContext.Response.WriteAsJsonAsync(new { error = "rate_limit_exceeded" }, ct);
            };

            opt.AddPolicy("GeneralApi", ctx =>
            {
                var plan = ctx.User.FindFirstValue("plan") ?? "anon";
                var (limit, window) = plan switch
                {
                    "enterprise" => (int.MaxValue, TimeSpan.FromMinutes(1)),
                    "team" => (1000, TimeSpan.FromMinutes(1)),
                    "pro" => (500, TimeSpan.FromMinutes(1)),
                    "free" => (200, TimeSpan.FromMinutes(1)),
                    _ => (50, TimeSpan.FromMinutes(1))
                };
                var key = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier) ?? GetClientIp(ctx);
                return RateLimitPartition.GetFixedWindowLimiter(key, _ =>
                    new FixedWindowRateLimiterOptions { PermitLimit = limit, Window = window, AutoReplenishment = true });
            });

            opt.AddPolicy("LessonSubmit", ctx =>
            {
                var plan = ctx.User.FindFirstValue("plan") ?? "anon";
                var limit = plan switch { "pro" or "team" or "enterprise" => 200, "free" => 60, _ => 20 };
                var key = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier) ?? GetClientIp(ctx);
                return RateLimitPartition.GetFixedWindowLimiter(key, _ =>
                    new FixedWindowRateLimiterOptions { PermitLimit = limit, Window = TimeSpan.FromMinutes(1), AutoReplenishment = true });
            });

            opt.AddPolicy("AuthLogin", ctx =>
                RateLimitPartition.GetFixedWindowLimiter(GetClientIp(ctx), _ =>
                    new FixedWindowRateLimiterOptions { PermitLimit = 5, Window = TimeSpan.FromMinutes(15), AutoReplenishment = true }));

            opt.AddPolicy("AuthSignup", ctx =>
                RateLimitPartition.GetFixedWindowLimiter(GetClientIp(ctx), _ =>
                    new FixedWindowRateLimiterOptions { PermitLimit = 3, Window = TimeSpan.FromHours(1), AutoReplenishment = true }));

            opt.AddPolicy("ExamStart", ctx =>
            {
                var key = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier) ?? GetClientIp(ctx);
                return RateLimitPartition.GetFixedWindowLimiter(key, _ =>
                    new FixedWindowRateLimiterOptions { PermitLimit = 10, Window = TimeSpan.FromHours(1), AutoReplenishment = true });
            });

            opt.AddPolicy("AiQuota", ctx =>
            {
                var plan = ctx.User.FindFirstValue("plan") ?? "anon";
                var limit = plan switch { "enterprise" => 500, "team" => 200, "pro" => 100, "free" => 20, _ => 5 };
                var key = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier) ?? GetClientIp(ctx);
                return RateLimitPartition.GetFixedWindowLimiter(key, _ =>
                    new FixedWindowRateLimiterOptions { PermitLimit = limit, Window = TimeSpan.FromHours(1), AutoReplenishment = true });
            });
        });

        return services;
    }

    private static string GetClientIp(HttpContext ctx) =>
        ctx.Request.Headers["CF-Connecting-IP"].FirstOrDefault()
        ?? ctx.Request.Headers["X-Forwarded-For"].FirstOrDefault()
        ?? ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown";
}
```

- [ ] **Step 2: Wire**

```csharp
builder.Services.AddSkillvioRateLimiting(builder.Configuration);
// pipeline:
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.UseMiddleware<JwtIssuerMiddleware>();
app.MapReverseProxy();
```

- [ ] **Step 3: Tests** verify 429 on 6th login, 4th signup, 51st anon GET, etc.

- [ ] **Step 4: Commit**

```bash
git commit -am "feat(s2-t5): tier-based rate limiter (6 policies, plan-aware partitioning)"
git push
```

---

## Task 6 — Production Middleware Pipeline (OWASP Headers)

**Spec ref:** §6 Middleware Pipeline (13-layer), §3 Soru4 (C), §8 S2.T6
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Gateway/Middleware/SecurityHeadersMiddleware.cs`
- Create: `src/Skillvio.Gateway/Middleware/CorrelationIdMiddleware.cs`
- Create: `src/Skillvio.Gateway/Middleware/RequestLoggingMiddleware.cs`
- Modify: `Program.cs` (full 13-layer pipeline)

- [ ] **Step 1: SecurityHeadersMiddleware**

```csharp
public class SecurityHeadersMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext ctx)
    {
        ctx.Response.OnStarting(() =>
        {
            var h = ctx.Response.Headers;
            h["X-Content-Type-Options"] = "nosniff";
            h["X-Frame-Options"] = "DENY";
            h["X-XSS-Protection"] = "1; mode=block";
            h["Referrer-Policy"] = "strict-origin-when-cross-origin";
            h["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()";
            h["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload";
            h["Content-Security-Policy"] = "default-src 'self'; frame-ancestors 'none'";
            return Task.CompletedTask;
        });
        await next(ctx);
    }
}
```

- [ ] **Step 2: CorrelationIdMiddleware**

```csharp
public class CorrelationIdMiddleware(RequestDelegate next)
{
    public const string HeaderName = "X-Correlation-Id";
    public async Task InvokeAsync(HttpContext ctx)
    {
        var id = ctx.Request.Headers[HeaderName].FirstOrDefault() ?? Guid.NewGuid().ToString();
        ctx.Items["CorrelationId"] = id;
        ctx.Response.Headers[HeaderName] = id;
        ctx.Request.Headers[HeaderName] = id;  // forwarded to downstream
        using (Serilog.Context.LogContext.PushProperty("CorrelationId", id))
        using (Activity.Current?.AddTag("correlation_id", id))
        {
            await next(ctx);
        }
    }
}
```

- [ ] **Step 3: RequestLoggingMiddleware** — logs method/path/status/duration, scrubs sensitive paths (login/refresh/change-password) bodies.

- [ ] **Step 4: 13-layer pipeline in Program.cs**

```csharp
app.UseSerilogRequestLogging();
app.UseMiddleware<CorrelationIdMiddleware>();
app.UseMiddleware<SecurityHeadersMiddleware>();
app.UseRouting();
app.UseCors();   // from config; .skillvio.io subdomains
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.UseMiddleware<JwtIssuerMiddleware>();
app.UseMiddleware<RequestLoggingMiddleware>();
app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready");
app.MapPrometheusScrapingEndpoint();
app.MapReverseProxy();
app.Run();
```

- [ ] **Step 5: Tests** — assert each header set on response

- [ ] **Step 6: Commit**

```bash
git commit -am "feat(s2-t6): production middleware pipeline (OWASP headers, correlation, request log)"
git push
```

---

## Task 7 — Polly Resilience (YARP+Polly Hybrid)

**Spec ref:** §3 Soru5 (E), §7 Resilience Configuration, §8 S2.T7
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Gateway/Resilience/PollyConfig.cs`
- Modify: `Program.cs`

- [ ] **Step 1: Add packages**

```bash
dotnet add src/Skillvio.Gateway package Microsoft.Extensions.Http.Resilience
dotnet add src/Skillvio.Gateway package Polly
```

- [ ] **Step 2: Configure Polly v8 resilience strategies for the YARP HttpClient**

```csharp
public static class PollyConfig
{
    public static IServiceCollection AddSkillvioResilience(this IServiceCollection services)
    {
        services.ConfigureHttpClientDefaults(b =>
        {
            b.AddStandardResilienceHandler(opt =>
            {
                opt.Retry.MaxRetryAttempts = 3;
                opt.Retry.UseJitter = true;
                opt.Retry.Delay = TimeSpan.FromMilliseconds(200);
                opt.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(30);
                opt.CircuitBreaker.MinimumThroughput = 10;
                opt.CircuitBreaker.FailureRatio = 0.5;
                opt.AttemptTimeout.Timeout = TimeSpan.FromSeconds(10);
                opt.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
            });
        });

        return services;
    }
}
```

YARP picks up the standard resilience handler automatically when its `HttpClient` is configured this way.

- [ ] **Step 3: Wire**

```csharp
builder.Services.AddSkillvioResilience();
```

- [ ] **Step 4: Commit**

```bash
git commit -am "feat(s2-t7): Polly v8 standard resilience (retry+circuit breaker+timeout) for YARP HttpClient"
git push
```

---

## Task 8 — Health Checks + Observability

**Spec ref:** §8 S2.T8
**Estimated:** 0.5 day
**Files:**
- Create: `src/Skillvio.Gateway/Health/HealthConfig.cs`
- Create: `src/Skillvio.Gateway/Observability/SerilogConfig.cs`
- Create: `src/Skillvio.Gateway/Observability/OpenTelemetryConfig.cs`

- [ ] **Step 1: Health checks**

```csharp
public static class HealthConfig
{
    public static IServiceCollection AddSkillvioHealth(this IServiceCollection services, IConfiguration config)
    {
        services.AddHealthChecks()
            .AddRedis(config.GetConnectionString("Redis")!, name: "redis", tags: ["ready"])
            .AddUrlGroup(new Uri($"{config["DownstreamServices:Identity"]}/health/ready"), name: "identity", tags: ["ready"])
            .AddUrlGroup(new Uri($"{config["DownstreamServices:Learning"]}/health/ready"), name: "learning", tags: ["ready"]);
        return services;
    }
}

// Map:
app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = _ => false });
app.MapHealthChecks("/health/ready", new HealthCheckOptions { Predicate = h => h.Tags.Contains("ready") });
```

- [ ] **Step 2: Serilog → Loki** (same as Identity T10)

- [ ] **Step 3: OpenTelemetry → Tempo + Prometheus**

```csharp
services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("skillvio-gateway"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("Yarp.ReverseProxy")
        .AddOtlpExporter(o => { o.Endpoint = new Uri(config["Otel:Endpoint"]!); }))
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter());
```

- [ ] **Step 4: Commit**

```bash
git commit -am "feat(s2-t8): health checks + Serilog/Loki + OTel/Tempo + Prometheus"
git push
```

---

## Task 9 — Compliance Audit DB

**Spec ref:** §8 S2.T9
**Estimated:** 0.5 day
**Files:**
- Create: `src/Skillvio.Gateway/Compliance/GatewayDbContext.cs`
- Create: `src/Skillvio.Gateway/Compliance/ComplianceAuditEntry.cs`
- Create: `src/Skillvio.Gateway/Compliance/IGatewayComplianceLogger.cs`
- Create: `src/Skillvio.Gateway/Compliance/GatewayComplianceLogger.cs`
- Create: `src/Skillvio.Gateway/Compliance/Migrations/<timestamp>_Init.cs`

- [ ] **Step 1: Add EF packages**

- [ ] **Step 2: Define `gateway_compliance_audit` table** — different from Identity's; this records gateway-level events (rate limit triggered, auth failure, suspicious IP, etc.)

- [ ] **Step 3: Wire into rate limiter `OnRejected` callback** to log persistent rate limit violations

- [ ] **Step 4: Append-only trigger** (same pattern as Identity)

- [ ] **Step 5: Commit**

```bash
git commit -am "feat(s2-t9): gateway compliance audit DB (rate limit triggers, auth failures)"
git push
```

---

## Task 10 — Tests (Integration + Unit)

**Spec ref:** §8 S2.T10
**Estimated:** 1.5 days
**Files:**
- Create: comprehensive test suite under `tests/Skillvio.Gateway.IntegrationTests/`

- [ ] **Step 1: GatewayFixture** — Postgres + Redis + 2 mock downstream HTTP servers (TestServer for identity + learning)

- [ ] **Step 2: Routing tests** — verify each YARP route forwards correctly

- [ ] **Step 3: Cookie cross-service test** — already in T3, expand

- [ ] **Step 4: JWT issuance test** — assert `Authorization: Bearer ...` set on forwarded request

- [ ] **Step 5: Rate limit tests** — for each policy, verify 429 + Retry-After header

- [ ] **Step 6: OWASP header tests** — assert all 7 security headers on responses

- [ ] **Step 7: Authorization tests** — RequireAdmin returns 403 for non-admin

- [ ] **Step 8: Health tests** — /health/live + /health/ready respond correctly

- [ ] **Step 9: Run full suite, target ≥30 green tests**

```bash
dotnet test
```

- [ ] **Step 10: Commit**

```bash
git commit -am "test(s2-t10): comprehensive integration tests (≥30 tests, routing/auth/JWT/rate limit/headers)"
git push
```

---

## Task 11 — Docker + CI

**Spec ref:** §8 S2.T11
**Estimated:** 0.5 day
**Files:**
- Create: `src/Skillvio.Gateway/Dockerfile` (multi-stage Alpine)
- Create: `.dockerignore`
- Modify: `.github/workflows/ci.yml`

Same pattern as Identity Sprint 1 T11. Build image, run smoke locally, push to GHCR via CI.

- [ ] Commit: `feat(s2-t11): multi-stage Alpine Dockerfile + GHA docker-build job`

---

## Task 12 — Frontend Smoke Test

**Spec ref:** §8 S2.T12
**Estimated:** 0.5 day
**Files:**
- Create: `tests/Skillvio.Gateway.SmokeTests/run-smoke.sh`
- Create: `tests/Skillvio.Gateway.SmokeTests/README.md`

End-to-end smoke: start Identity + Gateway + (optionally) Learning. Hit `/api/v1/auth/login` THROUGH the gateway, verify cookie set, verify `/api/v1/users/me` THROUGH gateway returns user, etc.

- [ ] Commit: `test(s2-t12): smoke test through gateway (login → /me → logout via gateway)`

---

## Task 13 — ADR 0011 + Sprint 1 Spec Update

**Spec ref:** §8 S2.T13
**Estimated:** 0.5 day
**Files:**
- Create: `skillvio-planning/adr/0011-url-versioning.md`
- Modify: `skillvio-planning/docs/superpowers/specs/2026-05-04-sprint-1-identity-design.md` (already uses /api/v1/* — verify)

Done from `skillvio-planning` repo, not gateway. Tag Sprint 2 release `v0.1.0-sprint2`.

- [ ] Commit: `docs(s2-t13): ADR 0011 URL versioning + Sprint 2 release tag`

---

## Sprint 2 Wrap-Up

- [ ] Run full suite + verify all integration tests green
- [ ] Tag: `git tag -a v0.1.0-sprint2 -m "Sprint 2 complete — Gateway"`
- [ ] Push tag

---

## Self-Review Checklist (post-write)

- [ ] **Spec coverage:** All 13 spec tasks (S2.T1–T13) covered ✅
- [ ] **Placeholder scan:** No "TBD" ✅
- [ ] **Type consistency:** `IGatewayJwtIssuer.IssueForDownstream`, cookie scheme `"Skillvio.Cookies"`, JWT issuer `"skillvio-gateway"`, audience `"skillvio-services"` consistent ✅
- [ ] **Cross-task integrity:** T4 JWT depends on T3 cookie; T5 rate limit depends on T4 (plan claim from JWT/cookie); T8 observability uses T6 correlation ID ✅
- [ ] **Cross-sprint integrity:** Cookie scheme name + DataProtection key prefix MUST match Identity T5 exactly ✅
