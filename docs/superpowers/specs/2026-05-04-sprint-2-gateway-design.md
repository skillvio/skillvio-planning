# Sprint 2 — API Gateway Design Spec

- **Status:** Draft (awaiting user review)
- **Date:** 2026-05-04
- **Author:** Brainstorming session (Skillvio team + Claude)
- **Sprint:** 2 / 8
- **Estimated effort:** 2 weeks
- **Repo:** [skillvio/skillvio-gateway](https://github.com/skillvio/skillvio-gateway)
- **Related ADRs:** 0002 (microservices), 0003 (.NET 10 + EF Core 10), 0006 (cookie session), 0009 (YARP), 0011 (URL versioning — to be written)
- **Depends on:** Sprint 1 (Identity Service tamamlanmış olmalı)

---

## 1. Amaç

Skillvio'nun **tek entry point'i** — `api.skillvio.io` — burayı kuruyoruz. Gateway tüm
HTTP isteklerini karşılayacak, doğrulayacak, oranlayacak (rate limit), ve doğru downstream
servise yönlendirecek.

Sprint 2 sonunda:
- Frontend `/api/v1/auth/login` → Gateway → Identity service → cookie döner
- Mevcut Node.js backend tamamen kapatılır
- Gateway production-grade resilience ile çalışır
- Downstream servis çökse bile gateway sağlıklı kalır

## 2. Kapsam

### Bu sprintte var

- YARP tabanlı reverse proxy (.NET 10 LTS)
- Cookie → JWT hibrit auth (gateway issuer, downstream verifier)
- Tier-based rate limiting (Free/Pro/Team/Enterprise/Anonim)
- 13-katmanlı production middleware pipeline (OWASP security headers dahil)
- YARP health check + Polly retry/CB resilience
- URL versioning (`/api/v1/...`)
- OpenTelemetry (Sprint 1 ile aynı stack)
- Multi-stage Docker build + GitHub Actions CI
- Frontend smoke test: tüm mevcut endpoint'ler gateway üzerinden çalışır

### Bu sprintte yok

- Cloudflare WAF aktivasyonu (Sprint 7 — Production)
- Admin panel routing (Sprint 6.5)
- v2 paralel route'lar (gerek olunca yazılır)
- gRPC routing (REST yeterli, gRPC gelecekte)
- WebSocket routing (lab service real-time için Sprint 4'te)
- Bot detection (Cloudflare Pro plan, ileride)

---

## 3. Brainstorming Karar Özeti

### Soru 1 — Gateway Teknolojisi: **A (YARP)**

ADR 0009 doğrulandı.
- Microsoft tarafından, .NET 10 native
- C# middleware pipeline ile custom logic (auth, rate limit, audit)
- Hot reload config (`IOptionsMonitor<ReverseProxyOptions>`)
- Identity service ile aynı runtime → debug + observability tek pattern
- OpenTelemetry .NET native — diğer servislerle aynı OTel collector

### Soru 2 — Auth: Gateway vs Service Sınırı: **D (Cookie → JWT hibrit)**

- **Gateway:** Cookie session validate (Redis lookup) → kısa-süreli JWT üret (5 dk TTL, HS256)
- **Cookie strip** edilir; downstream'e geçmez
- **JWT** `Authorization: Bearer` header ile downstream'e
- **Servisler stateless** — JWT signature doğrular, claims kullanır, DB hit yok
- **Key rotation:** `GATEWAY_JWT_SIGNING_KEY` + `_NEXT` (24 saat dual validation overlap)
- **Bypass route'lar:** login, signup, OAuth callback, webhooks, health (cookie set öncesi)
- **Network izolasyonu:** Servisler gateway dışından gelen istekleri reddeder (Docker network / mTLS prod / K8s NetworkPolicy)

### Stack Versiyonu: **.NET 10 LTS** (ADR 0003 revize)

- .NET 10 LTS (2028-11'e kadar)
- EF Core 10
- ASP.NET Core 10 (built-in OpenAPI, HybridCache, RateLimiter)
- C# 14

### Soru 3 — Rate Limiting: **C+E hibrit**

**Edge layer (Cloudflare WAF — Sprint 7'de aktif):**
- DDoS koruması (otomatik, ücretsiz)
- Bot detection (Pro $20/ay, ileride)
- 1000+ req/dk per-IP → challenge

**Application layer (Gateway — Sprint 2):**
- ASP.NET Core 10 native `RateLimiter` + Redis cluster backend
- Tier-based partition (plan'a göre dinamik limit)
- 5 ayrı policy: GeneralApi, AiQuota, LabSubmit, AuthLogin, AuthSignup

**Limit Tablosu:**

| Kategori | Free | Pro | Team | Enterprise | Anonim |
|----------|------|-----|------|------------|--------|
| Genel API | 200/dk | 500/dk | 1000/dk | unlimited | 50/dk per-IP |
| AI explain/tutor | 10/saat | 200/saat | 500/saat | 2000/saat | 5/saat preview |
| Lab submit | 30/dk | 100/dk | 200/dk | unlimited | — (auth) |
| Auth login | 5/15dk per-username (tüm tier) | | | | |
| Auth signup | 3/saat per-IP (anonim) | | | | |
| Password reset | 5/saat per-username | | | | |

429 response: `Retry-After` header + `upgradeUrl` (Free → Pro yönlendirme).

### Soru 4 — Middleware Pipeline: **C (Production-grade + OWASP headers)**

13 katman, OWASP best practice + security headers + audit. Detay aşağıda Section 6.

### Soru 5 — Resilience: **E (YARP native + Polly)**

**YARP layer (cluster-level):**
- Active health check (`/health/ready` 15 sn'de bir, 5 sn timeout, ConsecutiveFailures policy)
- Passive health check (TransportFailureRate, 60 sn reactivation)
- Per-cluster activity timeout (identity 30sn, learning 30sn, lab 30sn, ai 90sn)

**Polly layer (request-level):**
- Retry: 3 deneme, exponential backoff + jitter, sadece 5xx + transient
- Circuit breaker: %50 fail rate / 30 sn → 60 sn open
- Timeout per-attempt: 5 sn (AI: 30 sn override)
- Total timeout: 15 sn (AI: 90 sn)

### Soru 6 — API Versioning: **B (URL versioning)**

- `/api/v1/...` tüm business endpoint'leri
- Gateway transform: v1 prefix downstream'e geçmez, servisler version-agnostic
- Versiyonsuz: `/health/*`, `/webhooks/stripe`, `/webhooks/resend`
- v2 geçiş protokolü: paralel route + Sunset header + 6 ay deprecation
- **ADR 0011** yazılacak

---

## 4. Mimari

### 4.1 Yüksek Seviye Çizim

```
                    ┌──────────────────────────────────────┐
                    │  Frontend Apps (Sprint 6/6.5)        │
                    │  www / app / admin .skillvio.io      │
                    └──────────────────┬───────────────────┘
                                       │ HTTPS, Cookie
                                       ▼
                    ┌──────────────────────────────────────┐
                    │  Cloudflare (Sprint 7'de WAF)        │
                    │  • DDoS koruması                     │
                    │  • Edge rate limit (1000+/dk)        │
                    │  • SSL termination                   │
                    └──────────────────┬───────────────────┘
                                       │
                                       ▼ api.skillvio.io
   ┌──────────────────────────────────────────────────────────┐
   │              skillvio-gateway (YARP, .NET 10)            │
   │                                                          │
   │  Middleware Pipeline (13 katman):                        │
   │  ────────────────────────────────                        │
   │   1. Exception handler                                   │
   │   2. Request ID (W3C trace_id)                          │
   │   3. Serilog request logging                            │
   │   4. Security headers (OWASP)                           │
   │   5. Forwarded headers (Cloudflare X-Forwarded-For)     │
   │   6. CORS (www/app/admin.skillvio.io)                   │
   │   7. Health check bypass                                 │
   │   8. Authentication (cookie → Redis session validate)   │
   │   9. JWT Issuer (cookie → 5dk JWT for downstream)       │
   │  10. Authorization                                       │
   │  11. Rate Limiter (tier-based + Redis backend)          │
   │  12. Compliance Audit (kritik aksiyonlar)               │
   │  13. YARP Routing + Transforms                          │
   └────┬─────────┬─────────┬─────────┬─────────┬────────────┘
        │         │         │         │         │
        │ Authorization: Bearer <JWT>           │
        ▼         ▼         ▼         ▼         ▼
   ┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐
   │identity││learning││  lab   ││   ai   ││ notif  │
   │service ││service ││service ││service ││service │
   └────────┘└────────┘└────────┘└────────┘└────────┘

   Resilience:
   ── YARP active/passive health check (cluster-level)
   ── Polly retry + circuit breaker + timeout (request-level)
```

### 4.2 Cookie → JWT Akışı

```
1. Browser
   GET /api/v1/learning/lessons/abc
   Cookie: skillvio.sid=<session-id>
        │
        ▼
2. Gateway middleware [8: Authentication]:
   - Redis lookup: skillvio.sid → User { id, username, role, plan, sessionId }
   - HttpContext.User = ClaimsPrincipal(user)
        │
        ▼
3. Gateway middleware [9: JWT Issuer]:
   - JWT generate (HS256, GATEWAY_JWT_SIGNING_KEY):
     {
       sub: <userId>,
       username: <username>,
       role: <role>,
       plan: <plan>,
       sid: <sessionId>,
       iss: "skillvio-gateway",
       aud: "skillvio-services",
       iat: <now>,
       exp: <now + 5min>
     }
   - HttpContext.Request.Headers["Authorization"] = "Bearer <jwt>"
   - HttpContext.Request.Headers.Remove("Cookie")  // strip
        │
        ▼
4. Gateway middleware [10: Authorization]:
   - [Authorize] / [Authorize(Roles="admin")] check (HttpContext.User üzerinden)
        │
        ▼
5. Gateway middleware [11: RateLimit]:
   - Plan'a göre limit kontrolü
        │
        ▼
6. Gateway middleware [13: YARP]:
   - Route match: /api/v1/learning/{**rest} → cluster "learning"
   - Transform: Path = /api/learning/lessons/abc
   - Forward to learning:4002/api/learning/lessons/abc
        │
        ▼
7. Learning Service:
   - Authentication middleware: JwtBearer scheme
   - JWT signature verify (GATEWAY_JWT_SIGNING_KEY)
   - HttpContext.User = ClaimsPrincipal(jwtClaims)
   - [Authorize] çalışır
   - Business logic
        │
        ▼
8. Response → Gateway → Browser
```

### 4.3 JWT Schema

```json
{
  "sub": "3f2a7c8d-4b1e-...",
  "username": "gokhan",
  "role": "user",
  "plan": "free",
  "sid": "ses_a1b2c3...",
  "iss": "skillvio-gateway",
  "aud": "skillvio-services",
  "iat": 1714780800,
  "exp": 1714781100,
  "jti": "<random-uuid>"
}
```

**Algorithm:** HS256 (shared secret — internal mTLS olmadığı sürece)
**Production'da RS256'ya geçiş:** Sprint 7+ (Identity service public key endpoint, servisler doğrular)

### 4.4 YARP Route Map

| Route | Path | Cluster | Transform |
|-------|------|---------|-----------|
| auth-v1 | `/api/v1/auth/{**rest}` | identity | Path → `/api/auth/{**rest}` |
| users-v1 | `/api/v1/users/{**rest}` | identity | Path → `/api/users/{**rest}` |
| certs-v1 | `/api/v1/certifications/{**rest}` | identity | Path → `/api/certifications/{**rest}` |
| onboarding-v1 | `/api/v1/onboarding/{**rest}` | identity | Path → `/api/onboarding/{**rest}` |
| learning-v1 | `/api/v1/learning/{**rest}` | learning | Path → `/api/learning/{**rest}` |
| roadmaps-v1 | `/api/v1/roadmaps/{**rest}` | learning | Path → `/api/roadmaps/{**rest}` |
| exams-v1 | `/api/v1/exams/{**rest}` | learning | Path → `/api/exams/{**rest}` |
| attempts-v1 | `/api/v1/attempts/{**rest}` | learning | Path → `/api/attempts/{**rest}` |
| labs-v1 | `/api/v1/labs/{**rest}` | lab | Path → `/api/labs/{**rest}` |
| ai-v1 | `/api/v1/ai/{**rest}` | ai | Path → `/api/ai/{**rest}` |
| admin-v1 | `/api/v1/admin/{**rest}` | identity (or per-service) | Path → `/api/admin/{**rest}` |
| webhooks (no version) | `/webhooks/{provider}` | identity (Stripe/Resend) | direct |
| health (no version) | `/health/*` | self (gateway) | direct |

---

## 5. YARP Configuration

### 5.1 appsettings.json

```json
{
  "ReverseProxy": {
    "Routes": {
      "auth-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/auth/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/auth/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "users-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/users/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/users/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "certs-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/certifications/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/certifications/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "onboarding-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/onboarding/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/onboarding/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "admin-v1": {
        "ClusterId": "identity",
        "Match": { "Path": "/api/v1/admin/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/admin/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ],
        "AuthorizationPolicy": "AdminOnly"
      },
      "learning-v1": {
        "ClusterId": "learning",
        "Match": { "Path": "/api/v1/learning/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/learning/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "roadmaps-v1": {
        "ClusterId": "learning",
        "Match": { "Path": "/api/v1/roadmaps/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/roadmaps/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "exams-v1": {
        "ClusterId": "learning",
        "Match": { "Path": "/api/v1/exams/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/exams/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "attempts-v1": {
        "ClusterId": "learning",
        "Match": { "Path": "/api/v1/attempts/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/attempts/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "labs-v1": {
        "ClusterId": "lab",
        "Match": { "Path": "/api/v1/labs/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/labs/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "ai-v1": {
        "ClusterId": "ai",
        "Match": { "Path": "/api/v1/ai/{**rest}" },
        "Transforms": [
          { "PathPattern": "/api/ai/{**rest}" },
          { "RequestHeaderRemove": "Cookie" }
        ]
      },
      "webhook-stripe": {
        "ClusterId": "identity",
        "Match": { "Path": "/webhooks/stripe" },
        "AuthorizationPolicy": "Anonymous"
      },
      "webhook-resend": {
        "ClusterId": "identity",
        "Match": { "Path": "/webhooks/resend" },
        "AuthorizationPolicy": "Anonymous"
      }
    },
    "Clusters": {
      "identity": {
        "Destinations": {
          "i1": { "Address": "http://identity:4001/" }
        },
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:15",
            "Timeout": "00:00:05",
            "Policy": "ConsecutiveFailures",
            "Path": "/health/ready"
          },
          "Passive": {
            "Enabled": true,
            "Policy": "TransportFailureRate",
            "ReactivationPeriod": "00:01:00"
          }
        },
        "HttpRequest": {
          "ActivityTimeout": "00:00:30"
        }
      },
      "learning": {
        "Destinations": { "l1": { "Address": "http://learning:4002/" } },
        "HealthCheck": { /* aynı */ },
        "HttpRequest": { "ActivityTimeout": "00:00:30" }
      },
      "lab": {
        "Destinations": { "lb1": { "Address": "http://lab:4003/" } },
        "HealthCheck": { /* aynı */ },
        "HttpRequest": { "ActivityTimeout": "00:00:30" }
      },
      "ai": {
        "Destinations": { "ai1": { "Address": "http://ai:4004/" } },
        "HealthCheck": { /* aynı */ },
        "HttpRequest": { "ActivityTimeout": "00:01:30" }
      }
    }
  }
}
```

---

## 6. Middleware Pipeline (Detaylı)

### 6.1 Program.cs Tam Yapı

```csharp
using Microsoft.AspNetCore.HttpOverrides;
using Microsoft.AspNetCore.RateLimiting;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using Serilog;
using Skillvio.Gateway;
using Skillvio.Gateway.Middleware;
using StackExchange.Redis;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// ─── Logging (Serilog → OTel → Loki) ───
builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("service.name", "skillvio.gateway")
    .Enrich.WithProperty("service.version", "1.0.0")
    .WriteTo.OpenTelemetry(opts =>
    {
        opts.Endpoint = ctx.Configuration["Otel:Endpoint"]!;
        opts.Protocol = OtlpProtocol.Grpc;
    }));

// ─── Redis (DataProtection + RateLimit + session validation) ───
var redis = ConnectionMultiplexer.Connect(builder.Configuration["Redis:ConnectionString"]!);
builder.Services.AddSingleton<IConnectionMultiplexer>(redis);
builder.Services.AddDataProtection().PersistKeysToStackExchangeRedis(redis, "DataProtection-Keys");

// ─── Authentication: Cookie scheme ───
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(opts =>
    {
        opts.Cookie.Name = "skillvio.sid";
        opts.Cookie.HttpOnly = true;
        opts.Cookie.SameSite = SameSiteMode.Lax;
        opts.Cookie.SecurePolicy = builder.Environment.IsProduction()
            ? CookieSecurePolicy.Always
            : CookieSecurePolicy.SameAsRequest;
        opts.Cookie.Domain = builder.Configuration["Cookie:Domain"];
        opts.ExpireTimeSpan = TimeSpan.FromMinutes(15);
        opts.SlidingExpiration = true;
        opts.SessionStore = new RedisSessionStore(redis);   // custom — Identity service ile aynı session'a bakar
        opts.Events.OnValidatePrincipal = async ctx =>
        {
            // Redis'te session var mı? Yoksa cookie reject et
            var sid = ctx.Principal!.FindFirst("sid")?.Value;
            if (sid is null || !await redis.GetDatabase().KeyExistsAsync($"session:{sid}"))
                ctx.RejectPrincipal();
        };
    });

// ─── Authorization Policies ───
builder.Services.AddAuthorization(opts =>
{
    opts.AddPolicy("AdminOnly", p => p.RequireRole("admin"));
    opts.AddPolicy("ModeratorOrAdmin", p => p.RequireRole("admin", "moderator"));
    opts.AddPolicy("ContentAuthor", p => p.RequireRole("admin", "content_author"));
    opts.AddPolicy("Authenticated", p => p.RequireAuthenticatedUser());
});

// ─── Rate Limiter (tier-based) ───
builder.Services.AddRateLimiter(options =>
{
    // Genel API
    options.AddPolicy("GeneralApi", ctx =>
    {
        var userId = ctx.User.FindFirst("sub")?.Value;
        var plan = ctx.User.FindFirst("plan")?.Value ?? "anon";
        var (limit, window) = plan switch
        {
            "enterprise" => (int.MaxValue, TimeSpan.FromMinutes(1)),
            "team"       => (1000, TimeSpan.FromMinutes(1)),
            "pro"        => (500, TimeSpan.FromMinutes(1)),
            "free"       => (200, TimeSpan.FromMinutes(1)),
            _            => (50, TimeSpan.FromMinutes(1)),  // anon
        };
        var key = userId ?? GetClientIp(ctx);
        return RateLimitPartition.GetFixedWindowLimiter(key, _ =>
            new FixedWindowRateLimiterOptions { PermitLimit = limit, Window = window });
    });

    // AI quota
    options.AddPolicy("AiQuota", ctx =>
    {
        var plan = ctx.User.FindFirst("plan")?.Value ?? "anon";
        var (limit, window) = plan switch
        {
            "enterprise" => (2000, TimeSpan.FromHours(1)),
            "team"       => (500, TimeSpan.FromHours(1)),
            "pro"        => (200, TimeSpan.FromHours(1)),
            "free"       => (10, TimeSpan.FromHours(1)),
            _            => (5, TimeSpan.FromHours(1)),
        };
        var key = ctx.User.FindFirst("sub")?.Value ?? GetClientIp(ctx);
        return RateLimitPartition.GetFixedWindowLimiter(key, _ =>
            new FixedWindowRateLimiterOptions { PermitLimit = limit, Window = window });
    });

    // Lab submit
    options.AddPolicy("LabSubmit", ctx =>
    {
        var plan = ctx.User.FindFirst("plan")?.Value ?? "anon";
        var (limit, window) = plan switch
        {
            "enterprise" => (int.MaxValue, TimeSpan.FromMinutes(1)),
            "team"       => (200, TimeSpan.FromMinutes(1)),
            "pro"        => (100, TimeSpan.FromMinutes(1)),
            "free"       => (30, TimeSpan.FromMinutes(1)),
            _            => (0, TimeSpan.FromMinutes(1)),  // auth gerek
        };
        var key = ctx.User.FindFirst("sub")?.Value ?? "anon";
        return RateLimitPartition.GetFixedWindowLimiter(key, _ =>
            new FixedWindowRateLimiterOptions { PermitLimit = limit, Window = window });
    });

    // Auth login (per-username + per-IP combine)
    options.AddPolicy("AuthLogin", ctx =>
    {
        // 5 deneme / 15 dk / username veya IP
        var key = ctx.Request.HasJsonContentType()
            ? ExtractUsernameFromBody(ctx) ?? GetClientIp(ctx)
            : GetClientIp(ctx);
        return RateLimitPartition.GetFixedWindowLimiter(key, _ =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 5,
                Window = TimeSpan.FromMinutes(15)
            });
    });

    // Auth signup (per-IP)
    options.AddPolicy("AuthSignup", ctx =>
        RateLimitPartition.GetFixedWindowLimiter(GetClientIp(ctx), _ =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 3,
                Window = TimeSpan.FromHours(1)
            }));

    // Reject handler
    options.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.StatusCode = 429;
        ctx.HttpContext.Response.Headers["Retry-After"] = "60";
        await ctx.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "rate_limit_exceeded",
            message = "İstek limitine ulaştın. Pro plan ile sınırsız erişim.",
            retryAfter = 60,
            upgradeUrl = "https://www.skillvio.io/pricing"
        }, cancellationToken: ct);
    };
});

// ─── OpenTelemetry (logs/metrics/traces) ───
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("skillvio.gateway", serviceVersion: "1.0.0"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("Yarp.ReverseProxy")
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddOtlpExporter());

// ─── Compliance Audit (Postgres) ───
builder.Services.AddDbContext<GatewayAuditDbContext>(o =>
    o.UseNpgsql(builder.Configuration["GatewayAudit:ConnectionString"])
     .UseSnakeCaseNamingConvention());
builder.Services.AddScoped<IComplianceAuditLogger, PostgresComplianceAuditLogger>();

// ─── HttpClient + Polly Resilience ───
builder.Services.ConfigureHttpClientDefaults(http =>
{
    http.AddResilienceHandler("default", pipeline =>
    {
        pipeline
            .AddRetry(new HttpRetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                Delay = TimeSpan.FromMilliseconds(100),
                BackoffType = DelayBackoffType.Exponential,
                UseJitter = true,
                ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
                    .Handle<HttpRequestException>()
                    .Handle<TimeoutRejectedException>()
                    .HandleResult(r => (int)r.StatusCode >= 500
                        && r.StatusCode != HttpStatusCode.NotImplemented)
            })
            .AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
            {
                FailureRatio = 0.5,
                SamplingDuration = TimeSpan.FromSeconds(30),
                MinimumThroughput = 10,
                BreakDuration = TimeSpan.FromSeconds(60),
            })
            .AddTimeout(TimeSpan.FromSeconds(5));
    });
});

// ─── YARP ───
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(builderCtx =>
    {
        builderCtx.AddRequestTransform(async ctx =>
        {
            // Cookie strip — JWT issuer middleware zaten Authorization ekledi
            ctx.ProxyRequest.Headers.Remove("Cookie");
            // X-Real-IP
            var realIp = ctx.HttpContext.Connection.RemoteIpAddress?.ToString();
            if (realIp != null)
                ctx.ProxyRequest.Headers.TryAddWithoutValidation("X-Real-IP", realIp);
            // Trace context (W3C)
            var requestId = ctx.HttpContext.TraceIdentifier;
            ctx.ProxyRequest.Headers.TryAddWithoutValidation("X-Request-Id", requestId);
        });
    });

// ─── Custom Services ───
builder.Services.AddSingleton<IGatewayJwtIssuer, GatewayJwtIssuer>();
builder.Services.AddSingleton<IRequestIdGenerator, RequestIdGenerator>();

// ─── Health Checks ───
builder.Services.AddHealthChecks()
    .AddRedis(builder.Configuration["Redis:ConnectionString"]!, "redis")
    .AddNpgSql(builder.Configuration["GatewayAudit:ConnectionString"]!, "postgres-audit");

// ─── Forwarded Headers (Cloudflare) ───
builder.Services.Configure<ForwardedHeadersOptions>(opts =>
{
    opts.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    // Cloudflare CIDR'lar — production'da env'den
    foreach (var cidr in builder.Configuration.GetSection("Cloudflare:Networks").Get<string[]>() ?? [])
        opts.KnownNetworks.Add(IPNetwork.Parse(cidr));
});

// ─── CORS ───
builder.Services.AddCors(opts =>
{
    opts.AddDefaultPolicy(p => p
        .WithOrigins(builder.Configuration.GetSection("Cors:Origins").Get<string[]>()!)
        .AllowCredentials()
        .AllowAnyHeader()
        .WithMethods("GET", "POST", "PATCH", "PUT", "DELETE", "OPTIONS"));
});

var app = builder.Build();

// ════════════════════════════════════════════════════════════════
//   MIDDLEWARE PIPELINE (sıra önemli)
// ════════════════════════════════════════════════════════════════

// 1. Exception handler — uncaught error → ProblemDetails
app.UseExceptionHandler(handler =>
{
    handler.Run(async ctx =>
    {
        ctx.Response.StatusCode = 500;
        ctx.Response.ContentType = "application/problem+json";
        var feature = ctx.Features.Get<IExceptionHandlerFeature>();
        Log.Error(feature?.Error, "Unhandled exception");
        await ctx.Response.WriteAsJsonAsync(new
        {
            type = "https://skillvio.io/errors/internal-server-error",
            title = "Internal Server Error",
            status = 500,
            traceId = ctx.TraceIdentifier
        });
    });
});

// 2. Request ID
app.UseRequestId();

// 3. Request logging
app.UseSerilogRequestLogging(opts =>
{
    opts.MessageTemplate = "{RequestMethod} {RequestPath} → {StatusCode} in {Elapsed:0}ms";
    opts.EnrichDiagnosticContext = (diag, ctx) =>
    {
        diag.Set("UserId", ctx.User.FindFirst("sub")?.Value);
        diag.Set("Plan", ctx.User.FindFirst("plan")?.Value);
        diag.Set("RequestId", ctx.TraceIdentifier);
        diag.Set("ClientIp", ctx.Connection.RemoteIpAddress?.ToString());
    };
});

// 4. Security headers (OWASP)
app.UseSecurityHeaders(app.Environment);

// 5. Forwarded headers
app.UseForwardedHeaders();

// 6. CORS
app.UseCors();

// 7. Health check (auth/rate limit bypass)
app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = _ => false });
app.MapHealthChecks("/health/ready");

// 8. Authentication (cookie validate)
app.UseAuthentication();

// 9. JWT Issuer (cookie → JWT for downstream)
app.UseGatewayJwtIssuer();

// 10. Authorization
app.UseAuthorization();

// 11. Rate Limiter
app.UseRateLimiter();

// 12. Compliance Audit (selective endpoints)
app.UseComplianceAudit();

// 13. YARP
app.MapReverseProxy();

app.Run();
```

### 6.2 Custom Middleware: Security Headers

```csharp
public static class SecurityHeadersExtensions
{
    public static IApplicationBuilder UseSecurityHeaders(
        this IApplicationBuilder app, IWebHostEnvironment env)
    {
        return app.Use(async (ctx, next) =>
        {
            ctx.Response.Headers["X-Content-Type-Options"] = "nosniff";
            ctx.Response.Headers["X-Frame-Options"] = "DENY";
            ctx.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
            ctx.Response.Headers["Permissions-Policy"] = "geolocation=(), microphone=(), camera=()";

            if (env.IsProduction())
            {
                ctx.Response.Headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload";
                ctx.Response.Headers["Content-Security-Policy"] = "default-src 'none'; frame-ancestors 'none'";
            }

            await next();
        });
    }
}
```

### 6.3 Custom Middleware: JWT Issuer

```csharp
public class GatewayJwtIssuerMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IGatewayJwtIssuer _issuer;

    public async Task InvokeAsync(HttpContext ctx)
    {
        // Bypass: anonim endpoint'ler
        if (ctx.User.Identity?.IsAuthenticated != true)
        {
            await _next(ctx);
            return;
        }

        // Cookie zaten validate edildi (8. middleware) — User context dolu
        var userId = ctx.User.FindFirst(ClaimTypes.NameIdentifier)!.Value;
        var username = ctx.User.FindFirst("username")!.Value;
        var role = ctx.User.FindFirst(ClaimTypes.Role)!.Value;
        var plan = ctx.User.FindFirst("plan")!.Value;
        var sid = ctx.User.FindFirst("sid")!.Value;

        var jwt = _issuer.GenerateForDownstream(userId, username, role, plan, sid);
        ctx.Request.Headers["Authorization"] = $"Bearer {jwt}";

        await _next(ctx);
    }
}

public class GatewayJwtIssuer : IGatewayJwtIssuer
{
    private readonly string _signingKey;
    public string GenerateForDownstream(string userId, string username, string role, string plan, string sid)
    {
        var handler = new JsonWebTokenHandler();
        var descriptor = new SecurityTokenDescriptor
        {
            Issuer = "skillvio-gateway",
            Audience = "skillvio-services",
            Subject = new ClaimsIdentity(new[]
            {
                new Claim(JwtRegisteredClaimNames.Sub, userId),
                new Claim("username", username),
                new Claim("role", role),
                new Claim("plan", plan),
                new Claim("sid", sid),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
            }),
            Expires = DateTime.UtcNow.AddMinutes(5),
            SigningCredentials = new SigningCredentials(
                new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_signingKey)),
                SecurityAlgorithms.HmacSha256)
        };
        return handler.CreateToken(descriptor);
    }
}
```

---

## 7. Resilience Configuration

### 7.1 Per-Cluster Timeout Matrix

| Cluster | YARP ActivityTimeout | Polly Per-Attempt | Polly Total |
|---------|----------------------|-------------------|-------------|
| identity | 30 sn | 5 sn | 15 sn |
| learning | 30 sn | 5 sn | 15 sn |
| lab | 30 sn | 10 sn | 30 sn |
| ai | 90 sn | 30 sn | 90 sn |
| notification | 30 sn | 5 sn | 15 sn |

### 7.2 Circuit Breaker Davranışı

```
Closed (normal):
  - Tüm istekler downstream'e gider
  - Fail rate izlenir

Open (downstream çökmüş):
  - Trigger: 30 sn pencerede %50+ fail (min 10 throughput)
  - Duration: 60 sn
  - Davranış: TÜM istekler anında 503 (downstream'e hiç çağrı yok)

Half-Open (test):
  - 60 sn sonra otomatik
  - 1 deneme isteği geçer
  - Başarılıysa → Closed
  - Başarısızsa → 60 sn daha Open
```

### 7.3 Hata Cevap Şablonları

```json
// 503 — Downstream çökmüş (CB Open)
{
  "type": "https://skillvio.io/errors/service-unavailable",
  "title": "Service Unavailable",
  "status": 503,
  "service": "learning",
  "message": "Geçici bir sorun var, birazdan tekrar deneyin.",
  "retryAfter": 30,
  "traceId": "00-..."
}

// 504 — Timeout
{
  "type": "https://skillvio.io/errors/gateway-timeout",
  "title": "Gateway Timeout",
  "status": 504,
  "service": "ai",
  "message": "İstek çok uzun sürdü. Tekrar dene.",
  "traceId": "00-..."
}

// 429 — Rate limit
{
  "type": "https://skillvio.io/errors/rate-limit-exceeded",
  "title": "Too Many Requests",
  "status": 429,
  "message": "İstek limitine ulaştın.",
  "retryAfter": 60,
  "upgradeUrl": "https://www.skillvio.io/pricing"
}
```

---

## 8. Sprint 2 Görev Listesi (Detaylı)

### S2.T1 — Solution + paketler (0.5 gün)
- `dotnet new webapi -n Skillvio.Gateway`
- `dotnet new xunit -n Skillvio.Gateway.Tests`
- NuGet paketler:
  - `Yarp.ReverseProxy`
  - `Microsoft.AspNetCore.RateLimiting`
  - `Microsoft.Extensions.Http.Resilience` (Polly)
  - `OpenTelemetry.*`
  - `Serilog.AspNetCore` + `Serilog.Sinks.OpenTelemetry`
  - `StackExchange.Redis`
  - `Microsoft.AspNetCore.DataProtection.StackExchangeRedis`
  - `Microsoft.AspNetCore.Authentication.Cookies`
  - `Microsoft.IdentityModel.JsonWebTokens`
  - `Npgsql.EntityFrameworkCore.PostgreSQL` (audit için)
  - `EFCore.NamingConventions`

**DoD:** `dotnet build` + `dotnet test` (boş test) yeşil.

### S2.T2 — YARP route config (0.5 gün)
- `appsettings.json`'da 13 route + 5 cluster (Section 5.1)
- `appsettings.Development.json` override (localhost:4001..4005)
- Route validation startup'ta (yanlış config → fail fast)

**DoD:** `dotnet run` ile gateway başlar; mock downstream service'lere proxy eder.

### S2.T3 — Cookie auth + Redis session validation (1 gün)
- `AddAuthentication().AddCookie(...)` config (Section 6.1)
- Custom `ITicketStore` veya `OnValidatePrincipal` event ile Redis session lookup
- Identity service ile **aynı Redis instance** + aynı key pattern (`session:{sid}`)
- Cookie domain config (dev: localhost, prod: `.skillvio.io`)

**DoD:** Identity'de oluşturulmuş session, gateway'de validate edilir; geçersiz session → 401.

### S2.T4 — JWT Issuer middleware (1 gün)
- `IGatewayJwtIssuer` + implementation (Section 6.3)
- HS256 signing, 5 dk TTL
- Tüm authenticated istekler için header inject
- Key rotation desteği (`SigningKey` + `SigningKeyNext`, 24 saat overlap)
- Identity service ile aynı `GATEWAY_JWT_SIGNING_KEY` env

**DoD:** Mock downstream service JWT'yi doğrular; signature/exp/iss/aud kontrolleri yeşil.

### S2.T5 — Rate Limiter (1 gün)
- 5 policy: GeneralApi, AiQuota, LabSubmit, AuthLogin, AuthSignup
- Tier-based partition (plan claim'inden)
- Redis backend (multiple gateway instance için)
- 429 response template

**DoD:** Manual test: free user 11. AI request → 429; pro user 11. → 200.

### S2.T6 — Middleware pipeline (1 gün)
- 13 katmanın hepsi sırasıyla
- Security headers middleware (Section 6.2)
- Compliance audit middleware (selective endpoint'ler için)
- ForwardedHeaders + CORS

**DoD:** Tüm middleware'ler integration test'te tek tek çalıştırılıp doğrulanmış.

### S2.T7 — Polly resilience (1 gün)
- `ConfigureHttpClientDefaults` + `AddResilienceHandler`
- Per-cluster timeout override (AI 30sn)
- Circuit breaker config
- Polly metrics OpenTelemetry'e emit

**DoD:** Toxiproxy ile downstream gecikme/fail simüle; CB state geçişleri test edilmiş.

### S2.T8 — Health checks + observability (0.5 gün)
- `/health/live`, `/health/ready` endpoint'leri
- Redis + Postgres health check
- OpenTelemetry config (Sprint 1 ile aynı)
- Grafana dashboard'a 4 panel ekle: request rate, error rate, p99 latency, CB state

**DoD:** `curl /health/ready` 200 döner; Grafana'da gateway metric'leri görünür.

### S2.T9 — Compliance audit DB (0.5 gün)
- Gateway'in **kendi** `compliance_audit` tablosu (bağımsız DB)
- Yazılan event'ler:
  - `auth.login_attempted` (rate limit lock öncesi)
  - `auth.rate_limit_exceeded`
  - `gateway.cb_opened` (Sentry alert için)
- Identity'nin compliance_audit'i ile **ayrı** (bounded context)

**DoD:** Test rate limit lock → audit row insert; audit query çalışır.

### S2.T10 — Test (1.5 gün)
- xUnit + WebApplicationFactory + Testcontainers
- Mock downstream service (TestServer)
- 20 happy path test
- 15 sad path test (CB open, rate limit, invalid cookie, missing JWT key)

**DoD:** Coverage ≥%70; tüm testler 5 dakikadan kısa.

### S2.T11 — Docker + CI (0.5 gün)
- Multi-stage Dockerfile (Native AOT opsiyonel — değerlendirilecek)
- `.dockerignore`
- GitHub Actions: build + test + image push GHCR
- `docker-compose.yml` skillvio-infra'da gateway service ekle

**DoD:** PR'da CI yeşil; main merge → image GHCR'da.

### S2.T12 — Frontend smoke test (0.5 gün)
- Eski Node.js backend container'ı kapat
- Frontend `apiClient.js`'in URL'lerini güncelle (`/api/...` → `/api/v1/...`)
- Manual smoke: login, signup, profile, me, logout, certs
- Cookie domain `.skillvio.io` test (dev'de localhost)

**DoD:** Tüm mevcut UI akışları gateway üzerinden çalışır.

### S2.T13 — ADR 0011 + Sprint 1 spec update (0.5 gün)
- ADR 0011 yaz: URL Versioning Strategy
- Sprint 1 spec'inde tüm `/api/...` → `/api/v1/...` (sadece doc)

**DoD:** Doküman commit'leri push edilmiş.

**Toplam tahmini: 11 gün × 1 developer = ~2.2 hafta**

---

## 9. Riskler & Hafifletme

| Risk | Olasılık | Etki | Hafifletme |
|------|----------|------|------------|
| Cookie domain dev/prod farkı | Yüksek | Düşük | Env-based config; localhost dev için ayrı, prod `.skillvio.io` |
| JWT key rotation senkron olmazsa | Düşük | Yüksek | 24 saat dual validation overlap; auto rotation Hangfire job (Sprint 7) |
| Redis session store down → tüm logout | Düşük | Yüksek | Redis Sentinel/Cluster prod'da; circuit breaker fallback (degraded mode) |
| Rate limit Redis backend yavaşlığı | Orta | Orta | Local in-memory fallback; Redis cluster |
| YARP health check false positive | Orta | Orta | ConsecutiveFailures ≥3; reactivation period ayarlanabilir |
| CB threshold çok düşük (sürekli açılır) | Orta | Orta | Production tuning Sprint 7; başlangıç defaults conservative |
| AI 90 sn timeout çok uzun (UX kötü) | Yüksek | Düşük | Frontend timeout 60 sn göster; gerçek call 90 sn arka planda; queue+poll pattern Sprint 5 |
| Cloudflare X-Forwarded-For spoofing | Düşük | Yüksek | KnownNetworks Cloudflare CIDR ile sıkı bağlama |
| Cookie ↔ JWT senkron drift (claim'ler farklılaşır) | Düşük | Orta | Single source of truth: cookie validate sırasında User ClaimsPrincipal oluşur, JWT aynısını kullanır |
| Sprint 1 endpoint path'leri Sprint 2 routing'e uymazsa | Orta | Orta | Sprint 1 spec'i bu sprint'te `/api/v1/` prefix ile güncelleniyor |

---

## 10. ENV Variables

```env
# Service
ASPNETCORE_ENVIRONMENT=Development
PORT=5000

# Redis
REDIS__CONNECTIONSTRING=redis:6379,password=...,defaultDatabase=0

# Cookie
COOKIE__DOMAIN=.skillvio.io                  # prod
# COOKIE__DOMAIN=localhost                   # dev

# JWT (gateway → downstream)
JWT__SIGNINGKEY=<32+ char random hex>
JWT__SIGNINGKEYNEXT=                         # rotation için optional
JWT__ISSUER=skillvio-gateway
JWT__AUDIENCE=skillvio-services
JWT__TTLSECONDS=300                          # 5 min

# CORS
CORS__ORIGINS=https://www.skillvio.io,https://app.skillvio.io,https://admin.skillvio.io

# Cloudflare CIDR (X-Forwarded-For trust)
CLOUDFLARE__NETWORKS=173.245.48.0/20,103.21.244.0/22,...

# Postgres (audit)
GATEWAYAUDIT__CONNECTIONSTRING=Host=postgres;Database=gateway_audit;Username=...;Password=...

# OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# YARP Cluster Endpoints
REVERSEPROXY__CLUSTERS__IDENTITY__DESTINATIONS__I1__ADDRESS=http://identity:4001/
REVERSEPROXY__CLUSTERS__LEARNING__DESTINATIONS__L1__ADDRESS=http://learning:4002/
REVERSEPROXY__CLUSTERS__LAB__DESTINATIONS__LB1__ADDRESS=http://lab:4003/
REVERSEPROXY__CLUSTERS__AI__DESTINATIONS__AI1__ADDRESS=http://ai:4004/
REVERSEPROXY__CLUSTERS__NOTIFICATION__DESTINATIONS__N1__ADDRESS=http://notification:4005/
```

---

## 11. Spec Self-Review

### Placeholder scan
- ✅ TBD/TODO yok
- ✅ Tüm route'lar full URL pattern + cluster
- ✅ Code snippet'leri tam (eksik kısım yok)

### Internal consistency
- ✅ JWT schema ile Identity service ClaimsPrincipal uyumlu
- ✅ Rate limit Identity'nin login attempt limit'iyle çakışmıyor (defense in depth)
- ✅ Sprint 1 endpoint path'leri v1 prefix ile uyumlu (T13'te güncellenecek)

### Scope check
- ✅ Sprint 2 odağı net: gateway tamamlandı + frontend bağlandı
- ✅ Cloudflare WAF aktivasyonu Sprint 7'ye atıldı (sadece config preview burada)
- ✅ 2.2 hafta tahmini gerçekçi

### Ambiguity check
- ✅ Cookie → JWT akışı 8 adımda net
- ✅ Per-cluster timeout matrix tablo halinde
- ✅ CB state geçişleri Closed/Open/Half-Open net açıklı

### Bilinçli atılanlar (YAGNI)
- ❌ Native AOT (S2.T11'de değerlendirilecek, şimdilik standart build)
- ❌ gRPC routing (REST yeterli)
- ❌ WebSocket routing (Sprint 4'te lab terminal için)
- ❌ Bot detection (Cloudflare Pro plan, Sprint 7+)

---

## 12. Approval Gate

> **Bu doküman draft. Brainstorming methodology gereği:**
>
> 1. Kullanıcı bu spec'i okur
> 2. Değişiklik isterse — yorum/değişiklik tur'u
> 3. Onaylanırsa → `writing-plans` skill'i devreye girer
>    - Sprint 2 GitHub issue'ları **detaylı acceptance criteria** ile güncellenir
>    - Her görev 2-5 dk'lık alt-task'a bölünür
>
> **Hiçbir kod yazılmayacak — bu spec onaylanmadan önce.**

---

## 13. Karar Geçmişi

| Karar | Seçenek | Tarih | Sebep |
|-------|---------|-------|-------|
| Gateway teknolojisi | A (YARP) | 2026-05-04 | ADR 0009 doğrulandı; .NET native; OTel uyumu |
| Stack versiyonu | .NET 10 LTS | 2026-05-04 | ADR 0003 revize; LTS-to-LTS, EF Core 10 + built-in OpenAPI |
| Auth split | D (Cookie → JWT hibrit) | 2026-05-04 | Stateless service + perf + revoke kontrolü |
| Rate limit | C+E (tier + Cloudflare) | 2026-05-04 | Pro tier değer; edge DDoS koruma |
| Middleware | C (production + OWASP) | 2026-05-04 | Defense in depth + audit hazır |
| Resilience | E (YARP + Polly) | 2026-05-04 | YARP'ın native değerini al + Polly request-level |
| Versioning | B (URL `/api/v1/`) | 2026-05-04 | Industry standard; mobile uyumu; CDN cache friendly |

---

## 14. Referanslar

- [YARP Documentation](https://microsoft.github.io/reverse-proxy/)
- [Polly v8 Resilience Strategies](https://www.pollydocs.org/strategies/index.html)
- [OWASP Secure Headers](https://owasp.org/www-project-secure-headers/)
- [ASP.NET Core 10 Rate Limiting](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [Microsoft.Extensions.Http.Resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [API Versioning Best Practices (Stripe)](https://stripe.com/blog/api-versioning)
- ADR 0003 — .NET 10 + EF Core 10
- ADR 0006 — Cookie Session Auth
- ADR 0009 — YARP Gateway
- ADR 0011 — URL Versioning Strategy (yazılacak)
