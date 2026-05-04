# Sprint 1 — Identity Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `skillvio-identity-service` from scratch on .NET 10 + EF Core 10 + Postgres 16. Authentication, authorization, sessions, plans, certifications, onboarding, audit. Replaces the legacy Node.js backend at `/Users/gokhannihal/exam/backend/`.

**Architecture:** Clean Architecture (Domain/Application/Infrastructure/Api). Cookie-based session via Redis. Refresh token rotation with family invalidation. Outbox pattern via MassTransit. Postgres compliance audit (~10K rows/yr). Loki + OpenTelemetry observability.

**Tech Stack:** .NET 10 LTS, ASP.NET Core 10 Minimal APIs + Identity, EF Core 10 + Npgsql + EFCore.NamingConventions, MassTransit (RabbitMQ + EF Outbox), Hangfire (Postgres), FluentValidation, BCrypt.Net-Next, Otp.NET (TOTP), Resend (email), Serilog → Loki, OpenTelemetry → Tempo, Prometheus.

**Spec:** `docs/superpowers/specs/2026-05-04-sprint-1-identity-design.md`

**Effort:** ~14 work-days, ~13 tasks (T1-T12). 1 dev solo.

**Repo:** `skillvio-identity-service` (skeleton on `main`, remote `git@github.com:skillvio/skillvio-identity-service.git`).

**Dependencies (must already be live):**
- ✅ `skillvio-shared-contracts` v0.1.0 NuGet (29 events, available locally and on GitHub)
- ✅ `skillvio-infra` docker-compose.dev.yml (Postgres + Redis + RabbitMQ + observability stack healthy)

---

## Pre-Flight Checklist

Before T1, verify:

- [ ] `skillvio-shared-contracts` v0.1.0 builds and packs locally (`dotnet pack`)
- [ ] `cd skillvio-infra && docker compose -f docker-compose.dev.yml up -d` shows all services healthy
- [ ] `psql -h localhost -U skillvio -d identity_db -c '\l'` lists `identity_db`
- [ ] `redis-cli ping` returns PONG
- [ ] OAuth credentials obtained (Google + GitHub apps registered) — can be deferred to T6 with placeholder values
- [ ] Resend API key obtained (or use Mailtrap for dev) — needed by T6 (email sending)

---

## Task 1 — Solution Skeleton (.NET 10, 4 Projects + 2 Test Projects)

**Spec ref:** §4.2 Clean Architecture Layout, §11 S1.T1
**Estimated:** 1 day
**Files:**
- Create: `skillvio-identity-service.sln`
- Create: `Directory.Build.props`
- Create: `Directory.Packages.props`
- Create: `global.json` (pin .NET 10.0.203)
- Create: `.editorconfig`
- Create: `.gitignore`
- Create: `src/Skillvio.Identity.Domain/Skillvio.Identity.Domain.csproj`
- Create: `src/Skillvio.Identity.Application/Skillvio.Identity.Application.csproj`
- Create: `src/Skillvio.Identity.Infrastructure/Skillvio.Identity.Infrastructure.csproj`
- Create: `src/Skillvio.Identity.Api/Skillvio.Identity.Api.csproj`
- Create: `tests/Skillvio.Identity.Domain.Tests/Skillvio.Identity.Domain.Tests.csproj`
- Create: `tests/Skillvio.Identity.IntegrationTests/Skillvio.Identity.IntegrationTests.csproj`
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Verify .NET 10 SDK + remote repo**

```bash
cd /Users/gokhannihal/skillvio/skillvio-identity-service
dotnet --version       # must report 10.0.x
git status             # clean tree on main with skeleton
```

- [ ] **Step 2: Write `global.json` (pin SDK)**

```json
{
  "sdk": { "version": "10.0.203", "rollForward": "latestPatch" }
}
```

- [ ] **Step 3: Write `Directory.Build.props`**

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

- [ ] **Step 4: Write `Directory.Packages.props` (central package management)**

Include all packages needed across Sprint 1 tasks:
```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.0" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.0" />
    <PackageVersion Include="EFCore.NamingConventions" Version="10.0.0" />
    <PackageVersion Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="10.0.0" />
    <PackageVersion Include="Microsoft.AspNetCore.Authentication.Google" Version="10.0.0" />
    <PackageVersion Include="AspNet.Security.OAuth.GitHub" Version="9.0.0" />
    <PackageVersion Include="MassTransit" Version="8.3.0" />
    <PackageVersion Include="MassTransit.RabbitMQ" Version="8.3.0" />
    <PackageVersion Include="MassTransit.EntityFrameworkCore" Version="8.3.0" />
    <PackageVersion Include="Hangfire.AspNetCore" Version="1.8.14" />
    <PackageVersion Include="Hangfire.PostgreSql" Version="1.20.10" />
    <PackageVersion Include="FluentValidation.AspNetCore" Version="11.3.0" />
    <PackageVersion Include="BCrypt.Net-Next" Version="4.0.3" />
    <PackageVersion Include="Otp.NET" Version="1.4.0" />
    <PackageVersion Include="QRCoder" Version="1.6.0" />
    <PackageVersion Include="Resend" Version="0.0.4" />
    <PackageVersion Include="StackExchange.Redis" Version="2.8.16" />
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Caching.StackExchangeRedis" Version="10.0.0" />
    <PackageVersion Include="Microsoft.AspNetCore.DataProtection.StackExchangeRedis" Version="10.0.0" />
    <PackageVersion Include="Serilog.AspNetCore" Version="8.0.3" />
    <PackageVersion Include="Serilog.Sinks.Grafana.Loki" Version="8.3.0" />
    <PackageVersion Include="OpenTelemetry.Extensions.Hosting" Version="1.10.0" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.10.1" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.EntityFrameworkCore" Version="1.10.0-beta.1" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.Http" Version="1.10.0" />
    <PackageVersion Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.10.0" />
    <PackageVersion Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" Version="1.10.0-beta.1" />
    <PackageVersion Include="Skillvio.Shared.Contracts" Version="0.1.0" />
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.11.1" />
    <PackageVersion Include="xunit" Version="2.9.2" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageVersion Include="FluentAssertions" Version="6.12.1" />
    <PackageVersion Include="Testcontainers.PostgreSql" Version="3.10.0" />
    <PackageVersion Include="Testcontainers.Redis" Version="3.10.0" />
    <PackageVersion Include="Testcontainers.RabbitMq" Version="3.10.0" />
    <PackageVersion Include="Microsoft.AspNetCore.Mvc.Testing" Version="10.0.0" />
    <PackageVersion Include="NSubstitute" Version="5.1.0" />
  </ItemGroup>
</Project>
```

- [ ] **Step 5: Create solution + Domain project**

```bash
dotnet new sln -n skillvio-identity-service --format sln
mkdir -p src/Skillvio.Identity.Domain
cd src/Skillvio.Identity.Domain
dotnet new classlib -n Skillvio.Identity.Domain -f net10.0 --force
rm Class1.cs
cd ../..
dotnet sln add src/Skillvio.Identity.Domain/Skillvio.Identity.Domain.csproj
```

- [ ] **Step 6: Create Application + Infrastructure + Api projects**

```bash
mkdir -p src/Skillvio.Identity.Application src/Skillvio.Identity.Infrastructure src/Skillvio.Identity.Api
cd src/Skillvio.Identity.Application && dotnet new classlib -f net10.0 --force && rm Class1.cs && cd ../..
cd src/Skillvio.Identity.Infrastructure && dotnet new classlib -f net10.0 --force && rm Class1.cs && cd ../..
cd src/Skillvio.Identity.Api && dotnet new web -f net10.0 --force && cd ../..
dotnet sln add src/Skillvio.Identity.Application/Skillvio.Identity.Application.csproj src/Skillvio.Identity.Infrastructure/Skillvio.Identity.Infrastructure.csproj src/Skillvio.Identity.Api/Skillvio.Identity.Api.csproj
dotnet add src/Skillvio.Identity.Application reference src/Skillvio.Identity.Domain
dotnet add src/Skillvio.Identity.Infrastructure reference src/Skillvio.Identity.Application
dotnet add src/Skillvio.Identity.Api reference src/Skillvio.Identity.Application src/Skillvio.Identity.Infrastructure
```

- [ ] **Step 7: Create test projects**

```bash
mkdir -p tests/Skillvio.Identity.Domain.Tests tests/Skillvio.Identity.IntegrationTests
cd tests/Skillvio.Identity.Domain.Tests && dotnet new xunit -f net10.0 --force && rm UnitTest1.cs && cd ../..
cd tests/Skillvio.Identity.IntegrationTests && dotnet new xunit -f net10.0 --force && rm UnitTest1.cs && cd ../..
dotnet sln add tests/Skillvio.Identity.Domain.Tests/ tests/Skillvio.Identity.IntegrationTests/
dotnet add tests/Skillvio.Identity.Domain.Tests reference src/Skillvio.Identity.Domain
dotnet add tests/Skillvio.Identity.IntegrationTests reference src/Skillvio.Identity.Api src/Skillvio.Identity.Infrastructure
```

- [ ] **Step 8: Add baseline package references in all .csproj files**

Each `*.csproj` lists `<PackageReference Include="..." />` (no Version because central management). Test projects must include xunit, FluentAssertions, Microsoft.NET.Test.Sdk. IntegrationTests must add Testcontainers.PostgreSql, Testcontainers.Redis, Testcontainers.RabbitMq, Microsoft.AspNetCore.Mvc.Testing.

- [ ] **Step 9: Verify clean build**

```bash
dotnet restore && dotnet build
```

Expected: 0 warnings, 0 errors.

- [ ] **Step 10: Add `.editorconfig` and `.gitignore`** (copy from `skillvio-shared-contracts` for consistency)

- [ ] **Step 11: Write minimal CI workflow** (Postgres/Redis/RabbitMQ services, dotnet test)

```yaml
# .github/workflows/ci.yml — see skillvio-learning-service plan T1 for template; adapt service ports
```

- [ ] **Step 12: Commit and push**

```bash
git add -A
git commit -m "chore(s1-t1): scaffold .NET 10 solution with 4 src + 2 test projects"
git push -u origin main
```

---

## Task 2 — Domain Models (Aggregates + Value Objects)

**Spec ref:** §4.2 (Domain layer), §5.1 (table list maps 1:1 to entities)
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Identity.Domain/Users/User.cs` (extends `IdentityUser<Guid>`)
- Create: `src/Skillvio.Identity.Domain/Users/UserProfile.cs`
- Create: `src/Skillvio.Identity.Domain/Users/UserPreferences.cs`
- Create: `src/Skillvio.Identity.Domain/Sessions/Session.cs`
- Create: `src/Skillvio.Identity.Domain/Sessions/RefreshToken.cs`
- Create: `src/Skillvio.Identity.Domain/Sessions/RefreshTokenFamily.cs`
- Create: `src/Skillvio.Identity.Domain/Plans/SubscriptionPlan.cs`
- Create: `src/Skillvio.Identity.Domain/Plans/UserSubscription.cs`
- Create: `src/Skillvio.Identity.Domain/Roles/Role.cs` (extends `IdentityRole<Guid>`)
- Create: `src/Skillvio.Identity.Domain/Roles/UserRole.cs`
- Create: `src/Skillvio.Identity.Domain/Certifications/CertificationApplication.cs`
- Create: `src/Skillvio.Identity.Domain/Certifications/CertificationDocument.cs`
- Create: `src/Skillvio.Identity.Domain/Mfa/TotpSecret.cs`
- Create: `src/Skillvio.Identity.Domain/Mfa/RecoveryCode.cs`
- Create: `src/Skillvio.Identity.Domain/Compliance/ComplianceAudit.cs`
- Create: `src/Skillvio.Identity.Domain/Onboarding/OnboardingState.cs`
- Create: `src/Skillvio.Identity.Domain/Common/ValueObjects/Email.cs`
- Create: `src/Skillvio.Identity.Domain/Common/ValueObjects/Username.cs`
- Create: `src/Skillvio.Identity.Domain/Common/Abstractions/IClock.cs`
- Create: `tests/Skillvio.Identity.Domain.Tests/Users/UserTests.cs`
- Create: `tests/Skillvio.Identity.Domain.Tests/Sessions/RefreshTokenFamilyTests.cs`

- [ ] **Step 1: Add ASP.NET Core Identity + EF Core packages to Domain (for IdentityUser/IdentityRole base types)**

```bash
dotnet add src/Skillvio.Identity.Domain package Microsoft.AspNetCore.Identity.EntityFrameworkCore
```

- [ ] **Step 2: Write `Email` value object with test (TDD red)**

Test:
```csharp
// tests/.../Common/EmailTests.cs
[Theory]
[InlineData("user@example.com", true)]
[InlineData("user+tag@example.co.uk", true)]
[InlineData("invalid", false)]
[InlineData("@example.com", false)]
[InlineData("user@", false)]
public void Create_ValidatesFormat(string input, bool valid)
{
    if (valid) Email.Create(input).Value.Should().Be(input.ToLowerInvariant());
    else FluentActions.Invoking(() => Email.Create(input)).Should().Throw<ArgumentException>();
}
```

Implementation:
```csharp
public sealed record Email
{
    public string Value { get; }
    private Email(string v) { Value = v; }
    public static Email Create(string raw)
    {
        if (string.IsNullOrWhiteSpace(raw) || !raw.Contains('@') || raw.StartsWith('@') || raw.EndsWith('@'))
            throw new ArgumentException("Invalid email", nameof(raw));
        var trimmed = raw.Trim().ToLowerInvariant();
        if (!System.Net.Mail.MailAddress.TryCreate(trimmed, out _))
            throw new ArgumentException("Invalid email", nameof(raw));
        return new Email(trimmed);
    }
    public override string ToString() => Value;
}
```

- [ ] **Step 3: Write `User` aggregate (extends ASP.NET Identity's `IdentityUser<Guid>`)**

```csharp
// src/Skillvio.Identity.Domain/Users/User.cs
using Microsoft.AspNetCore.Identity;

namespace Skillvio.Identity.Domain.Users;

public class User : IdentityUser<Guid>
{
    public string DisplayName { get; set; } = "";
    public string? AvatarUrl { get; set; }
    public string Locale { get; set; } = "en";
    public string Timezone { get; set; } = "UTC";
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset UpdatedAt { get; set; }
    public DateTimeOffset? LastLoginAt { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTimeOffset? SoftDeletedAt { get; set; }
    public DateTimeOffset? HardDeleteScheduledAt { get; set; }
    public string? SoftDeleteReason { get; set; }
    // Navigation
    public UserProfile? Profile { get; set; }
    public UserPreferences? Preferences { get; set; }
    public UserSubscription? Subscription { get; set; }
    public OnboardingState? Onboarding { get; set; }
}
```

- [ ] **Step 4: Test `RefreshTokenFamily.InvalidateAll()` invariant**

```csharp
[Fact]
public void InvalidateAll_RevokesEveryTokenInFamily()
{
    var family = RefreshTokenFamily.Create(Guid.NewGuid(), "device-A");
    family.AddToken("hash1", DateTimeOffset.UtcNow.AddDays(30));
    family.AddToken("hash2", DateTimeOffset.UtcNow.AddDays(30));
    family.InvalidateAll("reuse_detected");
    family.Tokens.Should().AllSatisfy(t => t.RevokedAt.Should().NotBeNull());
    family.IsRevoked.Should().BeTrue();
}
```

- [ ] **Step 5: Implement `RefreshTokenFamily` with rotate + reuse detection**

Logic:
- `Rotate(string oldTokenHash, string newTokenHash)` — revokes old, adds new
- `DetectReuse(string presentedHash)` — if presented hash is already revoked → invalidate family + return true
- `InvalidateAll(string reason)` — revokes all tokens + sets `IsRevoked = true`

- [ ] **Step 6: Implement remaining aggregates (one .cs per entity)**

Each follows the spec's column list verbatim. Use:
- `Guid Id` PK on all
- `DateTimeOffset` for all timestamps
- Nullable reference types enabled
- Navigation properties bidirectional where needed
- Owned types for composite VOs (Email, Username — but only mapped via EF in T3)

Specifically write:
- `UserProfile`, `UserPreferences`, `OnboardingState` (all 1:1 with User)
- `Session`, `RefreshToken` (1:N from User → Sessions)
- `SubscriptionPlan` (lookup), `UserSubscription` (1:1 with User)
- `Role`, `UserRole` (M:N junction)
- `CertificationApplication`, `CertificationDocument` (1:N)
- `TotpSecret`, `RecoveryCode` (1:1 + 1:N with User)
- `ComplianceAudit` (append-only log; see §5.1 for schema)

- [ ] **Step 7: Write `IClock` abstraction for testability**

```csharp
public interface IClock { DateTimeOffset UtcNow { get; } }
public class SystemClock : IClock { public DateTimeOffset UtcNow => DateTimeOffset.UtcNow; }
```

- [ ] **Step 8: Run Domain.Tests**

```bash
dotnet test tests/Skillvio.Identity.Domain.Tests
```

Expected: All green. Domain coverage ≥ 70%.

- [ ] **Step 9: Commit**

```bash
git commit -am "feat(s1-t2): domain aggregates (User, Session, Plan, Role, Cert, MFA, Compliance)"
git push
```

---

## Task 3 — EF Core + Initial Migration (Identity Tables)

**Spec ref:** §5 Database Schema
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Identity.Infrastructure/Persistence/IdentityDbContext.cs`
- Create: `src/Skillvio.Identity.Infrastructure/Persistence/Configurations/*.cs` (one per aggregate)
- Create: `src/Skillvio.Identity.Infrastructure/Persistence/Migrations/<timestamp>_Init.cs` (auto-generated)
- Create: `src/Skillvio.Identity.Infrastructure/Persistence/Seeders/PlanSeeder.cs`
- Create: `src/Skillvio.Identity.Infrastructure/Persistence/Seeders/RoleSeeder.cs`
- Create: `tests/Skillvio.Identity.IntegrationTests/Persistence/MigrationFixture.cs`
- Create: `tests/Skillvio.Identity.IntegrationTests/Persistence/MigrationTests.cs`

- [ ] **Step 1: Add EF Core packages to Infrastructure**

```bash
dotnet add src/Skillvio.Identity.Infrastructure package Microsoft.EntityFrameworkCore
dotnet add src/Skillvio.Identity.Infrastructure package Microsoft.EntityFrameworkCore.Design
dotnet add src/Skillvio.Identity.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add src/Skillvio.Identity.Infrastructure package EFCore.NamingConventions
dotnet add src/Skillvio.Identity.Infrastructure package Microsoft.AspNetCore.Identity.EntityFrameworkCore
```

- [ ] **Step 2: Write `IdentityDbContext` extending `IdentityDbContext<User, Role, Guid>`**

```csharp
public class IdentityDbContext(DbContextOptions<IdentityDbContext> options)
    : Microsoft.AspNetCore.Identity.EntityFrameworkCore.IdentityDbContext<User, Role, Guid>(options)
{
    public DbSet<UserProfile> UserProfiles => Set<UserProfile>();
    public DbSet<UserPreferences> UserPreferences => Set<UserPreferences>();
    public DbSet<Session> Sessions => Set<Session>();
    public DbSet<RefreshToken> RefreshTokens => Set<RefreshToken>();
    public DbSet<RefreshTokenFamily> RefreshTokenFamilies => Set<RefreshTokenFamily>();
    public DbSet<SubscriptionPlan> Plans => Set<SubscriptionPlan>();
    public DbSet<UserSubscription> Subscriptions => Set<UserSubscription>();
    public DbSet<CertificationApplication> CertificationApplications => Set<CertificationApplication>();
    public DbSet<CertificationDocument> CertificationDocuments => Set<CertificationDocument>();
    public DbSet<TotpSecret> TotpSecrets => Set<TotpSecret>();
    public DbSet<RecoveryCode> RecoveryCodes => Set<RecoveryCode>();
    public DbSet<ComplianceAudit> ComplianceAudits => Set<ComplianceAudit>();
    public DbSet<OnboardingState> OnboardingStates => Set<OnboardingState>();

    protected override void OnModelCreating(ModelBuilder mb)
    {
        base.OnModelCreating(mb);
        mb.HasPostgresExtension("uuid-ossp");
        mb.ApplyConfigurationsFromAssembly(typeof(IdentityDbContext).Assembly);
    }
}
```

- [ ] **Step 3: Write fluent configurations per aggregate**

For each entity in `Configurations/<Entity>Configuration.cs` write `IEntityTypeConfiguration<T>` matching spec §5.1 column types/indexes/constraints. Examples (from spec):

- `users` table renamed via `b.ToTable("users")` (override Identity default)
- `email_lower` computed column with unique index
- `compliance_audit` append-only — no UPDATE permission via `b.ToTable("compliance_audit", t => t.HasCheckConstraint(...))`
- `refresh_tokens` PK = (family_id, token_hash), index by `expires_at`
- `sessions.expires_at` index, `sessions.user_id` index

- [ ] **Step 4: Wire snake_case + register DbContext in Api/Program.cs**

```csharp
builder.Services.AddDbContext<IdentityDbContext>(opt => opt
    .UseNpgsql(builder.Configuration.GetConnectionString("Default"),
        npg => npg.MigrationsAssembly("Skillvio.Identity.Infrastructure"))
    .UseSnakeCaseNamingConvention());

builder.Services.AddIdentityCore<User>(o =>
{
    o.Password.RequiredLength = 8;
    o.Password.RequireDigit = true;
    o.Password.RequireLowercase = true;
    o.Password.RequireNonAlphanumeric = false;
    o.Password.RequireUppercase = false;
    o.User.RequireUniqueEmail = true;
    o.SignIn.RequireConfirmedEmail = true;
})
.AddRoles<Role>()
.AddEntityFrameworkStores<IdentityDbContext>()
.AddDefaultTokenProviders();
```

- [ ] **Step 5: Generate Init migration**

```bash
dotnet ef migrations add Init \
  --project src/Skillvio.Identity.Infrastructure \
  --startup-project src/Skillvio.Identity.Api \
  --context IdentityDbContext
```

- [ ] **Step 6: Augment migration with raw SQL for triggers/views**

In the generated `<timestamp>_Init.Up`:
- Add `CREATE INDEX users_email_lower_idx ON users (LOWER(email))`
- Add unique partial index `users_email_unique_when_active ON users(email) WHERE soft_deleted_at IS NULL`
- Add CHECK constraint preventing UPDATE/DELETE on `compliance_audit` (use a trigger: `CREATE TRIGGER compliance_audit_immutable BEFORE UPDATE OR DELETE ON compliance_audit FOR EACH ROW EXECUTE FUNCTION raise_immutable_error();`)

- [ ] **Step 7: Write Plan + Role seeders**

```csharp
// PlanSeeder
public static async Task SeedAsync(IdentityDbContext db, CancellationToken ct)
{
    var plans = new[]
    {
        new SubscriptionPlan { Code = "free", DisplayName = "Free", PriceMonthly = 0, MaxActiveTracks = 3 },
        new SubscriptionPlan { Code = "pro", DisplayName = "Pro", PriceMonthly = 9.99m, MaxActiveTracks = 5, UnlimitedHearts = true, ExamAccess = true },
        new SubscriptionPlan { Code = "team", DisplayName = "Team", PriceMonthly = 19.99m, MaxActiveTracks = 5, UnlimitedHearts = true, ExamAccess = true, TeamSeats = 5 },
        new SubscriptionPlan { Code = "enterprise", DisplayName = "Enterprise", PriceMonthly = 0 }  // custom pricing
    };
    foreach (var p in plans)
        if (!await db.Plans.AnyAsync(x => x.Code == p.Code, ct)) db.Plans.Add(p);
    await db.SaveChangesAsync(ct);
}

// RoleSeeder
var roles = new[] { "user", "admin", "moderator", "content_author", "support" };
```

Wire seeders in `Program.cs` to run on startup once (idempotent via `AnyAsync` check).

- [ ] **Step 8: Write migration test**

```csharp
[Fact]
public async Task Migrate_AppliesCleanly()
{
    await fx.DbContext.Database.MigrateAsync();
    (await fx.DbContext.Database.GetPendingMigrationsAsync()).Should().BeEmpty();
}

[Fact]
public async Task Migrate_SeedsBuiltinPlans()
{
    await fx.DbContext.Database.MigrateAsync();
    await PlanSeeder.SeedAsync(fx.DbContext, default);
    (await fx.DbContext.Plans.CountAsync()).Should().Be(4);
}
```

- [ ] **Step 9: Run integration tests**

```bash
dotnet test tests/Skillvio.Identity.IntegrationTests
```

- [ ] **Step 10: Commit**

```bash
git commit -am "feat(s1-t3): EF Core 10 + Init migration (Identity tables) + plan/role seeders"
git push
```

---

## Task 4 — Application Layer + Auth Flow (Register/Login/Verify/Refresh/Logout)

**Spec ref:** §6.1 Auth endpoints, §8.1 BCrypt+pepper, §8.5 OAuth state
**Estimated:** 3 days
**Files:**
- Create: `src/Skillvio.Identity.Application/Auth/Commands/RegisterUserCmd.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Commands/LoginCmd.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Commands/VerifyEmailCmd.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Commands/RefreshSessionCmd.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Commands/LogoutCmd.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Commands/ChangePasswordCmd.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Services/IPasswordHasher.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Services/PepperedPasswordHasher.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Services/IRefreshTokenService.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Services/RefreshTokenService.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Services/IEmailVerificationService.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Services/EmailVerificationService.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Services/IAuthService.cs` (orchestrator)
- Create: `src/Skillvio.Identity.Application/Auth/Services/AuthService.cs`
- Create: `src/Skillvio.Identity.Application/Auth/Validators/RegisterUserValidator.cs`
- Create: `tests/Skillvio.Identity.Domain.Tests/Auth/PepperedPasswordHasherTests.cs`
- Create: `tests/Skillvio.Identity.IntegrationTests/Auth/RegisterFlowTests.cs`
- Create: `tests/Skillvio.Identity.IntegrationTests/Auth/LoginFlowTests.cs`
- Create: `tests/Skillvio.Identity.IntegrationTests/Auth/RefreshRotationTests.cs`

- [ ] **Step 1: Add packages**

```bash
dotnet add src/Skillvio.Identity.Application package FluentValidation.AspNetCore
dotnet add src/Skillvio.Identity.Application package BCrypt.Net-Next
```

- [ ] **Step 2: Test PepperedPasswordHasher (TDD red)**

```csharp
public class PepperedPasswordHasherTests
{
    [Fact]
    public void Verify_RoundTrip()
    {
        var sut = new PepperedPasswordHasher(pepper: "test-pepper-32-chars-long-secret-x", workFactor: 4);
        var hash = sut.Hash("MyPassword123");
        sut.Verify("MyPassword123", hash).Should().BeTrue();
        sut.Verify("WrongPass", hash).Should().BeFalse();
    }
    [Fact]
    public void Hash_DifferentPeppers_DontMatch()
    {
        var h1 = new PepperedPasswordHasher("pepper-A-32-chars-long-secret-xx", 4).Hash("p");
        var h2 = new PepperedPasswordHasher("pepper-B-32-chars-long-secret-xx", 4);
        h2.Verify("p", h1).Should().BeFalse();
    }
}
```

- [ ] **Step 3: Implement PepperedPasswordHasher**

```csharp
public class PepperedPasswordHasher(string pepper, int workFactor = 12) : IPasswordHasher
{
    private string Pepper(string pwd) => $"{pwd}|{pepper}";
    public string Hash(string password) => BCrypt.Net.BCrypt.HashPassword(Pepper(password), workFactor);
    public bool Verify(string password, string hash) => BCrypt.Net.BCrypt.Verify(Pepper(password), hash);
}
```

- [ ] **Step 4: Implement RefreshTokenService — secure random + family rotation**

```csharp
public class RefreshTokenService(IdentityDbContext db, IClock clock) : IRefreshTokenService
{
    public (string PlainToken, string Hash) Generate()
    {
        var bytes = RandomNumberGenerator.GetBytes(32);
        var plain = Convert.ToBase64String(bytes);
        var hash = SHA256.HashData(Encoding.UTF8.GetBytes(plain));
        return (plain, Convert.ToHexString(hash));
    }

    public async Task<RotateResult> RotateAsync(string presentedTokenPlain, string deviceId, CancellationToken ct)
    {
        var presentedHash = Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(presentedTokenPlain)));
        var token = await db.RefreshTokens
            .Include(t => t.Family)
            .FirstOrDefaultAsync(t => t.TokenHash == presentedHash, ct);
        if (token is null) return RotateResult.NotFound;
        if (token.Family.IsRevoked) return RotateResult.FamilyRevoked;
        if (token.RevokedAt is not null)
        {
            // REUSE DETECTED — invalidate entire family
            token.Family.InvalidateAll("reuse_detected");
            await db.SaveChangesAsync(ct);
            return RotateResult.ReuseDetected;
        }
        if (token.ExpiresAt < clock.UtcNow) return RotateResult.Expired;

        var (newPlain, newHash) = Generate();
        token.RevokedAt = clock.UtcNow;
        token.Family.AddToken(newHash, clock.UtcNow.AddDays(30));
        await db.SaveChangesAsync(ct);
        return RotateResult.Ok(newPlain, newHash, token.Family.UserId);
    }
}
```

- [ ] **Step 5: Test register flow (integration)**

```csharp
[Fact]
public async Task Register_NewEmail_CreatesUser_SendsVerificationEmail()
{
    var resp = await _client.PostAsJsonAsync("/api/v1/auth/register", new
    {
        Email = "user@example.com",
        Password = "Password123",
        DisplayName = "Test User",
        Locale = "tr-TR",
        Timezone = "Europe/Istanbul"
    });
    resp.StatusCode.Should().Be(HttpStatusCode.Created);

    var user = await _db.Users.SingleAsync(u => u.NormalizedEmail == "USER@EXAMPLE.COM");
    user.EmailConfirmed.Should().BeFalse();

    _emailSenderMock.SentEmails.Should().Contain(e => e.Template == "verify-email");
}
```

- [ ] **Step 6: Test login + email-not-verified blocks**

```csharp
[Fact]
public async Task Login_UnverifiedEmail_Returns403()
{
    await Register("u@e.com", "Password123");
    var resp = await _client.PostAsJsonAsync("/api/v1/auth/login", new { Email = "u@e.com", Password = "Password123" });
    resp.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    var problem = await resp.Content.ReadFromJsonAsync<ProblemDetails>();
    problem!.Detail.Should().Contain("verify email");
}
```

- [ ] **Step 7: Test refresh rotation invalidates family on reuse**

```csharp
[Fact]
public async Task Refresh_ReuseOldToken_InvalidatesFamily()
{
    var (_, refresh1) = await LoginAndExtractTokens(verifiedUser);
    var resp1 = await CallRefresh(refresh1);
    resp1.StatusCode.Should().Be(HttpStatusCode.OK);
    var (_, refresh2) = ExtractCookies(resp1);
    var resp2 = await CallRefresh(refresh1);  // reuse old!
    resp2.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    var resp3 = await CallRefresh(refresh2);  // family invalidated
    resp3.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
}
```

- [ ] **Step 8: Implement remaining commands (Verify, ChangePassword, Logout)**

Each follows the same structure: command + handler in Application; integration test before implementation.

- [ ] **Step 9: Run all auth tests**

```bash
dotnet test tests/Skillvio.Identity.IntegrationTests --filter "FullyQualifiedName~Auth"
```

- [ ] **Step 10: Commit**

```bash
git commit -am "feat(s1-t4): auth flow (register/login/verify/refresh/logout) with peppered hash + family rotation"
git push
```

---

## Task 5 — Cookie + Redis Session

**Spec ref:** §3 Soru3 (Session model B), §8.4 Cookie güvenliği
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Identity.Infrastructure/Sessions/RedisSessionStore.cs`
- Create: `src/Skillvio.Identity.Infrastructure/Cookies/SkillvioCookieOptions.cs`
- Modify: `src/Skillvio.Identity.Api/Program.cs` (configure cookie auth + Redis backplane + DataProtection)

- [ ] **Step 1: Add packages**

```bash
dotnet add src/Skillvio.Identity.Infrastructure package StackExchange.Redis
dotnet add src/Skillvio.Identity.Infrastructure package Microsoft.AspNetCore.DataProtection.StackExchangeRedis
dotnet add src/Skillvio.Identity.Api package Microsoft.Extensions.Caching.StackExchangeRedis
```

- [ ] **Step 2: Configure DataProtection with Redis (shared keys for multi-instance)**

```csharp
var redis = ConnectionMultiplexer.Connect(builder.Configuration["ConnectionStrings:Redis"]!);
builder.Services.AddSingleton<IConnectionMultiplexer>(redis);
builder.Services.AddDataProtection()
    .PersistKeysToStackExchangeRedis(redis, "skillvio-identity:dataprotection-keys")
    .SetApplicationName("skillvio");
```

- [ ] **Step 3: Configure cookie auth**

```csharp
builder.Services.AddAuthentication(IdentityConstants.ApplicationScheme)
    .AddCookie(IdentityConstants.ApplicationScheme, opt =>
    {
        opt.Cookie.Name = "skillvio.sid";
        opt.Cookie.HttpOnly = true;
        opt.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        opt.Cookie.SameSite = SameSiteMode.Lax;
        opt.Cookie.Domain = builder.Configuration["Cookies:Domain"];  // ".skillvio.io" prod
        opt.ExpireTimeSpan = TimeSpan.FromMinutes(15);
        opt.SlidingExpiration = false;
        opt.Events.OnRedirectToLogin = ctx => { ctx.Response.StatusCode = 401; return Task.CompletedTask; };
        opt.Events.OnRedirectToAccessDenied = ctx => { ctx.Response.StatusCode = 403; return Task.CompletedTask; };
    });
```

- [ ] **Step 4: Refresh cookie path-restricted to /api/v1/auth/refresh**

When issuing refresh cookie:
```csharp
ctx.Response.Cookies.Append("skillvio.rt", plainRefresh, new CookieOptions
{
    HttpOnly = true, Secure = true, SameSite = SameSiteMode.Lax,
    Path = "/api/v1/auth/refresh",
    Domain = config["Cookies:Domain"],
    Expires = DateTimeOffset.UtcNow.AddDays(30)
});
```

- [ ] **Step 5: Test cookie is HttpOnly + Secure + Lax**

```csharp
[Fact]
public async Task Login_SetsSecureHttpOnlyCookies()
{
    var resp = await Login(verifiedUser);
    var sid = resp.Headers.GetValues("Set-Cookie").Single(c => c.StartsWith("skillvio.sid"));
    sid.Should().Contain("HttpOnly").And.Contain("Secure").And.Contain("SameSite=Lax");
}
```

- [ ] **Step 6: Test session persists across instances (Redis backplane)**

Run two app instances pointing to same Redis; verify cookie issued by instance A is decryptable by instance B.

- [ ] **Step 7: Commit**

```bash
git commit -am "feat(s1-t5): cookie session + Redis DataProtection backplane + path-scoped refresh cookie"
git push
```

---

## Task 6 — API Endpoints (Minimal APIs, ~30 endpoints)

**Spec ref:** §6 API Spec (Auth, Users, Certifications, Onboarding, Admin, Health)
**Estimated:** 2 days
**Files:**
- Create: `src/Skillvio.Identity.Api/Endpoints/AuthEndpoints.cs`
- Create: `src/Skillvio.Identity.Api/Endpoints/UsersEndpoints.cs`
- Create: `src/Skillvio.Identity.Api/Endpoints/CertificationsEndpoints.cs`
- Create: `src/Skillvio.Identity.Api/Endpoints/OnboardingEndpoints.cs`
- Create: `src/Skillvio.Identity.Api/Endpoints/AdminEndpoints.cs`
- Create: `src/Skillvio.Identity.Api/Endpoints/HealthEndpoints.cs`
- Create: `src/Skillvio.Identity.Api/Endpoints/MfaEndpoints.cs`
- Create: `src/Skillvio.Identity.Api/Filters/RequireRoleFilter.cs`
- Create: `src/Skillvio.Identity.Api/Filters/RateLimitPolicies.cs`
- Create: `tests/Skillvio.Identity.IntegrationTests/Api/*EndpointTests.cs` (one per endpoint group)

- [ ] **Step 1: Implement Auth endpoints**

```csharp
public static class AuthEndpoints
{
    public static void Map(IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/api/v1/auth").WithTags("Auth");
        g.MapPost("/register", async (RegisterRequest req, IAuthService svc, CancellationToken ct) =>
        {
            await svc.RegisterAsync(new RegisterUserCmd(req.Email, req.Password, req.DisplayName, req.Locale, req.Timezone), ct);
            return Results.Created("/api/v1/auth/me", null);
        }).WithRequestValidation<RegisterRequest>();

        g.MapPost("/login", LoginHandler);
        g.MapPost("/verify-email", VerifyEmailHandler);
        g.MapPost("/refresh", RefreshHandler);
        g.MapPost("/logout", LogoutHandler).RequireAuthorization();
        g.MapPost("/forgot-password", ForgotPasswordHandler);
        g.MapPost("/reset-password", ResetPasswordHandler);
        g.MapPost("/change-password", ChangePasswordHandler).RequireAuthorization();
        g.MapGet("/google", GoogleOAuthChallenge);
        g.MapGet("/google/callback", GoogleOAuthCallback);
        g.MapGet("/github", GithubOAuthChallenge);
        g.MapGet("/github/callback", GithubOAuthCallback);
    }
}
```

- [ ] **Step 2: Implement Users endpoints (`/me/*`)**

`GET /api/v1/users/me`, `PATCH /api/v1/users/me`, `GET /api/v1/users/me/preferences`, `PATCH /api/v1/users/me/preferences`, `GET /api/v1/users/me/sessions`, `DELETE /api/v1/users/me/sessions/{id}`, `POST /api/v1/users/me/soft-delete`, `POST /api/v1/users/me/restore`, `GET /api/v1/users/search?q=&limit=20` (federated search for Sprint 3 friends).

- [ ] **Step 3: Implement Certifications endpoints**

`POST /api/v1/certifications/applications` (apply with documents), `GET /api/v1/certifications/applications/me`, admin: `GET /api/v1/admin/certifications/pending`, `POST /api/v1/admin/certifications/{id}/approve`, `POST /api/v1/admin/certifications/{id}/reject`.

- [ ] **Step 4: Implement Onboarding endpoints**

`GET /api/v1/onboarding/state`, `PATCH /api/v1/onboarding/state` (3-step skippable per spec §3 Soru5), `POST /api/v1/onboarding/complete`.

- [ ] **Step 5: Implement Admin endpoints (role-gated)**

`GET /api/v1/admin/users`, `GET /api/v1/admin/users/{id}`, `POST /api/v1/admin/users/{id}/lock`, `POST /api/v1/admin/users/{id}/unlock`, `POST /api/v1/admin/users/{id}/hard-delete` (immediate, with audit), `POST /api/v1/admin/users/{id}/grant-role`, `POST /api/v1/admin/users/{id}/revoke-role`, `POST /api/v1/admin/users/{id}/change-plan`.

- [ ] **Step 6: Implement MFA endpoints**

`POST /api/v1/mfa/setup` (returns QR data URL + secret), `POST /api/v1/mfa/enable` (verify TOTP code, store), `POST /api/v1/mfa/disable`, `POST /api/v1/mfa/recovery-codes/regenerate`.

- [ ] **Step 7: Implement Health endpoints**

`GET /health/live` (process alive), `GET /health/ready` (DB + Redis + RabbitMQ readiness).

- [ ] **Step 8: Configure rate limiting policies**

```csharp
builder.Services.AddRateLimiter(opt =>
{
    opt.AddPolicy("AuthLogin", ctx =>
        RateLimitPartition.GetFixedWindowLimiter(GetIpOrUser(ctx), _ =>
            new FixedWindowRateLimiterOptions { PermitLimit = 5, Window = TimeSpan.FromMinutes(15) }));
    opt.AddPolicy("AuthSignup", ctx =>
        RateLimitPartition.GetFixedWindowLimiter(GetIp(ctx), _ =>
            new FixedWindowRateLimiterOptions { PermitLimit = 3, Window = TimeSpan.FromHours(1) }));
});
```

- [ ] **Step 9: Run integration tests for all endpoints**

```bash
dotnet test tests/Skillvio.Identity.IntegrationTests --filter "FullyQualifiedName~Endpoint"
```

- [ ] **Step 10: Commit**

```bash
git commit -am "feat(s1-t6): API endpoints (Auth, Users, Cert, Onboarding, Admin, MFA, Health) + rate limit"
git push
```

---

## Task 7 — Event Publishing (MassTransit Outbox)

**Spec ref:** §7 Domain Events
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Identity.Application/Events/IOutbox.cs`
- Create: `src/Skillvio.Identity.Infrastructure/Messaging/OutboxWriter.cs`
- Create: `src/Skillvio.Identity.Infrastructure/Messaging/OutboxDispatcher.cs`
- Modify: `Program.cs` (MassTransit config)

- [ ] **Step 1: Add MassTransit packages**

```bash
dotnet add src/Skillvio.Identity.Infrastructure package MassTransit
dotnet add src/Skillvio.Identity.Infrastructure package MassTransit.RabbitMQ
dotnet add src/Skillvio.Identity.Infrastructure package MassTransit.EntityFrameworkCore
dotnet add src/Skillvio.Identity.Application package Skillvio.Shared.Contracts
```

- [ ] **Step 2: Add outbox table via migration**

```bash
dotnet ef migrations add AddOutbox \
  --project src/Skillvio.Identity.Infrastructure \
  --startup-project src/Skillvio.Identity.Api
```

Schema:
```csharp
modelBuilder.Entity<OutboxMessage>(b =>
{
    b.ToTable("outbox_messages");
    b.HasKey(x => x.Id);
    b.Property(x => x.Payload).HasColumnType("jsonb");
    b.HasIndex(x => x.OccurredAt).HasFilter("dispatched_at IS NULL");
});
```

Add LISTEN/NOTIFY trigger via raw SQL (same pattern as Sprint 3a T2).

- [ ] **Step 3: Implement OutboxWriter**

```csharp
public class OutboxWriter(IdentityDbContext db, IClock clock) : IOutbox
{
    public async Task AddAsync<TEvent>(TEvent evt, CancellationToken ct = default) where TEvent : class
    {
        var msg = new OutboxMessage
        {
            Id = Guid.NewGuid(),
            OccurredAt = clock.UtcNow,
            EventType = typeof(TEvent).Name,
            Payload = JsonSerializer.SerializeToDocument(evt),
            DispatchedAt = null
        };
        await db.OutboxMessages.AddAsync(msg, ct);
    }
}
```

- [ ] **Step 4: Implement OutboxDispatcher (BackgroundService with LISTEN/NOTIFY)**

Same pattern as Sprint 3a T10. Dispatches to RabbitMQ via MassTransit `IBus`.

- [ ] **Step 5: Wire publishers into command handlers**

In `RegisterAsync`: after user created, await `_outbox.AddAsync(new UserRegisteredEvent(...))`. Spec §7.1 lists all events to publish: UserRegisteredEvent, UserSoftDeletedEvent, UserHardDeletedEvent, UserPlanChangedEvent, UserProfileUpdatedEvent, CertificationVerifiedEvent.

- [ ] **Step 6: Test event published after register**

```csharp
[Fact]
public async Task Register_PublishesUserRegisteredEvent()
{
    await Register("u@e.com", "Password123", displayName: "U");
    var outbox = await _db.OutboxMessages.Where(o => o.EventType == "UserRegisteredEvent").ToListAsync();
    outbox.Should().HaveCount(1);
}
```

- [ ] **Step 7: Test dispatcher publishes to bus**

Use MassTransit's test harness:
```csharp
[Fact]
public async Task Dispatcher_PublishesEventToBus()
{
    var harness = await StartBusHarness();
    await Register("u@e.com", "Password123", displayName: "U");
    await Eventually(async () => harness.Published.Select<UserRegisteredEvent>().Any().Should().BeTrue());
}
```

- [ ] **Step 8: Commit**

```bash
git commit -am "feat(s1-t7): outbox pattern + MassTransit RabbitMQ + 6 event publishers"
git push
```

---

## Task 8 — Hangfire Jobs

**Spec ref:** §3 Soru1 (auto-delete unverified 7d), §3 Soru4 (soft→hard delete 30d)
**Estimated:** 0.5 day
**Files:**
- Create: `src/Skillvio.Identity.Infrastructure/Jobs/UnverifiedUserCleanup.cs`
- Create: `src/Skillvio.Identity.Infrastructure/Jobs/SoftDeleteFinalizer.cs`
- Create: `src/Skillvio.Identity.Infrastructure/Jobs/RefreshTokenCleanup.cs`
- Create: `src/Skillvio.Identity.Infrastructure/Jobs/SessionExpirer.cs`
- Modify: `Program.cs` (Hangfire config)

- [ ] **Step 1: Add Hangfire packages**

```bash
dotnet add src/Skillvio.Identity.Api package Hangfire.AspNetCore
dotnet add src/Skillvio.Identity.Api package Hangfire.PostgreSql
```

- [ ] **Step 2: Configure Hangfire**

```csharp
builder.Services.AddHangfire(c => c
    .UsePostgreSqlStorage(opt => opt.UseNpgsqlConnection(builder.Configuration.GetConnectionString("Hangfire"))));
builder.Services.AddHangfireServer();
```

- [ ] **Step 3: Implement UnverifiedUserCleanup (daily)**

```csharp
public class UnverifiedUserCleanup(IdentityDbContext db, IClock clock)
{
    public async Task RunAsync(CancellationToken ct)
    {
        var cutoff = clock.UtcNow.AddDays(-7);
        await db.Users.Where(u => !u.EmailConfirmed && u.CreatedAt < cutoff).ExecuteDeleteAsync(ct);
    }
}
// Schedule: RecurringJob.AddOrUpdate<UnverifiedUserCleanup>("user.cleanup.unverified", j => j.RunAsync(default), "0 4 * * *");
```

- [ ] **Step 4: Implement SoftDeleteFinalizer (daily, finalizes soft→hard after 30 days)**

```csharp
public class SoftDeleteFinalizer(IdentityDbContext db, IOutbox outbox, IClock clock)
{
    public async Task RunAsync(CancellationToken ct)
    {
        var due = await db.Users.Where(u => u.SoftDeletedAt != null && u.HardDeleteScheduledAt < clock.UtcNow).ToListAsync(ct);
        foreach (var user in due)
        {
            await outbox.AddAsync(new UserHardDeletedEvent(Guid.NewGuid(), clock.UtcNow, user.Id), ct);
            db.Users.Remove(user);  // cascade via FK or explicit cleanup of related rows
            db.ComplianceAudits.Add(new ComplianceAudit { UserId = user.Id, Action = "data_deleted_hard", PerformedAt = clock.UtcNow });
        }
        await db.SaveChangesAsync(ct);
    }
}
// Schedule: "0 5 * * *"
```

- [ ] **Step 5: Implement RefreshTokenCleanup + SessionExpirer**

`RefreshTokenCleanup`: delete revoked tokens older than 90 days.
`SessionExpirer`: delete sessions where expires_at < now.

- [ ] **Step 6: Test soft→hard delete flow**

```csharp
[Fact]
public async Task SoftDelete_After30Days_BecomesHardDelete()
{
    var user = await CreateAndSoftDelete();
    AdvanceClock(31);
    await new SoftDeleteFinalizer(_db, _outbox, _clock).RunAsync(default);
    var deleted = await _db.Users.AnyAsync(u => u.Id == user.Id);
    deleted.Should().BeFalse();
    var ev = await _db.OutboxMessages.AnyAsync(o => o.EventType == "UserHardDeletedEvent");
    ev.Should().BeTrue();
    var audit = await _db.ComplianceAudits.AnyAsync(a => a.UserId == user.Id && a.Action == "data_deleted_hard");
    audit.Should().BeTrue();
}
```

- [ ] **Step 7: Commit**

```bash
git commit -am "feat(s1-t8): Hangfire jobs (unverified cleanup, soft-delete finalizer, token/session cleanup)"
git push
```

---

## Task 9 — Tests + Coverage

**Spec ref:** §11 S1.T9
**Estimated:** 2 days
**Files:** All test files in `tests/Skillvio.Identity.Domain.Tests/` and `tests/Skillvio.Identity.IntegrationTests/`

- [ ] **Step 1: Audit current coverage**

```bash
dotnet test --collect "XPlat Code Coverage"
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coveragereport" -reporttypes:"Html"
open coveragereport/index.html
```

Identify gaps: each service/handler/aggregate should have at least one test covering happy path + 1-2 edge cases.

- [ ] **Step 2: Fill in unit tests for Application services**

For each service in `Skillvio.Identity.Application/` write 3-5 tests covering:
- Happy path
- Validation rejection
- Concurrency / race
- Idempotency (where applicable)

- [ ] **Step 3: Fill in integration tests for sad paths**

- Login with locked account → 401 + lockout reason
- Refresh with expired token → 401
- Hard-delete user from admin → 204 + downstream services notified
- Onboarding state transitions
- MFA setup → enable → verify → login flow

- [ ] **Step 4: Verify coverage ≥ 70%**

```bash
dotnet test --collect "XPlat Code Coverage"
# Inspect generated report
```

If below 70%, add targeted tests until target met.

- [ ] **Step 5: Commit**

```bash
git commit -am "test(s1-t9): coverage to ≥70% (unit + integration sad paths)"
git push
```

---

## Task 10 — OpenTelemetry + Loki + Prometheus

**Spec ref:** §9 Observability
**Estimated:** 1 day
**Files:**
- Create: `src/Skillvio.Identity.Api/Observability/SerilogConfig.cs`
- Create: `src/Skillvio.Identity.Api/Observability/OpenTelemetryConfig.cs`
- Create: `src/Skillvio.Identity.Application/Compliance/IComplianceLogger.cs`
- Create: `src/Skillvio.Identity.Infrastructure/Compliance/ComplianceLogger.cs`
- Modify: `Program.cs`

- [ ] **Step 1: Add packages**

```bash
dotnet add src/Skillvio.Identity.Api package Serilog.AspNetCore
dotnet add src/Skillvio.Identity.Api package Serilog.Sinks.Grafana.Loki
dotnet add src/Skillvio.Identity.Api package OpenTelemetry.Extensions.Hosting
dotnet add src/Skillvio.Identity.Api package OpenTelemetry.Instrumentation.AspNetCore
dotnet add src/Skillvio.Identity.Api package OpenTelemetry.Instrumentation.EntityFrameworkCore
dotnet add src/Skillvio.Identity.Api package OpenTelemetry.Instrumentation.Http
dotnet add src/Skillvio.Identity.Api package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add src/Skillvio.Identity.Api package OpenTelemetry.Exporter.Prometheus.AspNetCore
```

- [ ] **Step 2: Configure Serilog → Loki**

```csharp
builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("service", "skillvio-identity")
    .WriteTo.Console()
    .WriteTo.GrafanaLoki(ctx.Configuration["Loki:Endpoint"]!,
        labels: [new() { Key = "service", Value = "skillvio-identity" }],
        propertiesAsLabels: ["level", "RequestPath"]));
```

- [ ] **Step 3: Configure OpenTelemetry**

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("skillvio-identity"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("MassTransit")
        .AddOtlpExporter(o => o.Endpoint = new Uri(builder.Configuration["Otel:Endpoint"]!)))
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter());
app.MapPrometheusScrapingEndpoint();  // /metrics
```

- [ ] **Step 4: Implement compliance logger**

```csharp
public class ComplianceLogger(IdentityDbContext db, IClock clock) : IComplianceLogger
{
    public async Task RecordAsync(string action, Guid? userId, string? reason, JsonDocument? context, CancellationToken ct)
    {
        db.ComplianceAudits.Add(new ComplianceAudit
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            Action = action,
            Reason = reason,
            Context = context,
            PerformedAt = clock.UtcNow
        });
        await db.SaveChangesAsync(ct);
    }
}
```

Wire into commands that have compliance implications (per spec §5.2):
- `account_created`, `email_verified`, `password_changed`, `mfa_enabled`, `mfa_disabled`, `account_locked`, `data_exported`, `data_deleted_soft`, `data_deleted_hard`, `cert_approved`, `cert_revoked`, `plan_changed`.

- [ ] **Step 5: Test compliance audit append-only**

```csharp
[Fact]
public async Task ComplianceAudit_CannotBeUpdatedOrDeleted()
{
    await _logger.RecordAsync("password_changed", _testUser.Id, null, null, default);
    var row = await _db.ComplianceAudits.SingleAsync();
    row.Action = "tampered";
    var act = async () => await _db.SaveChangesAsync();
    await act.Should().ThrowAsync<DbUpdateException>();
}
```

- [ ] **Step 6: Verify metrics scrape**

```bash
dotnet run --project src/Skillvio.Identity.Api &
curl -s http://localhost:8080/metrics | grep aspnetcore_request
```

- [ ] **Step 7: Commit**

```bash
git commit -am "feat(s1-t10): Serilog → Loki + OTel → Tempo + Prometheus + compliance audit logger"
git push
```

---

## Task 11 — Docker + GitHub Actions CI

**Spec ref:** §11 S1.T11
**Estimated:** 0.5 day
**Files:**
- Create: `src/Skillvio.Identity.Api/Dockerfile`
- Create: `.dockerignore`
- Modify: `.github/workflows/ci.yml` (add docker build job)

- [ ] **Step 1: Write multi-stage Dockerfile**

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["Directory.Build.props", "Directory.Packages.props", "global.json", "skillvio-identity-service.sln", "./"]
COPY ["src/", "src/"]
RUN dotnet restore "src/Skillvio.Identity.Api/Skillvio.Identity.Api.csproj"
RUN dotnet publish "src/Skillvio.Identity.Api/Skillvio.Identity.Api.csproj" -c Release -o /app --no-restore /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
RUN groupadd -r skillvio && useradd -r -g skillvio skillvio
COPY --from=build --chown=skillvio:skillvio /app .
USER skillvio
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
HEALTHCHECK --interval=30s --timeout=5s CMD curl -f http://localhost:8080/health/ready || exit 1
ENTRYPOINT ["dotnet", "Skillvio.Identity.Api.dll"]
```

- [ ] **Step 2: Build image locally**

```bash
docker build -t skillvio-identity:dev -f src/Skillvio.Identity.Api/Dockerfile .
docker run --rm -p 8080:8080 \
  -e ConnectionStrings__Default="..." \
  skillvio-identity:dev
curl http://localhost:8080/health/live  # → 200
```

- [ ] **Step 3: Add docker-build job to CI**

Append to `.github/workflows/ci.yml`:
```yaml
docker-build:
  needs: build-test
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: docker/setup-buildx-action@v3
    - uses: docker/login-action@v3
      if: github.ref == 'refs/heads/main'
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - uses: docker/build-push-action@v5
      with:
        context: .
        file: src/Skillvio.Identity.Api/Dockerfile
        push: ${{ github.ref == 'refs/heads/main' }}
        tags: |
          ghcr.io/skillvio/skillvio-identity:${{ github.sha }}
          ghcr.io/skillvio/skillvio-identity:latest
```

- [ ] **Step 4: Commit**

```bash
git commit -am "feat(s1-t11): multi-stage Dockerfile + GHA docker-build job"
git push
```

---

## Task 12 — Frontend Smoke Test

**Spec ref:** §11 S1.T12
**Estimated:** 0.5 day
**Files:**
- Create: `tests/Skillvio.Identity.SmokeTests/postman-collection.json` (or similar)
- Create: `tests/Skillvio.Identity.SmokeTests/run-smoke.sh`

- [ ] **Step 1: Write smoke test script (curl-based)**

```bash
#!/bin/bash
# tests/Skillvio.Identity.SmokeTests/run-smoke.sh
set -euo pipefail
BASE=${BASE_URL:-http://localhost:8080}
EMAIL="smoke-$(date +%s)@example.com"

echo "1. Register"
curl -fsS -X POST "$BASE/api/v1/auth/register" \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$EMAIL\",\"password\":\"Password123\",\"displayName\":\"Smoke\",\"locale\":\"tr-TR\",\"timezone\":\"UTC\"}"

echo "2. Verify email (using internal helper to bypass real email)"
TOKEN=$(psql -h localhost -U skillvio -d identity_db -t -c "SELECT email_confirmation_token FROM users WHERE email = '$EMAIL' LIMIT 1;")
curl -fsS -X POST "$BASE/api/v1/auth/verify-email" -H "Content-Type: application/json" -d "{\"token\":\"$TOKEN\"}"

echo "3. Login"
curl -fsS -X POST "$BASE/api/v1/auth/login" -c cookies.txt \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$EMAIL\",\"password\":\"Password123\"}"

echo "4. Get me"
curl -fsS "$BASE/api/v1/users/me" -b cookies.txt | jq .

echo "5. Logout"
curl -fsS -X POST "$BASE/api/v1/auth/logout" -b cookies.txt

echo "✅ Smoke passed"
```

- [ ] **Step 2: Run smoke against docker-composed stack**

```bash
cd ../skillvio-infra && docker compose up -d
cd ../skillvio-identity-service && dotnet run --project src/Skillvio.Identity.Api &
sleep 10
./tests/Skillvio.Identity.SmokeTests/run-smoke.sh
```

- [ ] **Step 3: Commit**

```bash
git commit -am "test(s1-t12): smoke test — full register→verify→login→logout flow"
git push
```

---

## Sprint 1 Wrap-Up

- [ ] **Step 1: Run full test suite + coverage**

```bash
dotnet test --collect "XPlat Code Coverage"
```

Targets: Domain ≥ 80%, Application ≥ 70%, Integration tests cover all happy paths + critical sad paths.

- [ ] **Step 2: Verify DoD against spec §11**

- [ ] All 30+ endpoints work end-to-end
- [ ] Cookie session works across instances (Redis backplane verified)
- [ ] Email verification + welcome email send via Resend (dev: Mailtrap)
- [ ] Refresh rotation + family invalidation tested
- [ ] OAuth (Google + GitHub) flow works (with placeholder credentials in dev)
- [ ] Soft delete + 30-day finalizer + UserHardDeletedEvent published
- [ ] Compliance audit logged for all sensitive actions
- [ ] OTel traces visible in Tempo (Grafana)
- [ ] Loki shows structured JSON logs
- [ ] Prometheus scrapes /metrics

- [ ] **Step 3: Tag release**

```bash
git tag -a v0.1.0-sprint1 -m "Sprint 1 complete — Identity Service"
git push --tags
```

---

## Self-Review Checklist (post-write)

- [ ] **Spec coverage:** All 12 spec tasks (S1.T1–T12) covered ✅
- [ ] **Placeholder scan:** No "TBD" or "implement later" ✅
- [ ] **Type consistency:** `IAuthService` methods consistent, `IOutbox.AddAsync` matches Sprint 3 plan, `IClock` interface matches Sprint 3 plan ✅
- [ ] **Cross-task integrity:** T7 outbox depends on T3 migration; T9 tests depend on T4-T8 implementations; T10 compliance logger wired into T4-T8 commands ✅

If any issue surfaces during execution, fix inline and continue.
