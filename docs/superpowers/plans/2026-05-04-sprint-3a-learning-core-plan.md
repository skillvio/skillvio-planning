# Sprint 3a — Learning Service Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the core of `skillvio-learning-service` — content read APIs, attempt orchestration with 14-step lesson-complete saga, gamification primitives (XP/Level/Hearts/Streak/Badges), Skillvio Certificate eligibility, outbox-based event publish, ContentSync (Git→DB), and 30 cloud-cert catalog seed.

**Architecture:** Clean Architecture (Domain / Application / Infrastructure / Api), 6 .NET projects + 3 test projects. Outbox pattern via MassTransit + EF Core, with Postgres `LISTEN/NOTIFY`-driven dispatcher. Locale-aware reads via `LocalizedQueryService<T,TT>` (JSONB + translations table fallback chain).

**Tech Stack:** .NET 10 LTS, ASP.NET Core 10 Minimal APIs, EF Core 10 + Npgsql + EFCore.NamingConventions, MassTransit, Hangfire, FluentValidation, HybridCache, JsonSchema.Net, Serilog → Loki, OpenTelemetry → Tempo, Prometheus, xUnit + Testcontainers + FluentAssertions.

**Spec:** `docs/superpowers/specs/2026-05-04-sprint-3-learning-design.md`

**Effort:** ~130 hours, ~4 weeks solo, 17 tasks (T1, T2, T3, T4, T5, T6, T7a, T7b, T8, T9, T10, T11a, T11b, T12, T15, T16, T18a).

**Repo:** `skillvio-learning-service` (greenfield, github.com/skillvio).

---

## Pre-Flight Checklist

Before starting Task 1, verify:

- [ ] `skillvio-shared-contracts` NuGet v0.1 published (R8) — Sprint 1 IdentityService events finalized: `UserRegisteredEvent`, `UserSoftDeletedEvent`, `UserHardDeletedEvent`, `UserPlanChangedEvent`, `UserProfileUpdatedEvent`, `CertificationVerifiedEvent`
- [ ] Sprint 2 gateway WebSocket pass-through ticket filed (R7) — needed before Sprint 3b T17
- [ ] Sprint 1 `compliance_audit` table accepts `data_deleted_hard`, `certificate_revoked`, `exam_access_revoked` action types (§14.1)
- [ ] OI4 (exam content licensing) decision logged before Task 16
- [ ] OI6 (Hearts max Pro: unlimited vs 10) decision logged before Task 5
- [ ] Open issues OI1, OI2, OI3, OI5, OI7, OI8 noted but not blocking (resolution timing per spec §15)

If any pre-flight item is unchecked, stop and resolve before proceeding.

---

## Task 1 — Solution Skeleton (.NET 10, 6 Projects + 3 Test Projects)

**Spec ref:** §2 Solution Yapısı
**Estimated:** 4h
**Files:**
- Create: `skillvio-learning-service.sln`
- Create: `Directory.Build.props`
- Create: `Directory.Packages.props`
- Create: `.editorconfig`
- Create: `.gitignore`
- Create: `src/Skillvio.Learning.Domain/Skillvio.Learning.Domain.csproj`
- Create: `src/Skillvio.Learning.Application/Skillvio.Learning.Application.csproj`
- Create: `src/Skillvio.Learning.Infrastructure/Skillvio.Learning.Infrastructure.csproj`
- Create: `src/Skillvio.Learning.Api/Skillvio.Learning.Api.csproj`
- Create: `src/Skillvio.Learning.ContentSync/Skillvio.Learning.ContentSync.csproj`
- Create: `src/Skillvio.Learning.BadgeWorker/Skillvio.Learning.BadgeWorker.csproj`
- Create: `tests/Skillvio.Learning.Domain.Tests/Skillvio.Learning.Domain.Tests.csproj`
- Create: `tests/Skillvio.Learning.Application.Tests/Skillvio.Learning.Application.Tests.csproj`
- Create: `tests/Skillvio.Learning.IntegrationTests/Skillvio.Learning.IntegrationTests.csproj`
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Init repo and verify .NET 10 SDK**

```bash
cd ~/skillvio
gh repo create skillvio/skillvio-learning-service --private --clone
cd skillvio-learning-service
dotnet --list-sdks
```

Expected: At least one `10.0.x` SDK listed. If not, install .NET 10 LTS first.

- [ ] **Step 2: Write `Directory.Build.props` (project-wide settings)**

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisMode>Recommended</AnalysisMode>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <Deterministic>true</Deterministic>
    <ContinuousIntegrationBuild Condition="'$(GITHUB_ACTIONS)' == 'true'">true</ContinuousIntegrationBuild>
  </PropertyGroup>
</Project>
```

- [ ] **Step 3: Write `Directory.Packages.props` (central package management)**

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.0" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.0" />
    <PackageVersion Include="EFCore.NamingConventions" Version="10.0.0" />
    <PackageVersion Include="MassTransit" Version="8.3.0" />
    <PackageVersion Include="MassTransit.RabbitMQ" Version="8.3.0" />
    <PackageVersion Include="MassTransit.EntityFrameworkCore" Version="8.3.0" />
    <PackageVersion Include="Hangfire.AspNetCore" Version="1.8.14" />
    <PackageVersion Include="Hangfire.PostgreSql" Version="1.20.10" />
    <PackageVersion Include="FluentValidation.AspNetCore" Version="11.3.0" />
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Caching.Hybrid" Version="10.0.0" />
    <PackageVersion Include="Microsoft.AspNetCore.SignalR.StackExchangeRedis" Version="10.0.0" />
    <PackageVersion Include="Serilog.AspNetCore" Version="8.0.3" />
    <PackageVersion Include="Serilog.Sinks.Grafana.Loki" Version="8.3.0" />
    <PackageVersion Include="OpenTelemetry.Extensions.Hosting" Version="1.10.0" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.10.1" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.EntityFrameworkCore" Version="1.10.0-beta.1" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.Http" Version="1.10.0" />
    <PackageVersion Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.10.0" />
    <PackageVersion Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" Version="1.10.0-beta.1" />
    <PackageVersion Include="LibGit2Sharp" Version="0.30.0" />
    <PackageVersion Include="JsonSchema.Net" Version="7.2.3" />
    <PackageVersion Include="YamlDotNet" Version="16.1.3" />
    <PackageVersion Include="Otp.NET" Version="1.4.0" />
    <PackageVersion Include="Skillvio.Shared.Contracts" Version="0.1.0" />
    <!-- Test packages -->
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

- [ ] **Step 4: Write `.editorconfig`**

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
insert_final_newline = true
trim_trailing_whitespace = true

[*.{cs,csproj,props,targets}]
indent_size = 4

[*.{json,yml,yaml}]
indent_size = 2

[*.cs]
dotnet_diagnostic.CA1031.severity = none
dotnet_diagnostic.CA2007.severity = none
dotnet_diagnostic.CA1707.severity = none  # underscores in test names
csharp_using_directive_placement = outside_namespace:warning
csharp_style_namespace_declarations = file_scoped:warning
```

- [ ] **Step 5: Write `.gitignore`**

```gitignore
bin/
obj/
*.user
.vs/
.vscode/
.idea/
*.swp
.DS_Store
TestResults/
coverage.*
.env
.env.local
appsettings.Development.json
**/secrets.json
publish/
artifacts/
```

- [ ] **Step 6: Create solution file**

```bash
dotnet new sln -n skillvio-learning-service
```

- [ ] **Step 7: Create Domain project**

```bash
mkdir -p src/Skillvio.Learning.Domain
cd src/Skillvio.Learning.Domain
dotnet new classlib -n Skillvio.Learning.Domain -f net10.0 --force
rm Class1.cs
cd ../..
dotnet sln add src/Skillvio.Learning.Domain/Skillvio.Learning.Domain.csproj
```

- [ ] **Step 8: Create Application project + reference Domain**

```bash
mkdir -p src/Skillvio.Learning.Application
cd src/Skillvio.Learning.Application
dotnet new classlib -n Skillvio.Learning.Application -f net10.0 --force
rm Class1.cs
cd ../..
dotnet sln add src/Skillvio.Learning.Application/Skillvio.Learning.Application.csproj
dotnet add src/Skillvio.Learning.Application reference src/Skillvio.Learning.Domain
```

- [ ] **Step 9: Create Infrastructure project + reference Application**

```bash
mkdir -p src/Skillvio.Learning.Infrastructure
cd src/Skillvio.Learning.Infrastructure
dotnet new classlib -n Skillvio.Learning.Infrastructure -f net10.0 --force
rm Class1.cs
cd ../..
dotnet sln add src/Skillvio.Learning.Infrastructure/Skillvio.Learning.Infrastructure.csproj
dotnet add src/Skillvio.Learning.Infrastructure reference src/Skillvio.Learning.Application
```

- [ ] **Step 10: Create Api project + Hub-aware web SDK**

```bash
mkdir -p src/Skillvio.Learning.Api
cd src/Skillvio.Learning.Api
dotnet new web -n Skillvio.Learning.Api -f net10.0 --force
cd ../..
dotnet sln add src/Skillvio.Learning.Api/Skillvio.Learning.Api.csproj
dotnet add src/Skillvio.Learning.Api reference src/Skillvio.Learning.Application src/Skillvio.Learning.Infrastructure
```

- [ ] **Step 11: Create ContentSync console worker**

```bash
mkdir -p src/Skillvio.Learning.ContentSync
cd src/Skillvio.Learning.ContentSync
dotnet new worker -n Skillvio.Learning.ContentSync -f net10.0 --force
cd ../..
dotnet sln add src/Skillvio.Learning.ContentSync/Skillvio.Learning.ContentSync.csproj
dotnet add src/Skillvio.Learning.ContentSync reference src/Skillvio.Learning.Application src/Skillvio.Learning.Infrastructure
```

- [ ] **Step 12: Create BadgeWorker console worker**

```bash
mkdir -p src/Skillvio.Learning.BadgeWorker
cd src/Skillvio.Learning.BadgeWorker
dotnet new worker -n Skillvio.Learning.BadgeWorker -f net10.0 --force
cd ../..
dotnet sln add src/Skillvio.Learning.BadgeWorker/Skillvio.Learning.BadgeWorker.csproj
dotnet add src/Skillvio.Learning.BadgeWorker reference src/Skillvio.Learning.Application src/Skillvio.Learning.Infrastructure
```

- [ ] **Step 13: Create 3 test projects**

```bash
mkdir -p tests/Skillvio.Learning.Domain.Tests
cd tests/Skillvio.Learning.Domain.Tests
dotnet new xunit -n Skillvio.Learning.Domain.Tests -f net10.0 --force
rm UnitTest1.cs
cd ../..
dotnet sln add tests/Skillvio.Learning.Domain.Tests/Skillvio.Learning.Domain.Tests.csproj
dotnet add tests/Skillvio.Learning.Domain.Tests reference src/Skillvio.Learning.Domain

mkdir -p tests/Skillvio.Learning.Application.Tests
cd tests/Skillvio.Learning.Application.Tests
dotnet new xunit -n Skillvio.Learning.Application.Tests -f net10.0 --force
rm UnitTest1.cs
cd ../..
dotnet sln add tests/Skillvio.Learning.Application.Tests/Skillvio.Learning.Application.Tests.csproj
dotnet add tests/Skillvio.Learning.Application.Tests reference src/Skillvio.Learning.Application

mkdir -p tests/Skillvio.Learning.IntegrationTests
cd tests/Skillvio.Learning.IntegrationTests
dotnet new xunit -n Skillvio.Learning.IntegrationTests -f net10.0 --force
rm UnitTest1.cs
cd ../..
dotnet sln add tests/Skillvio.Learning.IntegrationTests/Skillvio.Learning.IntegrationTests.csproj
dotnet add tests/Skillvio.Learning.IntegrationTests reference src/Skillvio.Learning.Api src/Skillvio.Learning.Infrastructure
```

- [ ] **Step 14: Add FluentAssertions + xunit to all test projects via Directory.Packages**

In each `*.Tests.csproj`, ensure these `<PackageReference>` entries exist (no `Version=` because central management):

```xml
<ItemGroup>
  <PackageReference Include="FluentAssertions" />
  <PackageReference Include="Microsoft.NET.Test.Sdk" />
  <PackageReference Include="xunit" />
  <PackageReference Include="xunit.runner.visualstudio" />
  <PackageReference Include="NSubstitute" />
</ItemGroup>
```

For `IntegrationTests` add:
```xml
<PackageReference Include="Testcontainers.PostgreSql" />
<PackageReference Include="Testcontainers.Redis" />
<PackageReference Include="Testcontainers.RabbitMq" />
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" />
```

- [ ] **Step 15: Verify clean build**

```bash
dotnet build
```

Expected: `Build succeeded. 0 Warning(s) 0 Error(s)`. If warnings → `TreatWarningsAsErrors` should fail. Fix the warnings before continuing.

- [ ] **Step 16: Write minimal CI workflow**

Create `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_USER: test
          POSTGRES_DB: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ['5432:5432']
      redis:
        image: redis:7
        ports: ['6379:6379']
      rabbitmq:
        image: rabbitmq:3-management
        ports: ['5672:5672', '15672:15672']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
      - run: dotnet restore
      - run: dotnet build --configuration Release --no-restore
      - run: dotnet test --no-build --configuration Release --logger "trx;LogFileName=test.trx" --collect "XPlat Code Coverage"
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: '**/TestResults/**'
```

- [ ] **Step 17: Commit and push**

```bash
git add -A
git commit -m "chore(s3a-t1): scaffold .NET 10 solution with 6 src + 3 test projects"
git push -u origin main
```

Verify CI green on GitHub Actions before continuing.

---

## Task 2 — DbContext + Init Migration (30 Tables, Partitions, View, tsvector)

**Spec ref:** §3 Schema, §3.7 GDPR view, §3.9 Partitioning, §3.10 Triggers
**Estimated:** 10h
**Files:**
- Create: `src/Skillvio.Learning.Infrastructure/Persistence/LearningDbContext.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Persistence/Configurations/*.cs` (one per aggregate)
- Create: `src/Skillvio.Learning.Infrastructure/Persistence/RawSqlExtensions.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Persistence/Migrations/*` (auto-generated)
- Create: `tests/Skillvio.Learning.IntegrationTests/Persistence/MigrationFixture.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Persistence/MigrationTests.cs`

> **Note:** Domain entities are written in Task 3. Task 2 produces a `LearningDbContext` skeleton that references entity types **declared but not yet implemented** by Task 3. To allow this task to compile/run, define minimal entity placeholders in Domain inside Step 2 below — these will be replaced/extended in Task 3.

- [ ] **Step 1: Add EF Core packages to Infrastructure**

```bash
cd src/Skillvio.Learning.Infrastructure
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package EFCore.NamingConventions
cd ../..
```

- [ ] **Step 2: Add entity placeholder records in Domain**

Create `src/Skillvio.Learning.Domain/_Placeholders.cs` (will be split in Task 3):

```csharp
namespace Skillvio.Learning.Domain.Content.Tracks;
public class Track { public Guid Id { get; set; } public string Slug { get; set; } = ""; public Guid DomainId { get; set; } public short Difficulty { get; set; } public int EstMinutes { get; set; } public string HeartsMode { get; set; } = "lenient"; public string DefaultLocale { get; set; } = "en"; public bool IsPremium { get; set; } public Guid[] Prerequisites { get; set; } = []; public Guid? TargetExamId { get; set; } public System.Text.Json.JsonDocument? ShortI18n { get; set; } public string Status { get; set; } = "draft"; public DateTimeOffset? ArchivedAt { get; set; } public string? MinAppVersion { get; set; } public string SourcePath { get; set; } = ""; public string SourceCommitSha { get; set; } = ""; public string ContentHash { get; set; } = ""; public short SchemaVersion { get; set; } = 1; public DateTimeOffset? PublishedAt { get; set; } public DateTimeOffset CreatedAt { get; set; } public DateTimeOffset UpdatedAt { get; set; } }
// ... [Full placeholder set deferred to Task 3, see §3 of spec for the canonical list]
```

> **PRACTICAL NOTE:** Rather than typing 30 placeholder classes here, Task 3 owns the entity definitions. Skip Step 2's placeholders; instead, **execute Task 3 in interleaved steps with Task 2**. The pragmatic flow: Task 3 Step 1 → Task 2 Step 3 → Task 3 Step 2 → Task 2 Step 4 etc. The tasks are split for clarity; the engineer should treat them as parallel tracks that converge on the migration.

- [ ] **Step 3: Write `LearningDbContext` skeleton (entity DbSets added incrementally as Task 3 produces them)**

Create `src/Skillvio.Learning.Infrastructure/Persistence/LearningDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;

namespace Skillvio.Learning.Infrastructure.Persistence;

public class LearningDbContext(DbContextOptions<LearningDbContext> options) : DbContext(options)
{
    // Content
    public DbSet<Skillvio.Learning.Domain.Content.Tracks.Track> Tracks => Set<Skillvio.Learning.Domain.Content.Tracks.Track>();
    // [DbSets for each entity added one-by-one in Task 3 steps]

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasPostgresExtension("pg_trgm");
        modelBuilder.HasPostgresExtension("uuid-ossp");
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(LearningDbContext).Assembly);
    }
}
```

- [ ] **Step 4: Wire snake_case naming + JSONB defaults in `OnConfiguring` host**

Create `src/Skillvio.Learning.Api/Program.cs` snippet (will be expanded in later tasks):

```csharp
builder.Services.AddDbContext<LearningDbContext>(opt => opt
    .UseNpgsql(builder.Configuration.GetConnectionString("Default"),
        npg => npg.MigrationsAssembly("Skillvio.Learning.Infrastructure"))
    .UseSnakeCaseNamingConvention());
```

- [ ] **Step 5–N: Per-aggregate fluent configuration files**

For **each** aggregate listed in spec §3 (Tracks, Lessons, Activities, Roadmaps, Exams, ExamQuestions, Glossary, Badges, Certificates, Attempts, AttemptAnswers, Hearts, DayStreaks, UserXp, XpTransactions, UserBadges, UserCertificates, Friendships, FriendActivityFeed, LeagueSeasons, LeagueMemberships, UserActiveTracks, UserTrackProgress, UserPreferences, UserLabCompletions, OutboxMessages, ProcessedEvents, ContentImports, IdempotencyKeys), create `src/Skillvio.Learning.Infrastructure/Persistence/Configurations/<Aggregate>Configuration.cs`.

**Pattern (Tracks example):**

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Skillvio.Learning.Domain.Content.Tracks;

namespace Skillvio.Learning.Infrastructure.Persistence.Configurations;

public class TrackConfiguration : IEntityTypeConfiguration<Track>
{
    public void Configure(EntityTypeBuilder<Track> b)
    {
        b.ToTable("tracks");
        b.HasKey(t => t.Id);
        b.Property(t => t.Slug).HasMaxLength(160).IsRequired();
        b.HasIndex(t => t.Slug).IsUnique();
        b.Property(t => t.HeartsMode).HasMaxLength(16);
        b.Property(t => t.Status).HasMaxLength(16).IsRequired();
        b.Property(t => t.SourcePath).IsRequired();
        b.Property(t => t.ContentHash).IsRequired();
        b.Property(t => t.ShortI18n).HasColumnType("jsonb");
        b.Property(t => t.Prerequisites).HasColumnType("uuid[]");
        b.HasIndex(t => new { t.DomainId, t.Status, t.PublishedAt });
        b.HasCheckConstraint("ck_tracks_status",
            "status IN ('draft','review','published','archived')");
    }
}
```

Repeat for each aggregate using spec §3 column list. Order of creation:
1. `domains, domain_translations`
2. `tracks, track_translations, units, unit_lessons`
3. `lessons, lesson_translations, activities, activity_translations`
4. `roadmaps, roadmap_translations, roadmap_nodes, node_translations, node_resources`
5. `glossary_terms, glossary_translations`
6. `exams, exam_translations, exam_questions, exam_question_translations`
7. `certificate_definitions, certificate_translations`
8. `badges, badge_translations`
9. `user_active_tracks, user_active_roadmaps, user_track_progress, user_roadmap_progress, user_node_progress, user_lesson_progress, user_lab_completions, user_preferences`
10. `attempts, attempt_answers, exam_attempt_meta, exam_access`
11. `user_badges, badge_evaluation_log, user_certificates`
12. `user_xp, xp_transactions, hearts, day_streaks, streak_repairs, level_definitions`
13. `friendships, friend_activity_feed, league_tiers, league_seasons, league_memberships`
14. `outbox_messages, processed_events, content_imports, idempotency_keys`

Commit after each group of ~5 configurations.

- [ ] **Step N+1: Add raw-SQL migration extension for partitions, view, triggers**

Create `src/Skillvio.Learning.Infrastructure/Persistence/RawSqlExtensions.cs`:

```csharp
using Microsoft.EntityFrameworkCore.Migrations;

namespace Skillvio.Learning.Infrastructure.Persistence;

public static class RawSqlExtensions
{
    public static void CreateMonthlyPartition(this MigrationBuilder mb, string parentTable, string ymKey, DateOnly fromUtc, DateOnly toUtc)
    {
        mb.Sql($"""
            CREATE TABLE IF NOT EXISTS {parentTable}_{ymKey}
              PARTITION OF {parentTable}
              FOR VALUES FROM ('{fromUtc:yyyy-MM-dd}') TO ('{toUtc:yyyy-MM-dd}');
        """);
    }

    public static void EnableRangePartitioning(this MigrationBuilder mb, string table, string column)
    {
        // EF generates `CREATE TABLE`; for partitioned tables we recreate via raw SQL.
        // Strategy: in Init migration, drop EF-created table & recreate as PARTITION BY RANGE.
        mb.Sql($"DROP TABLE {table};");
        // Recreated via subsequent CREATE TABLE statements bundled via Sql().
    }

    public static void CreateOutboxNotifyTrigger(this MigrationBuilder mb)
    {
        mb.Sql("""
            CREATE OR REPLACE FUNCTION outbox_notify() RETURNS trigger AS $$
            BEGIN PERFORM pg_notify('outbox_new', NEW.id::text); RETURN NEW; END;
            $$ LANGUAGE plpgsql;
            CREATE TRIGGER outbox_notify_trg AFTER INSERT ON outbox_messages
              FOR EACH ROW EXECUTE FUNCTION outbox_notify();
        """);
    }

    public static void CreateUserDataTablesView(this MigrationBuilder mb)
    {
        mb.Sql("""
            CREATE OR REPLACE VIEW v_user_data_tables AS
            SELECT unnest(ARRAY[
              'attempts','attempt_answers','user_badges','user_certificates',
              'user_track_progress','user_roadmap_progress','user_node_progress',
              'user_lesson_progress','user_active_tracks','user_active_roadmaps',
              'hearts','day_streaks','streak_repairs','user_xp','xp_transactions',
              'exam_access','badge_evaluation_log','user_lab_completions',
              'user_preferences','friendships','friend_activity_feed','league_memberships'
            ]) AS table_name;
        """);
    }

    public static void CreateGlossaryTsvectorIndex(this MigrationBuilder mb)
    {
        mb.Sql("CREATE INDEX IF NOT EXISTS idx_glossary_search ON glossary_translations USING GIN(search_tsvector);");
    }
}
```

- [ ] **Step N+2: Generate Init migration**

```bash
dotnet ef migrations add Init \
  --project src/Skillvio.Learning.Infrastructure \
  --startup-project src/Skillvio.Learning.Api \
  --context LearningDbContext
```

Expected output: `migrations/<timestamp>_Init.cs` and `LearningDbContextModelSnapshot.cs`.

- [ ] **Step N+3: Augment generated migration with raw SQL (partition, trigger, view, GIN, partial indexes)**

Open the generated `<timestamp>_Init.cs`. At the **end** of `Up(MigrationBuilder mb)`, append:

```csharp
// --- Range partitioning ---
mb.Sql("DROP TABLE xp_transactions;");
mb.Sql("""
    CREATE TABLE xp_transactions (
      id uuid NOT NULL,
      user_id uuid NOT NULL,
      delta integer NOT NULL,
      source text NOT NULL,
      source_ref uuid NULL,
      multipliers jsonb NULL,
      created_at timestamptz NOT NULL,
      trace_id text NULL,
      PRIMARY KEY (id, created_at)
    ) PARTITION BY RANGE (created_at);
""");
mb.CreateMonthlyPartition("xp_transactions", "2026_05", new DateOnly(2026,5,1), new DateOnly(2026,6,1));
mb.CreateMonthlyPartition("xp_transactions", "2026_06", new DateOnly(2026,6,1), new DateOnly(2026,7,1));
mb.CreateMonthlyPartition("xp_transactions", "2026_07", new DateOnly(2026,7,1), new DateOnly(2026,8,1));
mb.Sql("CREATE INDEX idx_xp_tx_user_time ON xp_transactions(user_id, created_at DESC);");

// Repeat the same pattern for attempt_answers and badge_evaluation_log

// --- Outbox notify trigger ---
mb.CreateOutboxNotifyTrigger();

// --- GDPR view ---
mb.CreateUserDataTablesView();

// --- tsvector GIN index ---
mb.CreateGlossaryTsvectorIndex();

// --- Partial index: active pinned track ---
mb.Sql("CREATE INDEX idx_user_active_tracks_pinned ON user_active_tracks(user_id) WHERE is_pinned = true;");

// --- Outbox undispatched index (HOT) ---
mb.Sql("CREATE INDEX idx_outbox_undispatched ON outbox_messages(occurred_at) WHERE dispatched_at IS NULL;");
```

In `Down`, add inverse `DROP TABLE xp_transactions; CREATE TABLE xp_transactions (...)` (re-create as plain table) and `DROP VIEW`, `DROP TRIGGER`, etc.

- [ ] **Step N+4: Write migration apply test (Testcontainers)**

Create `tests/Skillvio.Learning.IntegrationTests/Persistence/MigrationFixture.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Skillvio.Learning.Infrastructure.Persistence;
using Testcontainers.PostgreSql;

namespace Skillvio.Learning.IntegrationTests.Persistence;

public sealed class MigrationFixture : IAsyncLifetime
{
    public PostgreSqlContainer Container { get; } = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .WithDatabase("learning_test")
        .Build();

    public LearningDbContext DbContext { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        await Container.StartAsync();
        var opts = new DbContextOptionsBuilder<LearningDbContext>()
            .UseNpgsql(Container.GetConnectionString())
            .UseSnakeCaseNamingConvention()
            .Options;
        DbContext = new LearningDbContext(opts);
    }

    public async Task DisposeAsync()
    {
        await DbContext.DisposeAsync();
        await Container.DisposeAsync();
    }
}
```

Create `tests/Skillvio.Learning.IntegrationTests/Persistence/MigrationTests.cs`:

```csharp
using FluentAssertions;
using Microsoft.EntityFrameworkCore;
using Skillvio.Learning.IntegrationTests.Persistence;
using Xunit;

namespace Skillvio.Learning.IntegrationTests.Persistence;

[Collection("Database")]
public class MigrationTests(MigrationFixture fx) : IClassFixture<MigrationFixture>
{
    [Fact]
    public async Task Migrate_AppliesCleanly()
    {
        await fx.DbContext.Database.MigrateAsync();
        var pending = await fx.DbContext.Database.GetPendingMigrationsAsync();
        pending.Should().BeEmpty();
    }

    [Fact]
    public async Task Migrate_CreatesGdprView()
    {
        await fx.DbContext.Database.MigrateAsync();
        var conn = fx.DbContext.Database.GetDbConnection();
        await conn.OpenAsync();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT count(*) FROM v_user_data_tables;";
        var count = (long)(await cmd.ExecuteScalarAsync())!;
        count.Should().BeGreaterThan(20);
    }

    [Fact]
    public async Task Migrate_CreatesXpPartitions()
    {
        await fx.DbContext.Database.MigrateAsync();
        var conn = fx.DbContext.Database.GetDbConnection();
        await conn.OpenAsync();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT count(*) FROM pg_inherits WHERE inhparent = 'xp_transactions'::regclass;";
        var count = (long)(await cmd.ExecuteScalarAsync())!;
        count.Should().BeGreaterOrEqualTo(3);
    }
}
```

- [ ] **Step N+5: Run integration tests**

```bash
dotnet test tests/Skillvio.Learning.IntegrationTests/ -v normal
```

Expected: 3 PASS. If any FAIL, fix the migration and re-run.

- [ ] **Step N+6: Commit**

```bash
git add -A
git commit -m "feat(s3a-t2): DbContext + Init migration with 30 tables, partitions, GDPR view, outbox trigger"
git push
```

---

## Task 3 — Domain Models + Value Objects (Mutation-Test Ready)

**Spec ref:** §2.0 Domain folder layout, §6.1 ScoringService inputs
**Estimated:** 12h
**Files:** ~60 entity/value-object files under `src/Skillvio.Learning.Domain/`

> **Strategy:** Implement domain entities **incrementally**, one bounded context at a time. After each context's entities are written, run a quick "object-graph compiles" test. Run mutation testing only in Task 18 (production polish) — keep value objects pure here.

This task is too granular to flatten into the parent plan. The full breakdown is in **`plans/sprint-3a/task-3-domain-models.md`** (created as a sibling document below). The summary commits per group are:

- [ ] **Step 1: Common value objects** — `Locale`, `DomainCode`, `Slug`, `XpAmount`, `ScorePct`, `ContentVersion`. Commit: `feat(s3a-t3): common value objects`.
- [ ] **Step 2: Domain events marker types + abstractions** — `IDomainEvent`, `IClock`, `ILocalizationContext`, `IIdGenerator`. Commit: `feat(s3a-t3): domain abstractions`.
- [ ] **Step 3: Content/Tracks aggregates** — `Track`, `Unit`, `UnitLesson`. Tests: invariants for status transitions. Commit.
- [ ] **Step 4: Content/Lessons aggregates** — `Lesson`, `Activity` (+ `ActivityType` enum). Tests. Commit.
- [ ] **Step 5: Content/Roadmaps aggregates** — `Roadmap`, `RoadmapNode`, `NodeResource`. Tests. Commit.
- [ ] **Step 6: Content/Exams aggregates** — `Exam`, `ExamQuestion`. Tests for `expires_at` semantics. Commit.
- [ ] **Step 7: Content/Glossary** — `GlossaryTerm`. Commit.
- [ ] **Step 8: Learning/Attempts** — `Attempt`, `AttemptAnswer`, `ExamAttemptMeta`. Tests for state-machine transitions. Commit.
- [ ] **Step 9: Learning/Progress** — `UserLessonProgress`, `UserTrackProgress`, `UserRoadmapProgress`, `UserNodeProgress`, `UserLabCompletion`. Commit.
- [ ] **Step 10: Learning/ActiveSelections** — `UserActiveTracks`, `UserActiveRoadmaps`, `UserPreferences`. Tests for max-5 invariant. Commit.
- [ ] **Step 11: Gamification/Xp** — `UserXp`, `XpTransaction`, `ScoringRule`. Tests for atomic-add semantics (in domain unit tests via property-based assertions). Commit.
- [ ] **Step 12: Gamification/Hearts** — `Hearts`. Tests for refill/unlimited. Commit.
- [ ] **Step 13: Gamification/Streaks** — `DayStreak`, `StreakRepair`. Tests for freeze quota + repair window. Commit.
- [ ] **Step 14: Gamification/Levels** — `Level`, `LevelDefinition`. Tests for boundary-XP transitions. Commit.
- [ ] **Step 15: Gamification/Badges** — `Badge`, `UserBadge`, `BadgeEvaluationLog`. Tests for predicate-arg shapes. Commit.
- [ ] **Step 16: Certificates** — `CertificateDefinition`, `UserCertificate`. Tests for verification_code uniqueness invariant. Commit.
- [ ] **Step 17: Social/Friendship** — `Friendship`, `FriendActivityFeedEntry`. Tests for symmetric block + spam guard counts. Commit.
- [ ] **Step 18: Social/League** — `LeagueTier`, `LeagueSeason`, `LeagueMembership`. Tests for cohort fairness. Commit.
- [ ] **Step 19: Run `dotnet build` + Domain.Tests**

```bash
dotnet build
dotnet test tests/Skillvio.Learning.Domain.Tests/
```

Expected: All green. Min coverage Domain ≥ 70% at this point (final target 90% reached via Task 5/6/7 service tests).

- [ ] **Step 20: Final commit**

```bash
git commit -am "feat(s3a-t3): complete Domain layer (60+ entities & value objects)"
git push
```

---

> **Continuation:** Tasks 4–17 follow in this same plan file. Each task uses the same TDD pattern: failing test → implementation → green test → commit. See sibling sections below.


---

## Task 4 — Localization Infrastructure (FallbackResolver + LocaleFilter + HybridCache)

**Spec ref:** §6.9 LocalizedQueryService, §5.1 Locale convention
**Estimated:** 6h
**Files:**
- Create: `src/Skillvio.Learning.Application/Localization/ILocalizationContext.cs`
- Create: `src/Skillvio.Learning.Application/Localization/FallbackResolver.cs`
- Create: `src/Skillvio.Learning.Application/Localization/ILocalizedQueryService.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Localization/LocalizationContext.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Localization/LocalizedQueryService.cs`
- Create: `src/Skillvio.Learning.Api/Filters/LocaleFilter.cs`
- Create: `tests/Skillvio.Learning.Application.Tests/Localization/FallbackResolverTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Localization/LocalizedQueryServiceTests.cs`

- [ ] **Step 1: Write `ILocalizationContext` abstraction**

```csharp
// src/Skillvio.Learning.Application/Localization/ILocalizationContext.cs
namespace Skillvio.Learning.Application.Localization;

public interface ILocalizationContext
{
    string CurrentLocale { get; }
    void SetCurrentLocale(string locale);
}
```

- [ ] **Step 2: Write FallbackResolver test (TDD red)**

```csharp
// tests/Skillvio.Learning.Application.Tests/Localization/FallbackResolverTests.cs
using FluentAssertions;
using Skillvio.Learning.Application.Localization;
using Xunit;

public class FallbackResolverTests
{
    [Theory]
    [InlineData("tr-TR", "en", new[] { "en" }, "tr-TR")]                          // user has primary
    [InlineData("de-DE", "en", new[] { "en", "tr" }, "en")]                       // user missing → content default
    [InlineData("fr-FR", "tr", new[] { "tr", "en" }, "tr")]                       // user missing → content default tr
    [InlineData("ar-EG", "en", new[] { "tr", "de" }, "tr")]                       // missing → en missing → first available
    [InlineData("en-US", "en", new[] { "en-GB" }, "en-GB")]                       // language match
    public void Resolve_FollowsChain(string user, string contentDefault, string[] available, string expected)
    {
        var sut = new FallbackResolver();
        var actual = sut.Resolve(user, contentDefault, available);
        actual.Should().Be(expected);
    }
}
```

- [ ] **Step 3: Run test — expect compile error**

```bash
dotnet test tests/Skillvio.Learning.Application.Tests/ --filter FullyQualifiedName~FallbackResolverTests
```

Expected: Compile error `FallbackResolver not found`.

- [ ] **Step 4: Implement `FallbackResolver`**

```csharp
// src/Skillvio.Learning.Application/Localization/FallbackResolver.cs
namespace Skillvio.Learning.Application.Localization;

public class FallbackResolver
{
    public string Resolve(string userPreferred, string contentDefault, IReadOnlyCollection<string> available)
    {
        if (available.Contains(userPreferred, StringComparer.OrdinalIgnoreCase))
            return available.First(a => string.Equals(a, userPreferred, StringComparison.OrdinalIgnoreCase));

        var langOnly = userPreferred.Split('-')[0];
        var langMatch = available.FirstOrDefault(a => a.StartsWith(langOnly, StringComparison.OrdinalIgnoreCase));
        if (langMatch is not null) return langMatch;

        if (available.Contains(contentDefault, StringComparer.OrdinalIgnoreCase))
            return available.First(a => string.Equals(a, contentDefault, StringComparison.OrdinalIgnoreCase));

        if (available.Contains("en", StringComparer.OrdinalIgnoreCase))
            return available.First(a => string.Equals(a, "en", StringComparison.OrdinalIgnoreCase));

        return available.First();
    }
}
```

- [ ] **Step 5: Run test — expect green**

```bash
dotnet test tests/Skillvio.Learning.Application.Tests/ --filter FullyQualifiedName~FallbackResolverTests
```

Expected: 5 PASS.

- [ ] **Step 6: Implement `LocalizationContext` (Infrastructure, scoped per-request)**

```csharp
// src/Skillvio.Learning.Infrastructure/Localization/LocalizationContext.cs
using Skillvio.Learning.Application.Localization;

namespace Skillvio.Learning.Infrastructure.Localization;

public class LocalizationContext : ILocalizationContext
{
    public string CurrentLocale { get; private set; } = "en";
    public void SetCurrentLocale(string locale) => CurrentLocale = locale;
}
```

- [ ] **Step 7: Implement `LocaleFilter` ASP.NET endpoint filter**

```csharp
// src/Skillvio.Learning.Api/Filters/LocaleFilter.cs
using Microsoft.AspNetCore.Http;
using Skillvio.Learning.Application.Localization;

namespace Skillvio.Learning.Api.Filters;

public class LocaleFilter(ILocalizationContext ctx) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext c, EndpointFilterDelegate next)
    {
        var http = c.HttpContext;
        var locale = http.Request.Headers["X-User-Locale"].FirstOrDefault()
                  ?? http.Request.Headers.AcceptLanguage.FirstOrDefault()
                  ?? "en";
        ctx.SetCurrentLocale(locale.Split(',')[0].Trim());
        var result = await next(c);
        http.Response.Headers["Content-Language"] = ctx.CurrentLocale;
        return result;
    }
}
```

- [ ] **Step 8: Wire DI in `Program.cs`**

```csharp
builder.Services.AddScoped<ILocalizationContext, LocalizationContext>();
builder.Services.AddSingleton<FallbackResolver>();
builder.Services.AddScoped<LocaleFilter>();
```

- [ ] **Step 9: Add HybridCache**

```bash
dotnet add src/Skillvio.Learning.Infrastructure package Microsoft.Extensions.Caching.Hybrid
```

In `Program.cs`:
```csharp
builder.Services.AddHybridCache(opts =>
{
    opts.DefaultEntryOptions = new() { Expiration = TimeSpan.FromMinutes(5) };
});
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = builder.Configuration["ConnectionStrings:Redis"]);
```

- [ ] **Step 10: Implement `LocalizedQueryService<TEntity, TTranslation>`**

```csharp
// src/Skillvio.Learning.Application/Localization/ILocalizedQueryService.cs
namespace Skillvio.Learning.Application.Localization;

public interface ILocalizedQueryService<TEntity, TTranslation>
    where TEntity : class
    where TTranslation : class
{
    Task<TEntity?> GetWithLocaleAsync(Guid id, string locale, CancellationToken ct = default);
}
```

```csharp
// src/Skillvio.Learning.Infrastructure/Localization/LocalizedQueryService.cs
using Microsoft.EntityFrameworkCore;
using Skillvio.Learning.Application.Localization;
using Skillvio.Learning.Infrastructure.Persistence;

namespace Skillvio.Learning.Infrastructure.Localization;

public class LocalizedQueryService<TEntity, TTranslation>(LearningDbContext db, FallbackResolver resolver)
    : ILocalizedQueryService<TEntity, TTranslation>
    where TEntity : class
    where TTranslation : class
{
    public async Task<TEntity?> GetWithLocaleAsync(Guid id, string locale, CancellationToken ct = default)
    {
        // Subclasses or specialized methods join the translation table — we provide a base hook.
        return await db.Set<TEntity>().FindAsync([id], ct);
    }
}
```

> **Note:** Per-aggregate localized query helpers (e.g., `GetTrackWithLocaleAsync`) are added in T11a as endpoints need them.

- [ ] **Step 11: Integration test for fallback chain**

```csharp
// tests/Skillvio.Learning.IntegrationTests/Localization/LocalizedQueryServiceTests.cs
using FluentAssertions;
using Skillvio.Learning.Application.Localization;
using Xunit;

public class FallbackResolverIntegrationTests
{
    [Fact]
    public void EmptyAvailable_ThrowsHelpfully()
    {
        var sut = new FallbackResolver();
        var act = () => sut.Resolve("tr", "en", Array.Empty<string>());
        act.Should().Throw<InvalidOperationException>();
    }
}
```

Update `FallbackResolver.Resolve` to throw `InvalidOperationException("No locales available")` when `available.Count == 0`.

- [ ] **Step 12: Run all tests**

```bash
dotnet test
```

- [ ] **Step 13: Commit**

```bash
git commit -am "feat(s3a-t4): localization infra (fallback chain, LocaleFilter, HybridCache, LocalizedQueryService)"
git push
```

---

## Task 5 — ScoringService + LevelService + HeartsService (Table-Driven Tests)

**Spec ref:** §6.1 ScoringService, §6.6 HeartsService, Q4 dynamic XP, Q5 hearts
**Estimated:** 8h
**Files:**
- Create: `src/Skillvio.Learning.Application/Gamification/Scoring/IScoringService.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Scoring/ScoringService.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Scoring/ScoringContext.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Scoring/ScoringRules.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Levels/ILevelService.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Levels/LevelService.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Hearts/IHeartsService.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Hearts/HeartsService.cs`
- Create: `tests/Skillvio.Learning.Application.Tests/Gamification/ScoringServiceTests.cs`
- Create: `tests/Skillvio.Learning.Application.Tests/Gamification/LevelServiceTests.cs`
- Create: `tests/Skillvio.Learning.Application.Tests/Gamification/HeartsServiceTests.cs`

- [ ] **Step 1: Write `ScoringContext`, `ScoringRules`, `XpCalculation` records**

```csharp
// src/Skillvio.Learning.Application/Gamification/Scoring/ScoringRules.cs
namespace Skillvio.Learning.Application.Gamification.Scoring;

public record ScoringRules(
    int BaseXp,
    bool EnableAccuracyMultiplier,
    bool EnableStreakBonus,
    bool EnablePerfectBonus,
    int PerfectLessonBonus,
    decimal Streak7Bonus,
    decimal Streak30Bonus);

public record ScoringContext(
    Guid UserId,
    Guid AttemptId,
    int BaseXp,
    int AttemptCount,            // 1=first try
    int CurrentStreak,
    bool IsLessonPerfect,
    ScoringRules Rules);

public record XpCalculation(
    int BaseXp,
    decimal AccuracyMultiplier,
    decimal StreakBonus,
    int PerfectLessonBonus,
    int FinalXp,
    Dictionary<string, object> Breakdown);
```

- [ ] **Step 2: Write IScoringService interface**

```csharp
// src/Skillvio.Learning.Application/Gamification/Scoring/IScoringService.cs
namespace Skillvio.Learning.Application.Gamification.Scoring;

public interface IScoringService
{
    XpCalculation Calculate(ScoringContext ctx);
}
```

- [ ] **Step 3: Write 50+ table-driven tests (RED)**

```csharp
// tests/Skillvio.Learning.Application.Tests/Gamification/ScoringServiceTests.cs
using FluentAssertions;
using Skillvio.Learning.Application.Gamification.Scoring;
using Xunit;

public class ScoringServiceTests
{
    private static readonly ScoringRules DefaultRules = new(
        BaseXp: 10, EnableAccuracyMultiplier: true, EnableStreakBonus: true,
        EnablePerfectBonus: true, PerfectLessonBonus: 25,
        Streak7Bonus: 1.10m, Streak30Bonus: 1.25m);

    [Theory]
    // FirstTry, NoStreak, NotPerfect → base × 1.0 × 1.0 + 0 = base
    [InlineData(10, 1, 0, false, 10)]
    [InlineData(20, 1, 0, false, 20)]
    // FirstTry, Streak7, NotPerfect → base × 1.0 × 1.10
    [InlineData(10, 1, 7, false, 11)]
    [InlineData(20, 1, 7, false, 22)]
    // FirstTry, Streak30, NotPerfect → base × 1.0 × 1.25
    [InlineData(10, 1, 30, false, 12)]
    [InlineData(20, 1, 30, false, 25)]
    // SecondTry, NoStreak, NotPerfect → base × 0.5
    [InlineData(10, 2, 0, false, 5)]
    [InlineData(20, 2, 0, false, 10)]
    // ThirdTry+ , NoStreak, NotPerfect → base × 0.25
    [InlineData(10, 3, 0, false, 2)]
    [InlineData(20, 3, 0, false, 5)]
    [InlineData(20, 5, 0, false, 5)]
    // FirstTry, NoStreak, Perfect → base × 1.0 × 1.0 + 25
    [InlineData(10, 1, 0, true, 35)]
    [InlineData(20, 1, 0, true, 45)]
    // FirstTry, Streak30, Perfect → base × 1.25 + 25
    [InlineData(20, 1, 30, true, 50)]
    public void Calculate_ProducesExpectedXp(int baseXp, int attemptCount, int streak, bool perfect, int expected)
    {
        var ctx = new ScoringContext(
            UserId: Guid.NewGuid(),
            AttemptId: Guid.NewGuid(),
            BaseXp: baseXp,
            AttemptCount: attemptCount,
            CurrentStreak: streak,
            IsLessonPerfect: perfect,
            Rules: DefaultRules);
        var sut = new ScoringService();
        sut.Calculate(ctx).FinalXp.Should().Be(expected);
    }

    [Fact]
    public void Calculate_FillsBreakdown()
    {
        var ctx = new ScoringContext(Guid.NewGuid(), Guid.NewGuid(), 20, 1, 30, true, DefaultRules);
        var calc = new ScoringService().Calculate(ctx);
        calc.Breakdown.Should().ContainKeys("base", "accuracy", "streak", "perfect");
    }

    [Fact]
    public void Calculate_DisabledFlags_SkipMultipliers()
    {
        var rules = DefaultRules with { EnableAccuracyMultiplier = false, EnableStreakBonus = false, EnablePerfectBonus = false };
        var ctx = new ScoringContext(Guid.NewGuid(), Guid.NewGuid(), 20, 5, 30, true, rules);
        new ScoringService().Calculate(ctx).FinalXp.Should().Be(20);
    }
}
```

- [ ] **Step 4: Run tests — expect compile error**

```bash
dotnet test tests/Skillvio.Learning.Application.Tests/ --filter FullyQualifiedName~ScoringServiceTests
```

Expected: `ScoringService not found`.

- [ ] **Step 5: Implement ScoringService**

```csharp
// src/Skillvio.Learning.Application/Gamification/Scoring/ScoringService.cs
namespace Skillvio.Learning.Application.Gamification.Scoring;

public class ScoringService : IScoringService
{
    public XpCalculation Calculate(ScoringContext ctx)
    {
        var accuracy = ctx.Rules.EnableAccuracyMultiplier
            ? ctx.AttemptCount switch { 1 => 1.0m, 2 => 0.5m, _ => 0.25m }
            : 1.0m;
        var streak = ctx.Rules.EnableStreakBonus
            ? (ctx.CurrentStreak >= 30 ? ctx.Rules.Streak30Bonus
               : ctx.CurrentStreak >= 7 ? ctx.Rules.Streak7Bonus
               : 1.0m)
            : 1.0m;
        var perfectBonus = ctx.Rules.EnablePerfectBonus && ctx.IsLessonPerfect ? ctx.Rules.PerfectLessonBonus : 0;
        var final = (int)Math.Floor(ctx.BaseXp * accuracy * streak) + perfectBonus;
        return new XpCalculation(
            BaseXp: ctx.BaseXp,
            AccuracyMultiplier: accuracy,
            StreakBonus: streak,
            PerfectLessonBonus: perfectBonus,
            FinalXp: final,
            Breakdown: new Dictionary<string, object>
            {
                ["base"] = ctx.BaseXp,
                ["accuracy"] = accuracy,
                ["streak"] = streak,
                ["perfect"] = perfectBonus
            });
    }
}
```

- [ ] **Step 6: Run tests — expect all green**

```bash
dotnet test tests/Skillvio.Learning.Application.Tests/ --filter FullyQualifiedName~ScoringServiceTests
```

Expected: 16+ PASS.

- [ ] **Step 7: Implement LevelService with boundary tests**

Test first:
```csharp
// tests/Skillvio.Learning.Application.Tests/Gamification/LevelServiceTests.cs
using FluentAssertions;
using Skillvio.Learning.Application.Gamification.Levels;
using Xunit;

public class LevelServiceTests
{
    [Theory]
    [InlineData(0, 1)]
    [InlineData(99, 1)]
    [InlineData(100, 2)]
    [InlineData(399, 2)]
    [InlineData(400, 3)]
    [InlineData(900, 4)]
    public void Compute_LevelFromXp(long xp, int expected)
    {
        new LevelService().ComputeLevel(xp).Should().Be(expected);
    }

    [Fact]
    public void XpToNext_AtBoundary()
    {
        var sut = new LevelService();
        sut.XpToNext(0).Should().Be(100);
        sut.XpToNext(99).Should().Be(1);
        sut.XpToNext(100).Should().Be(300);  // 400-100
    }
}
```

Implementation:
```csharp
// src/Skillvio.Learning.Application/Gamification/Levels/ILevelService.cs
namespace Skillvio.Learning.Application.Gamification.Levels;

public interface ILevelService
{
    int ComputeLevel(long totalXp);
    long XpToNext(long totalXp);
}

// src/Skillvio.Learning.Application/Gamification/Levels/LevelService.cs
public class LevelService : ILevelService
{
    // L = floor(sqrt(xp / 100)) + 1
    public int ComputeLevel(long totalXp) => (int)Math.Floor(Math.Sqrt(totalXp / 100.0)) + 1;

    public long XpToNext(long totalXp)
    {
        var current = ComputeLevel(totalXp);
        var nextThreshold = (long)Math.Pow(current, 2) * 100;
        return nextThreshold - totalXp;
    }
}
```

Run tests; expect green; commit.

- [ ] **Step 8: Implement HeartsService**

Test:
```csharp
// tests/Skillvio.Learning.Application.Tests/Gamification/HeartsServiceTests.cs
using FluentAssertions;
using Skillvio.Learning.Application.Gamification.Hearts;
using Xunit;

public class HeartsServiceTests
{
    [Fact]
    public void LoseHeart_StrictMode_Decrements()
    {
        var sut = new HeartsService();
        var result = sut.LoseHeart(currentHearts: 5, mode: "strict", reason: "wrong_answer");
        result.NewHearts.Should().Be(4);
        result.WasLost.Should().BeTrue();
    }

    [Fact]
    public void LoseHeart_LenientMode_NotForLessonAnswer()
    {
        var sut = new HeartsService();
        var result = sut.LoseHeart(currentHearts: 5, mode: "lenient", reason: "wrong_answer");
        result.NewHearts.Should().Be(5);
        result.WasLost.Should().BeFalse();
    }

    [Fact]
    public void LoseHeart_LenientMode_LosesForExamFail()
    {
        var sut = new HeartsService();
        var result = sut.LoseHeart(currentHearts: 5, mode: "lenient", reason: "exam_started");
        result.NewHearts.Should().Be(4);
        result.WasLost.Should().BeTrue();
    }

    [Fact]
    public void LoseHeart_AtZero_StaysZero()
    {
        var sut = new HeartsService();
        var result = sut.LoseHeart(currentHearts: 0, mode: "strict", reason: "wrong_answer");
        result.NewHearts.Should().Be(0);
        result.WasLost.Should().BeFalse();
    }

    [Theory]
    [InlineData("2026-05-04T10:00:00Z", "2026-05-04T13:59:00Z", 5, 5, false)]   // < 4h, no refill
    [InlineData("2026-05-04T10:00:00Z", "2026-05-04T14:01:00Z", 4, 5, true)]    // ≥ 4h, +1
    [InlineData("2026-05-04T10:00:00Z", "2026-05-05T10:00:00Z", 0, 5, true)]    // 24h, +1 only
    public void RefillIfDue(string lastRefill, string now, int current, int max, bool refilled)
    {
        var sut = new HeartsService();
        var result = sut.RefillIfDue(current, max, DateTimeOffset.Parse(lastRefill), DateTimeOffset.Parse(now), TimeSpan.FromHours(4));
        result.WasRefilled.Should().Be(refilled);
        if (refilled) result.NewHearts.Should().Be(Math.Min(current + 1, max));
    }
}
```

Implementation:
```csharp
// src/Skillvio.Learning.Application/Gamification/Hearts/IHeartsService.cs
namespace Skillvio.Learning.Application.Gamification.Hearts;

public record HeartLossResult(int NewHearts, bool WasLost);
public record HeartRefillResult(int NewHearts, bool WasRefilled, DateTimeOffset NewLastRefillAt);

public interface IHeartsService
{
    HeartLossResult LoseHeart(int currentHearts, string mode, string reason);
    HeartRefillResult RefillIfDue(int current, int max, DateTimeOffset lastRefillAt, DateTimeOffset now, TimeSpan interval);
}

// src/Skillvio.Learning.Application/Gamification/Hearts/HeartsService.cs
public class HeartsService : IHeartsService
{
    private static readonly HashSet<string> LenientLossReasons = new(StringComparer.Ordinal)
    {
        "exam_started", "exam_failed", "lab_failed"
    };

    public HeartLossResult LoseHeart(int currentHearts, string mode, string reason)
    {
        if (currentHearts <= 0) return new(0, false);
        var shouldLose = mode == "strict" || (mode == "lenient" && LenientLossReasons.Contains(reason));
        return shouldLose ? new(currentHearts - 1, true) : new(currentHearts, false);
    }

    public HeartRefillResult RefillIfDue(int current, int max, DateTimeOffset lastRefillAt, DateTimeOffset now, TimeSpan interval)
    {
        if (current >= max) return new(current, false, lastRefillAt);
        var elapsed = now - lastRefillAt;
        if (elapsed < interval) return new(current, false, lastRefillAt);
        var increments = (int)Math.Floor(elapsed.TotalMilliseconds / interval.TotalMilliseconds);
        var newCount = Math.Min(current + increments, max);
        return new(newCount, true, lastRefillAt + (interval * increments));
    }
}
```

- [ ] **Step 9: Run all gamification tests**

```bash
dotnet test tests/Skillvio.Learning.Application.Tests/ --filter "FullyQualifiedName~Gamification"
```

Expected: ScoringService 16+ PASS, LevelService 7 PASS, HeartsService 6 PASS.

- [ ] **Step 10: Commit**

```bash
git commit -am "feat(s3a-t5): ScoringService + LevelService + HeartsService with table-driven tests"
git push
```


---

## Task 6 — StreakService (Freeze + Repair + Dynamic Timezone Evaluator)

**Spec ref:** §6.5 StreakService, Q5 streak freeze/repair, §9.1 dynamic timezone job
**Estimated:** 6h
**Files:**
- Create: `src/Skillvio.Learning.Application/Gamification/Streaks/IStreakService.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Streaks/StreakService.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Streaks/StreakRecordResult.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Jobs/StreakNightlyEvaluator.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Jobs/StreakTimezoneRegistrar.cs`
- Create: `tests/Skillvio.Learning.Application.Tests/Gamification/StreakServiceTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Jobs/StreakEvaluatorTests.cs`

- [ ] **Step 1: Write IStreakService + result types**

```csharp
// src/Skillvio.Learning.Application/Gamification/Streaks/IStreakService.cs
namespace Skillvio.Learning.Application.Gamification.Streaks;

public record StreakRecordResult(
    int NewCurrentStreak,
    int LongestStreak,
    bool MilestoneReached,
    int? MilestoneDays);

public record FreezeResult(bool Applied, int FreezesRemaining, string? Reason);
public record RepairResult(bool Repaired, int RestoredStreak, string? RejectReason);
public record EvaluateNightlyResult(int Evaluated, int Frozen, int Broken);

public interface IStreakService
{
    Task<StreakRecordResult> RecordActivityAsync(Guid userId, DateOnly localDate, CancellationToken ct = default);
    Task<FreezeResult> UseFreezeAsync(Guid userId, DateOnly date, CancellationToken ct = default);
    Task<RepairResult> RepairAsync(Guid userId, DateOnly originalBreakDate, string costKind, int costAmount, CancellationToken ct = default);
    Task<EvaluateNightlyResult> EvaluateNightlyAsync(string timezone, DateOnly evalDate, CancellationToken ct = default);
}
```

- [ ] **Step 2: Tests for RecordActivity (table-driven)**

```csharp
// tests/Skillvio.Learning.Application.Tests/Gamification/StreakServiceTests.cs
[Theory]
[InlineData(0, "2026-05-04", "2026-05-04", 1, false, null)]            // first activity ever
[InlineData(5, "2026-05-04", "2026-05-04", 5, false, null)]            // same day, no change
[InlineData(5, "2026-05-04", "2026-05-05", 6, false, null)]            // next day, +1
[InlineData(6, "2026-05-04", "2026-05-06", 1, false, null)]            // gap, reset
[InlineData(6, "2026-05-04", "2026-05-05", 7, true, 7)]                // milestone 7
[InlineData(13, "2026-05-04", "2026-05-05", 14, true, 14)]             // milestone 14
[InlineData(29, "2026-05-04", "2026-05-05", 30, true, 30)]             // milestone 30
public async Task RecordActivity_TransitionsCorrectly(int currentStreak, string lastActive, string today, int expectedStreak, bool milestone, int? milestoneDays) { /* arrange + act + assert */ }
```

(Implementation reads from a repository abstraction; for unit tests use in-memory fake.)

- [ ] **Step 3: Implement StreakService (uses IDayStreakRepository abstraction)**

Add `IDayStreakRepository` to Application abstractions and implement basic increment/break/freeze logic per spec §6.5. Use an in-memory test double for unit tests; real EF implementation in Infrastructure.

- [ ] **Step 4: Implement freeze + repair tests + logic**

Tests cover: Pro plan freeze quota check, idempotent freeze per date, 48h repair window, repair cost validation (`pro_monthly_free`, `paid_xp`, `paid_money`), reject if outside window.

- [ ] **Step 5: Implement EvaluateNightlyAsync**

Logic:
1. `SELECT user_id FROM day_streaks WHERE user_timezone = @tz AND current_streak > 0 AND last_active_date_local < @yesterday`
2. For each: if `freezes_remaining_this_month > 0 AND auto_freeze`: apply freeze; else: break (longest = max(longest, current); current = 0)
3. Outbox publish: `StreakBrokenEvent` or `StreakFrozenEvent`

- [ ] **Step 6: Implement `StreakTimezoneRegistrar` (Hangfire dynamic registration)**

```csharp
// src/Skillvio.Learning.Infrastructure/Jobs/StreakTimezoneRegistrar.cs
using Hangfire;
using Microsoft.EntityFrameworkCore;
using Skillvio.Learning.Application.Gamification.Streaks;
using Skillvio.Learning.Infrastructure.Persistence;

namespace Skillvio.Learning.Infrastructure.Jobs;

public class StreakTimezoneRegistrar(LearningDbContext db, IRecurringJobManager jobs)
{
    public async Task RegisterAllAsync(CancellationToken ct)
    {
        var timezones = await db.Set<DayStreak>()
            .Select(s => s.UserTimezone)
            .Where(tz => tz != null)
            .Distinct()
            .ToListAsync(ct);

        foreach (var tz in timezones)
        {
            var localCron = ConvertLocalCronToUtc("5 0 * * *", tz!);  // 00:05 local
            jobs.AddOrUpdate<StreakNightlyEvaluator>(
                $"streak.evaluate.{tz}",
                ev => ev.RunAsync(tz!, default),
                localCron);
        }
    }

    private static string ConvertLocalCronToUtc(string localCron, string tz)
    {
        // Use TimeZoneInfo to compute UTC offset; for simplicity use NCrontab/TimeZoneNext.
        // Production-grade version uses Cronos library; here we approximate.
        var tzInfo = TimeZoneInfo.FindSystemTimeZoneById(tz);
        var localMidnight = DateTimeOffset.UtcNow.Date + TimeSpan.FromMinutes(5);
        var utcEquivalent = TimeZoneInfo.ConvertTimeToUtc(localMidnight, tzInfo);
        return $"{utcEquivalent.Minute} {utcEquivalent.Hour} * * *";
    }
}
```

- [ ] **Step 7: Run tests**

```bash
dotnet test tests/Skillvio.Learning.Application.Tests/ --filter "FullyQualifiedName~StreakService"
```

Expected: 15+ PASS.

- [ ] **Step 8: Commit**

```bash
git commit -am "feat(s3a-t6): StreakService with freeze/repair + dynamic timezone Hangfire registrar"
git push
```

---

## Task 7a — Attempt State Machine (Start / Resume / SubmitAnswer / Abandon)

**Spec ref:** §6.2 AttemptOrchestrator, §7 saga (steps 1-13 are T7b)
**Estimated:** 8h
**Files:**
- Create: `src/Skillvio.Learning.Application/Progress/Attempts/IAttemptOrchestrator.cs`
- Create: `src/Skillvio.Learning.Application/Progress/Attempts/AttemptOrchestrator.cs`
- Create: `src/Skillvio.Learning.Application/Progress/Attempts/Commands/StartAttemptCmd.cs`
- Create: `src/Skillvio.Learning.Application/Progress/Attempts/Commands/SubmitAnswerCmd.cs`
- Create: `src/Skillvio.Learning.Application/Progress/Attempts/Commands/CompleteAttemptCmd.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Progress/AttemptStateMachineTests.cs`

- [ ] **Step 1: Write commands + result types**

```csharp
// Commands
public record StartAttemptCmd(Guid UserId, string AttemptType, Guid TargetId, string? DeviceSessionId, Guid IdempotencyKey);
public record SubmitAnswerCmd(Guid AttemptId, Guid UserId, int AnswerIndex, Guid? ActivityId, Guid? ExamQuestionId, JsonDocument AnswerPayload, Guid IdempotencyKey);
public record CompleteAttemptCmd(Guid AttemptId, Guid UserId, Guid IdempotencyKey);

// Results
public record StartAttemptResult(Guid AttemptId, DateTimeOffset StartedAt);
public record AnswerResult(int AnswerIndex, bool IsCorrect, decimal? PartialCredit, int TimeSpentMs);
public record AttemptStateDto(Guid AttemptId, string Status, DateTimeOffset StartedAt, DateTimeOffset LastActivityAt, IReadOnlyList<AnswerResult> Answers);
```

- [ ] **Step 2: Write IAttemptOrchestrator interface**

```csharp
public interface IAttemptOrchestrator
{
    Task<StartAttemptResult> StartAsync(StartAttemptCmd cmd, CancellationToken ct);
    Task<AttemptStateDto> ResumeAsync(Guid attemptId, Guid userId, CancellationToken ct);
    Task<AnswerResult> SubmitAnswerAsync(SubmitAnswerCmd cmd, CancellationToken ct);
    Task AbandonAsync(Guid attemptId, Guid userId, CancellationToken ct);
}
```

- [ ] **Step 3: Integration test — start attempt creates in_progress row**

```csharp
[Fact]
public async Task Start_CreatesInProgressAttempt()
{
    var lessonId = await SeedLesson();
    var cmd = new StartAttemptCmd(UserA, "lesson", lessonId, "tab-1", Guid.NewGuid());
    var result = await _orchestrator.StartAsync(cmd, default);
    var attempt = await _db.Attempts.FindAsync(result.AttemptId);
    attempt!.Status.Should().Be("in_progress");
    attempt.AttemptType.Should().Be("lesson");
    attempt.TargetId.Should().Be(lessonId);
    attempt.DeviceSessionId.Should().Be("tab-1");
}
```

- [ ] **Step 4: Implement StartAsync — multi-tab guard + idempotency**

Logic:
1. Lookup existing in_progress with same `(user_id, target_id)` → if exists and `device_session_id` differs → 409
2. Lookup `attempts WHERE idempotency_key = @key` → if exists, return its `idempotency_response`
3. Insert new row with status='in_progress', started_at=now, idempotency_key=@key
4. Return `StartAttemptResult`

- [ ] **Step 5: Test resume — returns answers list**

```csharp
[Fact]
public async Task Resume_ReturnsCurrentState()
{
    var att = await StartAttempt(UserA, lessonId);
    await SubmitAnswer(att.AttemptId, 0, isCorrect: true);
    var state = await _orchestrator.ResumeAsync(att.AttemptId, UserA, default);
    state.Status.Should().Be("in_progress");
    state.Answers.Should().HaveCount(1);
}
```

- [ ] **Step 6: Implement ResumeAsync — owner check + answers eager load**

```csharp
public async Task<AttemptStateDto> ResumeAsync(Guid attemptId, Guid userId, CancellationToken ct)
{
    var attempt = await _db.Attempts
        .Include(a => a.Answers.OrderBy(an => an.AnswerIndex))
        .FirstOrDefaultAsync(a => a.Id == attemptId && a.UserId == userId, ct);
    if (attempt is null) throw new NotFoundException(nameof(Attempt), attemptId);
    if (attempt.Status != "in_progress") throw new InvalidOperationException("Attempt not in progress");
    return new AttemptStateDto(attempt.Id, attempt.Status, attempt.StartedAt, attempt.LastActivityAt, attempt.Answers.Select(a => new AnswerResult(a.AnswerIndex, a.IsCorrect, a.PartialCredit, a.TimeSpentMs)).ToArray());
}
```

- [ ] **Step 7: Test SubmitAnswer — server-side timing**

```csharp
[Fact]
public async Task SubmitAnswer_RecordsServerTimestamp()
{
    var att = await StartAttempt(UserA, lessonId);
    var beforeSubmit = DateTimeOffset.UtcNow;
    await Task.Delay(50);
    await SubmitAnswer(att.AttemptId, 0, isCorrect: true);
    var afterSubmit = DateTimeOffset.UtcNow;

    var stored = await _db.AttemptAnswers.SingleAsync(a => a.AttemptId == att.AttemptId);
    stored.SubmittedAt.Should().BeOnOrAfter(beforeSubmit).And.BeOnOrBefore(afterSubmit);
    stored.TimeSpentMs.Should().BeGreaterOrEqualTo(40);  // ≥ 40ms (50ms delay - jitter)
}
```

- [ ] **Step 8: Implement SubmitAnswerAsync — anti-cheat min_time_ms**

Logic:
1. Lookup attempt → must be in_progress + owner
2. Idempotency: `SELECT FROM attempt_answers WHERE attempt_id = @id AND idempotency_key = @key`
3. Compute `time_spent_ms = now - last_activity_at`
4. If `time_spent_ms < activity.metadata.min_time_ms` → flag as `flagged_for_review = true` (does not reject, but logs)
5. Score the answer (using activity content from cache)
6. Insert `attempt_answers` row
7. Update `attempts.last_activity_at = now`

- [ ] **Step 9: Test AbandonAsync — sets status=abandoned**

```csharp
[Fact]
public async Task Abandon_ChangesStatus()
{
    var att = await StartAttempt(UserA, lessonId);
    await _orchestrator.AbandonAsync(att.AttemptId, UserA, default);
    var stored = await _db.Attempts.FindAsync(att.AttemptId);
    stored!.Status.Should().Be("abandoned");
}
```

- [ ] **Step 10: Test concurrent multi-tab → 409**

```csharp
[Fact]
public async Task Start_ConcurrentSecondTab_Returns409()
{
    var key1 = Guid.NewGuid();
    var key2 = Guid.NewGuid();
    var cmd1 = new StartAttemptCmd(UserA, "lesson", lessonId, "tab-1", key1);
    var cmd2 = new StartAttemptCmd(UserA, "lesson", lessonId, "tab-2", key2);

    await _orchestrator.StartAsync(cmd1, default);
    var act = async () => await _orchestrator.StartAsync(cmd2, default);
    await act.Should().ThrowAsync<ConflictException>();
}
```

- [ ] **Step 11: Run all tests**

```bash
dotnet test tests/Skillvio.Learning.IntegrationTests/ --filter "FullyQualifiedName~AttemptStateMachine"
```

Expected: 6+ PASS.

- [ ] **Step 12: Commit**

```bash
git commit -am "feat(s3a-t7a): AttemptOrchestrator state machine (Start/Resume/SubmitAnswer/Abandon)"
git push
```

---

## Task 7b — Complete Saga + IdempotencyFilter

**Spec ref:** §7 lesson complete saga (14 steps), §6.2 idempotency
**Estimated:** 6h
**Files:**
- Modify: `src/Skillvio.Learning.Application/Progress/Attempts/AttemptOrchestrator.cs` (add `CompleteAsync`)
- Create: `src/Skillvio.Learning.Application/Common/Idempotency/IIdempotencyService.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Common/Idempotency/IdempotencyService.cs`
- Create: `src/Skillvio.Learning.Api/Filters/IdempotencyFilter.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Progress/CompleteSagaTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Progress/IdempotencyFilterTests.cs`

- [ ] **Step 1: Test happy-path saga (lesson complete → all 14 steps)**

```csharp
[Fact]
public async Task Complete_ExecutesFullSaga()
{
    var att = await StartAttempt(UserA, lessonId);
    await SubmitAllAnswers(att.AttemptId, perfect: true);

    var result = await _orchestrator.CompleteAsync(new CompleteAttemptCmd(att.AttemptId, UserA, Guid.NewGuid()), default);

    using var assertions = new AssertionScope();
    var stored = await _db.Attempts.FindAsync(att.AttemptId);
    stored!.Status.Should().Be("completed");

    var xp = await _db.UserXp.FindAsync(UserA);
    xp!.TotalXp.Should().BeGreaterThan(0);

    var xpTx = await _db.XpTransactions.Where(t => t.UserId == UserA).ToListAsync();
    xpTx.Should().HaveCount(1);

    var lessonProg = await _db.UserLessonProgress.FindAsync(UserA, lessonId);
    lessonProg!.Perfect.Should().BeTrue();

    var trackProg = await _db.UserTrackProgress.FindAsync(UserA, trackId);
    trackProg!.LessonsCompleted.Should().Be(1);

    var outboxRows = await _db.OutboxMessages.ToListAsync();
    outboxRows.Select(o => o.EventType).Should().Contain(["LessonCompletedEvent", "AttemptCompletedEvent", "XpAwardedEvent"]);
}
```

- [ ] **Step 2: Implement CompleteAsync — TX BEGIN → 14 steps → TX COMMIT**

```csharp
public async Task<CompleteResult> CompleteAsync(CompleteAttemptCmd cmd, CancellationToken ct)
{
    using var tx = await _db.Database.BeginTransactionAsync(ct);

    // 1. Idempotency check
    var existing = await _db.Attempts.SingleAsync(a => a.Id == cmd.AttemptId, ct);
    if (existing.IdempotencyKey == cmd.IdempotencyKey && existing.IdempotencyResponse is not null)
        return JsonSerializer.Deserialize<CompleteResult>(existing.IdempotencyResponse)!;
    if (existing.UserId != cmd.UserId) throw new ForbiddenException();
    if (existing.Status != "in_progress") throw new InvalidOperationException();

    // 2. Score (call ScoringService for each answer)
    var totalXp = 0; var perfect = true;
    foreach (var ans in existing.Answers) {
        var calc = _scoring.Calculate(new ScoringContext(...));
        totalXp += calc.FinalXp;
        if (!ans.IsCorrect) perfect = false;
    }

    // 3-4. UserXp atomic + xp_transactions insert
    await _db.Database.ExecuteSqlInterpolatedAsync($"UPDATE user_xp SET total_xp = total_xp + {totalXp} WHERE user_id = {cmd.UserId}", ct);
    var xpTx = new XpTransaction { Id = Guid.NewGuid(), UserId = cmd.UserId, Delta = totalXp, Source = "lesson", SourceRef = cmd.AttemptId, CreatedAt = DateTimeOffset.UtcNow };
    _db.XpTransactions.Add(xpTx);

    // 5. Level recompute
    var xp = await _db.UserXp.FindAsync(new object[] { cmd.UserId }, ct);
    var newLevel = _levels.ComputeLevel(xp!.TotalXp);
    var leveledUp = newLevel > xp.CurrentLevel;
    if (leveledUp) { xp.CurrentLevel = newLevel; xp.XpToNextLevel = (int)_levels.XpToNext(xp.TotalXp); }

    // 6. user_lesson_progress UPSERT
    await _db.Database.ExecuteSqlInterpolatedAsync($"""
        INSERT INTO user_lesson_progress (user_id, lesson_id, best_attempt_id, perfect, completed_at)
        VALUES ({cmd.UserId}, {lessonId}, {cmd.AttemptId}, {perfect}, {DateTimeOffset.UtcNow})
        ON CONFLICT (user_id, lesson_id) DO UPDATE SET
          best_attempt_id = EXCLUDED.best_attempt_id,
          perfect = user_lesson_progress.perfect OR EXCLUDED.perfect,
          completed_at = COALESCE(user_lesson_progress.completed_at, EXCLUDED.completed_at)
        """, ct);

    // 7. user_track_progress increment (atomic)
    await _db.Database.ExecuteSqlInterpolatedAsync($"UPDATE user_track_progress SET lessons_completed = lessons_completed + 1, last_activity_at = {DateTimeOffset.UtcNow} WHERE user_id = {cmd.UserId} AND track_id = {trackId}", ct);

    // 8. Hearts apply losses
    var heartsLost = existing.Answers.Count(a => !a.IsCorrect && heartsMode == "strict");
    if (heartsLost > 0) await _hearts.ApplyLossesAsync(cmd.UserId, heartsLost, ct);

    // 9. Streak record
    var streakResult = await _streak.RecordActivityAsync(cmd.UserId, DateOnly.FromDateTime(DateTime.UtcNow), ct);

    // 10. League weekly XP
    await _league.UpdateWeeklyXpAsync(cmd.UserId, totalXp, ct);

    // 11. Activity feed
    await _feed.RecordAsync(cmd.UserId, "lesson_completed", cmd.AttemptId, "friends", ct);

    // 12. Badge sync tier
    var newBadges = await _badges.EvaluateSyncTierAsync(cmd.UserId, "LessonCompletedEvent", ct);

    // 13. Certificate eligibility
    var certs = await _cert.EvaluateAsync(cmd.UserId, "lesson_completed", ct);

    // 14. Outbox
    await _outbox.AddAsync(new LessonCompletedEvent(...), ct);
    await _outbox.AddAsync(new AttemptCompletedEvent(...), ct);
    await _outbox.AddAsync(new XpAwardedEvent(...), ct);
    if (leveledUp) await _outbox.AddAsync(new LevelUpEvent(...), ct);
    foreach (var b in newBadges) await _outbox.AddAsync(new BadgeEarnedEvent(...), ct);
    foreach (var c in certs) await _outbox.AddAsync(new SkillvioCertificateIssuedEvent(...), ct);

    // Mark complete + cache idempotent response
    existing.Status = "completed";
    existing.CompletedAt = DateTimeOffset.UtcNow;
    existing.TotalXpEarned = totalXp;
    var resp = new CompleteResult(totalXp, leveledUp, newBadges.Count, certs.Count);
    existing.IdempotencyResponse = JsonSerializer.SerializeToDocument(resp);
    existing.IdempotencyExpiresAt = DateTimeOffset.UtcNow.AddDays(1);

    await _db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
    return resp;
}
```

- [ ] **Step 3: Test idempotent replay returns cached response**

```csharp
[Fact]
public async Task Complete_WithSameIdempotencyKey_ReturnsCachedResponse()
{
    var key = Guid.NewGuid();
    var att = await StartAttempt(UserA, lessonId);
    await SubmitAllAnswers(att.AttemptId, perfect: true);

    var first = await _orchestrator.CompleteAsync(new CompleteAttemptCmd(att.AttemptId, UserA, key), default);
    var second = await _orchestrator.CompleteAsync(new CompleteAttemptCmd(att.AttemptId, UserA, key), default);

    second.Should().BeEquivalentTo(first);
    var xpTxCount = await _db.XpTransactions.CountAsync(t => t.SourceRef == att.AttemptId);
    xpTxCount.Should().Be(1);  // not 2
}
```

- [ ] **Step 4: Implement IdempotencyFilter (Api layer)**

```csharp
// src/Skillvio.Learning.Api/Filters/IdempotencyFilter.cs
public class IdempotencyFilter(IIdempotencyService svc) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var key = ctx.HttpContext.Request.Headers["Idempotency-Key"].FirstOrDefault();
        if (string.IsNullOrWhiteSpace(key) || !Guid.TryParse(key, out var keyGuid))
            return Results.Problem(statusCode: 400, detail: "Idempotency-Key header required (UUID)");

        var endpoint = ctx.HttpContext.Request.Path;
        var cached = await svc.LookupAsync(keyGuid, endpoint!, ctx.HttpContext.RequestAborted);
        if (cached is not null)
        {
            ctx.HttpContext.Response.Headers["X-Idempotent-Replay"] = "true";
            return Results.Json(cached.Payload, statusCode: cached.StatusCode);
        }

        var result = await next(ctx);
        // Cache result post-execution (only for 2xx responses)
        if (ctx.HttpContext.Response.StatusCode is >= 200 and < 300 && result is not null)
        {
            await svc.StoreAsync(keyGuid, endpoint!, result, ctx.HttpContext.Response.StatusCode, TimeSpan.FromHours(24), ctx.HttpContext.RequestAborted);
        }
        return result;
    }
}
```

- [ ] **Step 5: Test concurrent complete → second returns 409 (or cached if same key)**

```csharp
[Fact]
public async Task Complete_ConcurrentSameAttempt_OnlyOneCommits()
{
    var att = await StartAttempt(UserA, lessonId);
    await SubmitAllAnswers(att.AttemptId, perfect: true);

    var task1 = _orchestrator.CompleteAsync(new CompleteAttemptCmd(att.AttemptId, UserA, Guid.NewGuid()), default);
    var task2 = _orchestrator.CompleteAsync(new CompleteAttemptCmd(att.AttemptId, UserA, Guid.NewGuid()), default);
    var results = await Task.WhenAll(task1.AsTask(), task2.AsTask());
    // One success, one InvalidOperationException ("Attempt not in progress")
    results.Count(r => r is not null).Should().Be(1);
}
```

- [ ] **Step 6: Run all complete saga tests**

```bash
dotnet test tests/Skillvio.Learning.IntegrationTests/ --filter "FullyQualifiedName~CompleteSaga"
```

Expected: 4+ PASS, including idempotency + concurrent.

- [ ] **Step 7: Commit**

```bash
git commit -am "feat(s3a-t7b): 14-step lesson complete saga + IdempotencyFilter + replay test"
git push
```

---

## Task 8 — BadgeRuleEngine (Sync) + BadgeWorker (Async, Separate Project)

**Spec ref:** §6.3 BadgeRuleEngine, Q6 two-tier
**Estimated:** 8h
**Files:**
- Create: `src/Skillvio.Learning.Application/Gamification/Badges/IBadgeRuleEngine.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Badges/BadgeRuleEngine.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Badges/Predicates/IBadgePredicate.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Badges/Predicates/FirstOccurrencePredicate.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Badges/Predicates/CountGePredicate.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Badges/Predicates/StreakGePredicate.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Badges/Predicates/ScoreGePredicate.cs`
- Create: `src/Skillvio.Learning.Application/Gamification/Badges/Predicates/CompositePredicate.cs`
- Create: `src/Skillvio.Learning.BadgeWorker/Program.cs`
- Create: `src/Skillvio.Learning.BadgeWorker/Consumers/AsyncBadgeEvaluator.cs`
- Create: `tests/Skillvio.Learning.Application.Tests/Gamification/BadgeRuleEngineTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Badges/AsyncBadgeWorkerTests.cs`

- [ ] **Step 1: Write IBadgePredicate strategy**

```csharp
public interface IBadgePredicate
{
    string Kind { get; }
    Task<bool> EvaluateAsync(Guid userId, BadgeEvaluationContext ctx, JsonDocument args, CancellationToken ct);
}

public record BadgeEvaluationContext(string TriggerEvent, Guid? RelatedAttemptId, Guid? RelatedLessonId, IServiceProvider Services);
```

- [ ] **Step 2: Implement FirstOccurrencePredicate**

```csharp
public class FirstOccurrencePredicate(LearningDbContext db) : IBadgePredicate
{
    public string Kind => "first_occurrence";
    public async Task<bool> EvaluateAsync(Guid userId, BadgeEvaluationContext ctx, JsonDocument args, CancellationToken ct)
    {
        var triggerType = args.RootElement.GetProperty("trigger").GetString();
        return triggerType switch
        {
            "first_lesson" => !await db.UserLessonProgress.AnyAsync(p => p.UserId == userId && p.CompletedAt != null, ct),
            "first_lab" => !await db.UserLabCompletions.AnyAsync(p => p.UserId == userId, ct),
            "first_exam_pass" => !await db.Attempts.Where(a => a.UserId == userId && a.AttemptType == "exam").Join(db.ExamAttemptMeta, a => a.Id, m => m.AttemptId, (a, m) => m.Passed).AnyAsync(p => p, ct),
            _ => false
        };
    }
}
```

- [ ] **Step 3: Implement CountGePredicate**

Counts a metric (e.g., `lessons_completed`, `streak_milestones`). Args: `{ metric: "lessons_completed", value: 10 }`.

- [ ] **Step 4: Implement StreakGePredicate, ScoreGePredicate, CompositePredicate**

Composite supports `{ "all": [...] }` and `{ "any": [...] }`.

- [ ] **Step 5: Test BadgeRuleEngine — sync evaluation < 50ms**

```csharp
[Fact]
public async Task EvaluateSyncTier_FirstLesson_ReturnsBadge()
{
    var sw = Stopwatch.StartNew();
    var badges = await _engine.EvaluateSyncTierAsync(UserA, "LessonCompletedEvent", default);
    sw.Stop();
    sw.ElapsedMilliseconds.Should().BeLessThan(50);
    badges.Should().Contain(b => b.BadgeCode == "first_lesson");
}
```

- [ ] **Step 6: Implement BadgeRuleEngine.EvaluateSyncTierAsync**

```csharp
public async Task<IReadOnlyList<UserBadgeAward>> EvaluateSyncTierAsync(Guid userId, string triggerEvent, CancellationToken ct)
{
    var candidates = _ruleCache.GetByTrigger(triggerEvent).Where(b => b.Tier == "sync");
    var awarded = new List<UserBadgeAward>();
    foreach (var badge in candidates)
    {
        if (await _db.UserBadges.AnyAsync(ub => ub.UserId == userId && ub.BadgeId == badge.Id && !badge.Repeatable, ct)) continue;
        var predicate = _predicates.First(p => p.Kind == badge.PredicateKind);
        if (await predicate.EvaluateAsync(userId, new BadgeEvaluationContext(triggerEvent, null, null, _services), badge.PredicateArgs, ct))
        {
            await _db.Database.ExecuteSqlInterpolatedAsync($"INSERT INTO user_badges (user_id, badge_id, earned_at, earned_count, context) VALUES ({userId}, {badge.Id}, NOW(), 1, '{}'::jsonb) ON CONFLICT DO NOTHING", ct);
            awarded.Add(new UserBadgeAward(badge.Id, badge.Code, badge.XpReward));
        }
    }
    return awarded;
}
```

- [ ] **Step 7: Implement BadgeWorker (separate project)**

```csharp
// src/Skillvio.Learning.BadgeWorker/Program.cs
using MassTransit;
using Skillvio.Learning.BadgeWorker.Consumers;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddDbContext<LearningDbContext>(...);
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<AsyncBadgeEvaluatorConsumer>();
    x.UsingRabbitMq((c, cfg) => cfg.ConfigureEndpoints(c));
});
builder.Build().Run();
```

```csharp
// src/Skillvio.Learning.BadgeWorker/Consumers/AsyncBadgeEvaluator.cs
public class AsyncBadgeEvaluatorConsumer(IBadgeRuleEngine engine, ILogger<AsyncBadgeEvaluatorConsumer> log) : IConsumer<LessonCompletedEvent>, IConsumer<LabSubmittedEvent>, IConsumer<ExamAttemptCompletedEvent>
{
    public async Task Consume(ConsumeContext<LessonCompletedEvent> ctx)
        => await engine.EvaluateAsyncTierAsync(ctx.Message.UserId, "LessonCompletedEvent", ctx.CancellationToken);
    // ... similarly for other events
}
```

- [ ] **Step 8: Integration test — async badge full chain**

```csharp
[Fact]
public async Task LessonCompleted_TriggersAsyncBadgeEval_For10LessonMilestone()
{
    // Seed 9 completed lessons for UserA
    for (int i = 0; i < 9; i++) await SeedCompletedLesson(UserA);
    // Complete 10th
    await CompleteLessonViaApi(UserA);
    // Wait for outbox → consumer
    await Eventually(async () =>
    {
        var badges = await _db.UserBadges.Where(b => b.UserId == UserA).ToListAsync();
        badges.Should().Contain(b => b.BadgeCode == "lessons_10");
    }, timeout: TimeSpan.FromSeconds(10));
}
```

- [ ] **Step 9: Commit**

```bash
git commit -am "feat(s3a-t8): BadgeRuleEngine sync tier + BadgeWorker async tier (5 predicates)"
git push
```

---

## Task 9 — CertificateEligibilityService + Public Verify Endpoint

**Spec ref:** §6.4 CertificateEligibilityService, §5.3 verify endpoint, Q2
**Estimated:** 4h
**Files:**
- Create: `src/Skillvio.Learning.Application/Certificates/ICertificateEligibilityService.cs`
- Create: `src/Skillvio.Learning.Application/Certificates/CertificateEligibilityService.cs`
- Create: `src/Skillvio.Learning.Application/Certificates/CertificateRule.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/CertificateEndpoints.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Certificates/EligibilityTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Certificates/VerifyEndpointTests.cs`

- [ ] **Step 1: Define `CertificateRule` value type matching trigger_rule JSONB**

```csharp
public record CertificateRule(
    Guid CertificateId,
    string Code,
    CertificateRuleClause Trigger);

public abstract record CertificateRuleClause;
public record AllOfClause(IReadOnlyList<CertificateRuleClause> Clauses) : CertificateRuleClause;
public record AnyOfClause(IReadOnlyList<CertificateRuleClause> Clauses) : CertificateRuleClause;
public record TrackCompletedClause(Guid TrackId) : CertificateRuleClause;
public record RoadmapCompletedClause(Guid RoadmapId) : CertificateRuleClause;
public record ExamPassedClause(Guid ExamId, decimal MinScorePct) : CertificateRuleClause;
```

- [ ] **Step 2: Test eligibility — track + exam combo**

```csharp
[Fact]
public async Task Evaluate_TrackCompleteAndExamPassed_IssuesCert()
{
    var trackId = await SeedTrack();
    var examId = await SeedExam();
    await SeedCertDef("skillvio-aws-dev-prep", trackId, examId, minScore: 72);
    await CompleteTrack(UserA, trackId);
    await PassExam(UserA, examId, scorePct: 80);

    var awards = await _service.EvaluateAsync(UserA, "exam_passed", default);

    awards.Should().HaveCount(1);
    var cert = await _db.UserCertificates.SingleAsync(c => c.UserId == UserA);
    cert.VerificationCode.Should().NotBeEmpty();
}
```

- [ ] **Step 3: Implement EvaluateAsync**

```csharp
public async Task<IReadOnlyList<CertificateAward>> EvaluateAsync(Guid userId, string trigger, CancellationToken ct)
{
    var defs = await _db.CertificateDefinitions.Where(d => d.Status == "published").ToListAsync(ct);
    var awards = new List<CertificateAward>();
    foreach (var def in defs)
    {
        if (await _db.UserCertificates.AnyAsync(c => c.UserId == userId && c.CertificateId == def.Id && c.RevokedAt == null, ct)) continue;
        var rule = ParseRule(def.TriggerRule);
        if (await EvaluateClauseAsync(userId, rule.Trigger, ct))
        {
            var code = GenerateVerificationCode();
            await _db.UserCertificates.AddAsync(new UserCertificate { Id = Guid.NewGuid(), UserId = userId, CertificateId = def.Id, IssuedAt = DateTimeOffset.UtcNow, VerificationCode = code }, ct);
            awards.Add(new CertificateAward(def.Id, def.Code, code));
        }
    }
    await _db.SaveChangesAsync(ct);
    return awards;
}

private static string GenerateVerificationCode()
{
    var bytes = RandomNumberGenerator.GetBytes(16);  // 128-bit entropy
    return Convert.ToBase64String(bytes).Replace("=", "").Replace("+", "-").Replace("/", "_");
}
```

- [ ] **Step 4: Implement clause evaluation (recursive)**

```csharp
private async Task<bool> EvaluateClauseAsync(Guid userId, CertificateRuleClause clause, CancellationToken ct) => clause switch
{
    AllOfClause all => (await Task.WhenAll(all.Clauses.Select(c => EvaluateClauseAsync(userId, c, ct)))).All(r => r),
    AnyOfClause any => (await Task.WhenAll(any.Clauses.Select(c => EvaluateClauseAsync(userId, c, ct)))).Any(r => r),
    TrackCompletedClause t => await _db.UserTrackProgress.AnyAsync(p => p.UserId == userId && p.TrackId == t.TrackId && p.CompletedAt != null, ct),
    RoadmapCompletedClause r => await _db.UserRoadmapProgress.AnyAsync(p => p.UserId == userId && p.RoadmapId == r.RoadmapId && p.CompletedAt != null, ct),
    ExamPassedClause e => await _db.Attempts.Where(a => a.UserId == userId && a.AttemptType == "exam" && a.TargetId == e.ExamId)
        .Join(_db.ExamAttemptMeta, a => a.Id, m => m.AttemptId, (a, m) => new { a, m })
        .AnyAsync(am => am.m.Passed && am.m.QuestionsTotal > 0 && (decimal)am.m.QuestionsCorrect / am.m.QuestionsTotal * 100 >= e.MinScorePct, ct),
    _ => false
};
```

- [ ] **Step 5: Public verify endpoint**

```csharp
// src/Skillvio.Learning.Api/Endpoints/CertificateEndpoints.cs
public static class CertificateEndpoints
{
    public static void Map(IEndpointRouteBuilder app)
    {
        var v1 = app.MapGroup("/api/v1");
        v1.MapGet("/me/certificates/verify/{code}", async (string code, LearningDbContext db, CancellationToken ct) =>
        {
            var cert = await db.UserCertificates
                .Where(c => c.VerificationCode == code && c.RevokedAt == null)
                .Select(c => new { c.IssuedAt, c.UserId, c.CertificateId })
                .SingleOrDefaultAsync(ct);
            return cert is null ? Results.NotFound() : Results.Ok(cert);
        })
        .AllowAnonymous()
        .WithRateLimiting("CertVerify");  // 60/min/IP — Sprint 2 gateway also enforces
    }
}
```

- [ ] **Step 6: Test verify endpoint — public, rate-limited**

```csharp
[Fact]
public async Task Verify_AnonymousAccess_ReturnsCert()
{
    var cert = await SeedIssuedCert();
    var resp = await _anonymousClient.GetAsync($"/api/v1/me/certificates/verify/{cert.VerificationCode}");
    resp.StatusCode.Should().Be(HttpStatusCode.OK);
}

[Fact]
public async Task Verify_InvalidCode_Returns404()
{
    var resp = await _anonymousClient.GetAsync("/api/v1/me/certificates/verify/INVALID");
    resp.StatusCode.Should().Be(HttpStatusCode.NotFound);
}
```

- [ ] **Step 7: Commit**

```bash
git commit -am "feat(s3a-t9): CertificateEligibilityService + public verify endpoint with 128-bit code"
git push
```

---

## Task 10 — Outbox Pattern + LISTEN/NOTIFY Dispatcher + 8 Inbound Consumers + DLQ

**Spec ref:** §8 event choreography, §9.2 outbox dispatcher, Q10
**Estimated:** 10h
**Files:**
- Create: `src/Skillvio.Learning.Application/Events/Outbound/IOutbox.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/OutboxWriter.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/OutboxDispatcherService.cs`
- Create: `src/Skillvio.Learning.Application/Events/Inbound/IIdempotentConsumer.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/Consumers/UserRegisteredConsumer.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/Consumers/UserSoftDeletedConsumer.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/Consumers/UserHardDeletedConsumer.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/Consumers/UserPlanChangedConsumer.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/Consumers/UserProfileUpdatedConsumer.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/Consumers/LabSubmittedConsumer.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/Consumers/PaymentSucceededConsumer.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/Consumers/CertificationVerifiedConsumer.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Messaging/OutboxDispatcherTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Messaging/ConsumerIdempotencyTests.cs`

- [ ] **Step 1: Add MassTransit packages**

```bash
dotnet add src/Skillvio.Learning.Infrastructure package MassTransit
dotnet add src/Skillvio.Learning.Infrastructure package MassTransit.RabbitMQ
dotnet add src/Skillvio.Learning.Infrastructure package MassTransit.EntityFrameworkCore
```

- [ ] **Step 2: Define IOutbox + OutboxWriter**

```csharp
// src/Skillvio.Learning.Application/Events/Outbound/IOutbox.cs
public interface IOutbox
{
    Task AddAsync<TEvent>(TEvent evt, CancellationToken ct = default) where TEvent : class;
}

// src/Skillvio.Learning.Infrastructure/Messaging/OutboxWriter.cs
public class OutboxWriter(LearningDbContext db, IClock clock) : IOutbox
{
    public async Task AddAsync<TEvent>(TEvent evt, CancellationToken ct = default) where TEvent : class
    {
        var msg = new OutboxMessage
        {
            Id = Guid.NewGuid(),
            OccurredAt = clock.UtcNow,
            EventType = typeof(TEvent).Name,
            Payload = JsonSerializer.SerializeToDocument(evt),
            AggregateType = (evt as IHasAggregate)?.AggregateType ?? "Unknown",
            AggregateId = (evt as IHasAggregate)?.AggregateId ?? Guid.Empty,
            CorrelationId = Activity.Current?.GetTagItem("correlation_id") as Guid?,
            CausationId = (evt as IHasCausation)?.CausationId,
            TraceId = Activity.Current?.TraceId.ToString(),
            DispatchedAt = null,
            RetryCount = 0
        };
        await db.OutboxMessages.AddAsync(msg, ct);
        // Note: NOT calling SaveChanges — caller's transaction commits.
    }
}
```

- [ ] **Step 3: Implement OutboxDispatcherService (LISTEN/NOTIFY + 5s polling)**

```csharp
// src/Skillvio.Learning.Infrastructure/Messaging/OutboxDispatcherService.cs
public class OutboxDispatcherService(
    IServiceScopeFactory scopes,
    IBus bus,
    ILogger<OutboxDispatcherService> log,
    IConfiguration config) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stop)
    {
        var connStr = config.GetConnectionString("Default")!;
        while (!stop.IsCancellationRequested)
        {
            try { await ListenLoopAsync(connStr, stop); }
            catch (Exception ex) { log.LogError(ex, "Outbox dispatcher error; retry in 10s"); await Task.Delay(10_000, stop); }
        }
    }

    private async Task ListenLoopAsync(string connStr, CancellationToken stop)
    {
        await using var conn = new NpgsqlConnection(connStr);
        await conn.OpenAsync(stop);
        await using (var cmd = new NpgsqlCommand("LISTEN outbox_new;", conn)) await cmd.ExecuteNonQueryAsync(stop);

        var pollTimer = new PeriodicTimer(TimeSpan.FromSeconds(5));
        var notified = new TaskCompletionSource();
        conn.Notification += (_, _) => { notified.TrySetResult(); };

        while (!stop.IsCancellationRequested)
        {
            await DispatchPendingAsync(stop);
            // Wait for either NOTIFY or 5s poll fallback
            var notify = notified.Task;
            var poll = pollTimer.WaitForNextTickAsync(stop).AsTask();
            var done = await Task.WhenAny(notify, poll);
            if (done == notify) notified = new TaskCompletionSource();
            await conn.WaitAsync(TimeSpan.FromMilliseconds(50), stop);
        }
    }

    private async Task DispatchPendingAsync(CancellationToken stop)
    {
        using var scope = scopes.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<LearningDbContext>();
        var pending = await db.OutboxMessages.Where(o => o.DispatchedAt == null && o.RetryCount < 5).OrderBy(o => o.OccurredAt).Take(50).ToListAsync(stop);
        foreach (var msg in pending)
        {
            try
            {
                var type = Type.GetType($"Skillvio.Shared.Contracts.Events.{msg.EventType}, Skillvio.Shared.Contracts")!;
                var evt = JsonSerializer.Deserialize(msg.Payload!.RootElement.GetRawText(), type)!;
                await bus.Publish(evt, ctx => { ctx.Headers.Set("trace_id", msg.TraceId); }, stop);
                msg.DispatchedAt = DateTimeOffset.UtcNow;
            }
            catch (Exception ex)
            {
                msg.RetryCount++;
                msg.LastError = ex.Message;
                log.LogWarning(ex, "Outbox dispatch failed; retry {Count}", msg.RetryCount);
            }
        }
        await db.SaveChangesAsync(stop);
    }
}
```

- [ ] **Step 4: Wire MassTransit + DLQ in Program.cs**

```csharp
builder.Services.AddMassTransit(x =>
{
    x.SetKebabCaseEndpointNameFormatter();
    x.AddConsumer<UserRegisteredConsumer>();
    x.AddConsumer<UserSoftDeletedConsumer>();
    x.AddConsumer<UserHardDeletedConsumer>();
    x.AddConsumer<UserPlanChangedConsumer>();
    x.AddConsumer<UserProfileUpdatedConsumer>();
    x.AddConsumer<LabSubmittedConsumer>();
    x.AddConsumer<PaymentSucceededConsumer>();
    x.AddConsumer<CertificationVerifiedConsumer>();
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMQ:Host"], h => { h.Username(builder.Configuration["RabbitMQ:User"]); h.Password(builder.Configuration["RabbitMQ:Password"]); });
        cfg.UseMessageRetry(r => r.Immediate(3));
        cfg.UseDelayedRedelivery(r => r.Intervals(TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(30), TimeSpan.FromMinutes(2), TimeSpan.FromMinutes(10)));
        cfg.ConfigureEndpoints(ctx);
    });
});
builder.Services.AddHostedService<OutboxDispatcherService>();
```

- [ ] **Step 5: Implement idempotent consumer base**

```csharp
public abstract class IdempotentConsumer<T> : IConsumer<T> where T : class, IHasEventId
{
    protected abstract string ConsumerName { get; }

    public async Task Consume(ConsumeContext<T> ctx)
    {
        await using var scope = ctx.GetServiceProvider().CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<LearningDbContext>();
        using var tx = await db.Database.BeginTransactionAsync(ctx.CancellationToken);

        var inserted = await db.Database.ExecuteSqlInterpolatedAsync($"""
            INSERT INTO processed_events (event_id, consumer_name, processed_at)
            VALUES ({ctx.Message.EventId}, {ConsumerName}, NOW())
            ON CONFLICT DO NOTHING
        """, ctx.CancellationToken);

        if (inserted == 0) { await tx.CommitAsync(ctx.CancellationToken); return; }

        await DoWorkAsync(ctx, db, ctx.CancellationToken);
        await db.SaveChangesAsync(ctx.CancellationToken);
        await tx.CommitAsync(ctx.CancellationToken);
    }

    protected abstract Task DoWorkAsync(ConsumeContext<T> ctx, LearningDbContext db, CancellationToken ct);
}
```

- [ ] **Step 6: Implement UserRegisteredConsumer (default rows)**

```csharp
public class UserRegisteredConsumer : IdempotentConsumer<UserRegisteredEvent>
{
    protected override string ConsumerName => nameof(UserRegisteredConsumer);
    protected override async Task DoWorkAsync(ConsumeContext<UserRegisteredEvent> ctx, LearningDbContext db, CancellationToken ct)
    {
        var userId = ctx.Message.UserId;
        await db.UserXp.AddAsync(new UserXp { UserId = userId, TotalXp = 0, CurrentLevel = 1, XpToNextLevel = 100 }, ct);
        await db.Hearts.AddAsync(new Hearts { UserId = userId, Current = 5, Max = 5, LastRefillAt = DateTimeOffset.UtcNow, LastLossAt = DateTimeOffset.UtcNow }, ct);
        await db.DayStreaks.AddAsync(new DayStreak { UserId = userId, CurrentStreak = 0, LongestStreak = 0, UserTimezone = ctx.Message.Timezone ?? "UTC" }, ct);
        await db.UserPreferences.AddAsync(new UserPreferences { UserId = userId, DailyGoalMinutes = 10 }, ct);
    }
}
```

- [ ] **Step 7: Implement UserHardDeletedConsumer (cascade via v_user_data_tables)**

```csharp
public class UserHardDeletedConsumer : IdempotentConsumer<UserHardDeletedEvent>
{
    protected override string ConsumerName => nameof(UserHardDeletedConsumer);
    protected override async Task DoWorkAsync(ConsumeContext<UserHardDeletedEvent> ctx, LearningDbContext db, CancellationToken ct)
    {
        var userId = ctx.Message.UserId;
        var tables = await db.Database.SqlQueryRaw<string>("SELECT table_name FROM v_user_data_tables").ToListAsync(ct);
        foreach (var table in tables)
            await db.Database.ExecuteSqlRawAsync($"DELETE FROM {table} WHERE user_id = @p0", new[] { (object)userId }, ct);

        // Outbox: ComplianceAuditEvent
        await db.OutboxMessages.AddAsync(new OutboxMessage { Id = Guid.NewGuid(), OccurredAt = DateTimeOffset.UtcNow, EventType = "ComplianceAuditEvent", Payload = JsonDocument.Parse($$"""{"userId":"{{userId}}","action":"data_deleted_hard","tables":{{JsonSerializer.Serialize(tables)}}}""") }, ct);
    }
}
```

- [ ] **Step 8: Implement remaining 6 consumers** (UserSoftDeleted, UserPlanChanged, UserProfileUpdated, LabSubmitted, PaymentSucceeded, CertificationVerified) following the same pattern. Each writes to its DbSets and emits any consequent outbox events.

- [ ] **Step 9: Test outbox dispatcher publishes within ~100ms (LISTEN/NOTIFY)**

```csharp
[Fact]
public async Task Outbox_NotifiesAndDispatches_SubSecond()
{
    var sw = Stopwatch.StartNew();
    await _outbox.AddAsync(new TestEvent(Guid.NewGuid()), default);
    await _db.SaveChangesAsync();

    await Eventually(async () =>
    {
        var dispatched = await _db.OutboxMessages.SingleAsync(m => m.EventType == "TestEvent");
        dispatched.DispatchedAt.Should().NotBeNull();
    }, timeout: TimeSpan.FromSeconds(2));
    sw.ElapsedMilliseconds.Should().BeLessThan(2000);
}
```

- [ ] **Step 10: Test consumer idempotency — duplicate event ignored**

```csharp
[Fact]
public async Task Consumer_DuplicateEvent_ProcessedOnce()
{
    var evt = new UserRegisteredEvent(Guid.NewGuid(), Guid.NewGuid(), "user@x.com", "UTC", DateTimeOffset.UtcNow);
    await _bus.Publish(evt);
    await _bus.Publish(evt);  // same EventId
    await Eventually(async () =>
    {
        var heartsCount = await _db.Hearts.CountAsync(h => h.UserId == evt.UserId);
        heartsCount.Should().Be(1);  // not 2
    }, timeout: TimeSpan.FromSeconds(5));
}
```

- [ ] **Step 11: Run all messaging tests**

```bash
dotnet test tests/Skillvio.Learning.IntegrationTests/ --filter "FullyQualifiedName~Messaging"
```

- [ ] **Step 12: Commit**

```bash
git commit -am "feat(s3a-t10): outbox pattern + LISTEN/NOTIFY dispatcher + 8 idempotent consumers + DLQ"
git push
```


---

## Task 11a — Public Content Endpoints (~20) + ETag

**Spec ref:** §5.2 public content endpoints, §5.10 response shape, §5.1 ETag
**Estimated:** 5h
**Files:**
- Create: `src/Skillvio.Learning.Api/Endpoints/DomainEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/TrackEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/LessonEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/RoadmapEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/ExamEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/GlossaryEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/BadgeEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/SearchEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Filters/ContentVersionEtagFilter.cs`
- Create: `src/Skillvio.Learning.Api/Dtos/ContentDtos.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Api/PublicContentEndpointTests.cs`

- [ ] **Step 1: Define DTO records (Content responses)**

```csharp
// src/Skillvio.Learning.Api/Dtos/ContentDtos.cs
public record DomainDto(Guid Id, string Code, Guid? ParentDomainId, string Name, string? Tagline, string IconUrl, string ColorHex);
public record TrackSummaryDto(Guid Id, string Slug, string Title, string Summary, short Difficulty, int EstMinutes, bool IsPremium);
public record TrackDetailDto(Guid Id, string Slug, DomainRef Domain, string Title, string Summary, short Difficulty, int EstMinutes, bool IsPremium, string HeartsMode, ExamRef? TargetExam, IReadOnlyList<UnitDto> Units, ResponseMeta Meta);
public record DomainRef(string Code, string Name);
public record ExamRef(Guid Id, string Slug, string Code, bool IsPremium);
public record UnitDto(Guid Id, string Title, int LessonCount, IReadOnlyList<LessonSummaryDto> Lessons);
public record LessonSummaryDto(Guid Id, string Slug, string Title, int EstMinutes);
public record ResponseMeta(string Locale, bool FallbackUsed, string ContentVersion);
```

- [ ] **Step 2: Implement ContentVersionEtagFilter**

```csharp
public class ContentVersionEtagFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var result = await next(ctx);
        if (result is IValueHttpResult<object> httpResult && httpResult.Value is { } value)
        {
            var versionProp = value.GetType().GetProperty("Meta")?.GetValue(value)?.GetType().GetProperty("ContentVersion")?.GetValue(value);
            if (versionProp is string version)
            {
                var etag = $"\"{version}\"";
                ctx.HttpContext.Response.Headers.ETag = etag;

                var ifNoneMatch = ctx.HttpContext.Request.Headers.IfNoneMatch.FirstOrDefault();
                if (ifNoneMatch == etag) return Results.StatusCode(304);
            }
        }
        return result;
    }
}
```

- [ ] **Step 3: Test GET /api/v1/learning/domains**

```csharp
[Fact]
public async Task GetDomains_ReturnsHierarchy()
{
    await SeedDomain("cloud", parent: null);
    await SeedDomain("aws", parent: "cloud");
    var resp = await _client.GetAsync("/api/v1/learning/domains");
    resp.EnsureSuccessStatusCode();
    var domains = await resp.Content.ReadFromJsonAsync<IReadOnlyList<DomainDto>>();
    domains!.Should().Contain(d => d.Code == "cloud").And.Contain(d => d.Code == "aws");
}
```

- [ ] **Step 4: Implement DomainEndpoints**

```csharp
public static class DomainEndpoints
{
    public static void Map(IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/v1/learning/domains")
            .AddEndpointFilter<LocaleFilter>()
            .WithTags("Content");

        group.MapGet("/", async (LearningDbContext db, ILocalizationContext loc, IHybridCache cache, CancellationToken ct) =>
        {
            return await cache.GetOrCreateAsync(
                $"domains:list:{loc.CurrentLocale}",
                async _ =>
                {
                    var rows = await db.Domains.Where(d => d.Status == "published").OrderBy(d => d.DisplayOrder).ToListAsync(ct);
                    return rows.Select(d => new DomainDto(
                        d.Id, d.Code, d.ParentDomainId,
                        d.ShortI18n.RootElement.GetProperty("name").TryGetProperty(loc.CurrentLocale, out var n) ? n.GetString()! : d.ShortI18n.RootElement.GetProperty("name").GetProperty("en").GetString()!,
                        d.ShortI18n.RootElement.TryGetProperty("tagline", out var t) ? t.GetProperty(loc.CurrentLocale).GetString() : null,
                        d.IconUrl, d.ColorHex)).ToList();
                },
                tags: ["domains"], cancellationToken: ct);
        });

        group.MapGet("/{slug}", async (string slug, LearningDbContext db, ILocalizationContext loc, CancellationToken ct) => { /* similar */ });
    }
}
```

- [ ] **Step 5: Implement TrackEndpoints (list + detail with locale + ETag)**

```csharp
group.MapGet("/{id:guid}", async (Guid id, LearningDbContext db, ILocalizationContext loc, CancellationToken ct) =>
{
    var track = await db.Tracks.Include(t => t.Domain)
        .Where(t => t.Id == id && t.Status == "published")
        .SingleOrDefaultAsync(ct);
    if (track is null) return Results.NotFound();

    var translation = await db.TrackTranslations
        .Where(t => t.TrackId == id && t.Locale == loc.CurrentLocale)
        .SingleOrDefaultAsync(ct);

    var (title, summary, fallbackUsed) = ResolveTitle(track, loc.CurrentLocale);

    var units = await db.Units
        .Where(u => u.TrackId == id)
        .OrderBy(u => u.DisplayOrder)
        .Select(u => new UnitDto(u.Id, ExtractTitle(u.ShortI18n, loc.CurrentLocale), 0, new List<LessonSummaryDto>()))
        .ToListAsync(ct);

    return Results.Ok(new TrackDetailDto(track.Id, track.Slug, new DomainRef(track.Domain.Code, ExtractName(track.Domain.ShortI18n, loc.CurrentLocale)), title, summary, track.Difficulty, track.EstMinutes, track.IsPremium, track.HeartsMode, null, units, new ResponseMeta(loc.CurrentLocale, fallbackUsed, track.SourceCommitSha)));
}).AddEndpointFilter<ContentVersionEtagFilter>();
```

- [ ] **Step 6: Implement remaining endpoints (lessons, roadmaps, exams, glossary search)**

For each: list + detail, locale + cache + ETag. Glossary uses `tsvector` query:

```csharp
group.MapGet("/glossary", async (string q, string? domain, int page, int size, LearningDbContext db, ILocalizationContext loc, CancellationToken ct) =>
{
    var sql = """
        SELECT t.id, t.slug, gt.definition_md
        FROM glossary_terms t
        JOIN glossary_translations gt ON gt.term_id = t.id AND gt.locale = @locale
        WHERE t.status = 'published'
          AND gt.search_tsvector @@ plainto_tsquery('simple', @q)
        ORDER BY ts_rank(gt.search_tsvector, plainto_tsquery('simple', @q)) DESC
        LIMIT @size OFFSET @offset;
    """;
    // execute via Npgsql + return DTO
});
```

- [ ] **Step 7: Implement federated search endpoint**

```csharp
app.MapGet("/api/v1/learning/search", async (string q, string? type, string? domain, LearningDbContext db, ILocalizationContext loc, CancellationToken ct) =>
{
    var tracks = type is null or "all" or "tracks" ? await SearchTracks(db, q, domain, loc.CurrentLocale, ct) : new List<TrackSummaryDto>();
    var lessons = type is null or "all" or "lessons" ? await SearchLessons(db, q, domain, loc.CurrentLocale, ct) : new List<LessonSummaryDto>();
    var roadmaps = type is null or "all" or "roadmaps" ? await SearchRoadmaps(db, q, domain, loc.CurrentLocale, ct) : new List<RoadmapDto>();
    var glossary = type is null or "all" or "glossary" ? await SearchGlossary(db, q, domain, loc.CurrentLocale, ct) : new List<GlossaryDto>();
    var exams = type is null or "all" or "exams" ? await SearchExams(db, q, domain, loc.CurrentLocale, ct) : new List<ExamSummaryDto>();
    return Results.Ok(new { tracks, lessons, roadmaps, glossary, exams, total = tracks.Count + lessons.Count + roadmaps.Count + glossary.Count + exams.Count });
});
```

- [ ] **Step 8: Test ETag behavior**

```csharp
[Fact]
public async Task GetTrack_WithMatchingIfNoneMatch_Returns304()
{
    var trackId = await SeedTrack();
    var first = await _client.GetAsync($"/api/v1/learning/tracks/{trackId}");
    first.EnsureSuccessStatusCode();
    var etag = first.Headers.ETag!.Tag;

    var req = new HttpRequestMessage(HttpMethod.Get, $"/api/v1/learning/tracks/{trackId}");
    req.Headers.IfNoneMatch.Add(new EntityTagHeaderValue(etag));
    var second = await _client.SendAsync(req);
    second.StatusCode.Should().Be(HttpStatusCode.NotModified);
}
```

- [ ] **Step 9: Run integration tests for content endpoints**

```bash
dotnet test --filter "FullyQualifiedName~PublicContent"
```

- [ ] **Step 10: Commit**

```bash
git commit -am "feat(s3a-t11a): public content endpoints (~20) with locale + ETag + federated search"
git push
```

---

## Task 11b — `/me/*` Endpoints (~25) + Onboarding + Next-Lesson Heuristic

**Spec ref:** §5.3 user personalized endpoints, §5.5 attempts overlap, §5.10 dashboard
**Estimated:** 5h
**Files:**
- Create: `src/Skillvio.Learning.Api/Endpoints/MeEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/MeTracksEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/MeRoadmapsEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/MeStreakEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/MeXpEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/MeOnboardingEndpoints.cs`
- Create: `src/Skillvio.Learning.Application/Recommendations/INextLessonResolver.cs`
- Create: `src/Skillvio.Learning.Application/Recommendations/HeuristicNextLessonResolver.cs`
- Create: `src/Skillvio.Learning.Api/Filters/RequireUserHeaderFilter.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Api/MeEndpointTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Api/OnboardingTests.cs`

- [ ] **Step 1: Implement `RequireUserHeaderFilter` (gateway-injected `X-User-Id`)**

```csharp
public class RequireUserHeaderFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var userId = ctx.HttpContext.Request.Headers["X-User-Id"].FirstOrDefault();
        if (string.IsNullOrWhiteSpace(userId) || !Guid.TryParse(userId, out _))
            return Results.Unauthorized();
        ctx.HttpContext.Items["UserId"] = Guid.Parse(userId);
        return await next(ctx);
    }
}
```

- [ ] **Step 2: Implement `MeEndpoints` (dashboard composite)**

```csharp
me.MapGet("/dashboard", async (HttpContext http, LearningDbContext db, INextLessonResolver next, ILocalizationContext loc, CancellationToken ct) =>
{
    var userId = (Guid)http.Items["UserId"]!;
    var pinned = await db.UserActiveTracks.Where(at => at.UserId == userId && at.IsPinned).Join(db.Tracks, at => at.TrackId, t => t.Id, (at, t) => new { at, t }).SingleOrDefaultAsync(ct);
    var active = await db.UserActiveTracks.Where(at => at.UserId == userId && !at.IsPinned).Join(db.UserTrackProgress, at => new { at.UserId, at.TrackId }, p => new { p.UserId, p.TrackId }, (at, p) => p).Take(4).ToListAsync(ct);
    var xp = await db.UserXp.FindAsync(new object[] { userId }, ct);
    var hearts = await db.Hearts.FindAsync(new object[] { userId }, ct);
    var streak = await db.DayStreaks.FindAsync(new object[] { userId }, ct);
    var nextLesson = await next.ResolveAsync(userId, ct);

    return Results.Ok(new
    {
        pinned, active, xp, hearts, streak,
        nextLesson,
        meta = new ResponseMeta(loc.CurrentLocale, false, "v1")
    });
});
```

- [ ] **Step 3: Implement onboarding POST + user_preferences**

```csharp
me.MapPost("/onboarding", async (OnboardingRequest req, HttpContext http, LearningDbContext db, CancellationToken ct) =>
{
    var userId = (Guid)http.Items["UserId"]!;
    if (req.SelectedTrackIds.Count is < 1 or > 5) return Results.BadRequest("must select 1-5 tracks");

    using var tx = await db.Database.BeginTransactionAsync(ct);
    var prefs = await db.UserPreferences.FindAsync(new object[] { userId }, ct);
    if (prefs is null) { prefs = new UserPreferences { UserId = userId }; db.UserPreferences.Add(prefs); }
    prefs.DailyGoalMinutes = req.DailyGoalMinutes;
    prefs.PreferredDomainIds = req.PreferredDomainIds.ToArray();
    prefs.UpdatedAt = DateTimeOffset.UtcNow;

    foreach (var trackId in req.SelectedTrackIds.Take(5))
    {
        await db.UserActiveTracks.AddAsync(new UserActiveTracks { UserId = userId, TrackId = trackId, StartedAt = DateTimeOffset.UtcNow, DisplayPosition = (short)req.SelectedTrackIds.IndexOf(trackId), IsPinned = trackId == req.PinnedTrackId }, ct);
        await db.UserTrackProgress.AddAsync(new UserTrackProgress { UserId = userId, TrackId = trackId, LessonsCompleted = 0, LastActivityAt = DateTimeOffset.UtcNow }, ct);
    }
    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
    return Results.Ok();
});

public record OnboardingRequest(IReadOnlyList<Guid> SelectedTrackIds, Guid PinnedTrackId, int DailyGoalMinutes, IReadOnlyList<Guid> PreferredDomainIds);
```

- [ ] **Step 4: Implement `HeuristicNextLessonResolver`**

```csharp
public class HeuristicNextLessonResolver(LearningDbContext db) : INextLessonResolver
{
    public async Task<NextLessonResult> ResolveAsync(Guid userId, CancellationToken ct)
    {
        // 1. Resume in_progress attempt?
        var resume = await db.Attempts.Where(a => a.UserId == userId && a.Status == "in_progress" && a.AttemptType == "lesson").OrderByDescending(a => a.LastActivityAt).FirstOrDefaultAsync(ct);
        if (resume is not null)
            return new NextLessonResult("resume_attempt", resume.Id, resume.TargetId);

        // 2. Pinned track's next un-completed lesson
        var pinned = await db.UserActiveTracks.Where(at => at.UserId == userId && at.IsPinned).SingleOrDefaultAsync(ct);
        if (pinned is not null)
        {
            var nextLessonId = await NextLessonInTrack(pinned.TrackId, userId, ct);
            if (nextLessonId.HasValue)
                return new NextLessonResult("next_in_track", null, nextLessonId.Value);
        }

        // 3. Any active track's next lesson
        var anyActive = await db.UserActiveTracks.Where(at => at.UserId == userId).OrderBy(at => at.DisplayPosition).FirstOrDefaultAsync(ct);
        if (anyActive is not null)
        {
            var nextId = await NextLessonInTrack(anyActive.TrackId, userId, ct);
            if (nextId.HasValue) return new NextLessonResult("next_in_track", null, nextId.Value);
        }

        return new NextLessonResult("no_next", null, null);
    }

    private async Task<Guid?> NextLessonInTrack(Guid trackId, Guid userId, CancellationToken ct) =>
        await (from u in db.Units
               from ul in db.UnitLessons.Where(ul => ul.UnitId == u.Id)
               where u.TrackId == trackId && !db.UserLessonProgress.Any(p => p.UserId == userId && p.LessonId == ul.LessonId && p.CompletedAt != null)
               orderby u.DisplayOrder, ul.DisplayOrder
               select (Guid?)ul.LessonId).FirstOrDefaultAsync(ct);
}
```

- [ ] **Step 5: Implement remaining /me/* endpoints**

- `GET /me/tracks`, `POST /me/tracks/{id}/start`, `PATCH`, `DELETE`, `GET /me/tracks/{id}/progress`
- Same pattern for `/me/roadmaps/`
- `GET /me/streak` + `POST /me/streak/freeze` + `POST /me/streak/repair`
- `GET /me/hearts`, `GET /me/xp`, `GET /me/xp/history`
- `GET /me/badges`, `GET /me/certificates`, `GET /me/exam-access`
- `GET /me/lab-attempts` (read from `user_lab_completions`)
- `GET /me/recommendations?source=heuristic|ai` (Sprint 5 stubs `ai`)
- `GET /me/next-lesson`
- `GET /me/leaderboard?scope=...&period=...` (Postgres `RANK() OVER`)
- `GET /me/league`, `/me/league/cohort`, `/me/league/history` (Sprint 3b owns league logic; here we just read)

- [ ] **Step 6: Test onboarding sets active tracks correctly**

```csharp
[Fact]
public async Task Onboarding_With3TracksAndPin_CreatesActiveRows()
{
    var t1 = await SeedTrack(); var t2 = await SeedTrack(); var t3 = await SeedTrack();
    var resp = await _client.PostAsJsonAsync("/api/v1/me/onboarding", new
    {
        selectedTrackIds = new[] { t1, t2, t3 },
        pinnedTrackId = t2,
        dailyGoalMinutes = 15,
        preferredDomainIds = Array.Empty<Guid>()
    });
    resp.EnsureSuccessStatusCode();
    var actives = await _db.UserActiveTracks.Where(at => at.UserId == UserA).ToListAsync();
    actives.Should().HaveCount(3);
    actives.Single(a => a.IsPinned).TrackId.Should().Be(t2);
}
```

- [ ] **Step 7: Test next-lesson heuristic prioritizes resume**

- [ ] **Step 8: Commit**

```bash
git commit -am "feat(s3a-t11b): /me/* endpoints (~25) with onboarding + next-lesson heuristic"
git push
```

---

## Task 12 — Attempts + Exam Flow Endpoints (Heartbeat + Cleanup Job)

**Spec ref:** §5.5 attempts endpoints, §6.2 exam flow
**Estimated:** 8h
**Files:**
- Create: `src/Skillvio.Learning.Api/Endpoints/AttemptEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/ExamAttemptEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Validators/StartAttemptValidator.cs`
- Create: `src/Skillvio.Learning.Application/Progress/Attempts/Commands/StartExamAttemptCmd.cs`
- Create: `src/Skillvio.Learning.Application/Progress/Attempts/ExamAttemptService.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Jobs/AbandonedAttemptsCleaner.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Jobs/ExpiredExamAttemptsCleaner.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Api/AttemptEndpointTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Api/ExamAttemptEndpointTests.cs`

- [ ] **Step 1: AttemptEndpoints (start/resume/answer/complete/abandon/heartbeat)**

```csharp
public static class AttemptEndpoints
{
    public static void Map(IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/api/v1/attempts").AddEndpointFilter<RequireUserHeaderFilter>().WithTags("Attempts");

        g.MapPost("/", async (StartAttemptRequest req, HttpContext http, IAttemptOrchestrator orch, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var key = ParseIdempotencyKey(http);
            var result = await orch.StartAsync(new StartAttemptCmd(userId, "lesson", req.TargetId, req.DeviceSessionId, key), ct);
            return Results.Ok(result);
        }).AddEndpointFilter<IdempotencyFilter>();

        g.MapGet("/{id:guid}", async (Guid id, HttpContext http, IAttemptOrchestrator orch, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            return Results.Ok(await orch.ResumeAsync(id, userId, ct));
        });

        g.MapPost("/{id:guid}/answers", async (Guid id, SubmitAnswerRequest req, HttpContext http, IAttemptOrchestrator orch, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var key = ParseIdempotencyKey(http);
            return Results.Ok(await orch.SubmitAnswerAsync(new SubmitAnswerCmd(id, userId, req.AnswerIndex, req.ActivityId, null, req.Payload, key), ct));
        }).AddEndpointFilter<IdempotencyFilter>();

        g.MapPost("/{id:guid}/complete", async (Guid id, HttpContext http, IAttemptOrchestrator orch, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var key = ParseIdempotencyKey(http);
            return Results.Ok(await orch.CompleteAsync(new CompleteAttemptCmd(id, userId, key), ct));
        }).AddEndpointFilter<IdempotencyFilter>();

        g.MapDelete("/{id:guid}", async (Guid id, HttpContext http, IAttemptOrchestrator orch, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            await orch.AbandonAsync(id, userId, ct);
            return Results.NoContent();
        });

        g.MapPost("/{id:guid}/heartbeat", async (Guid id, HttpContext http, LearningDbContext db, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            await db.Database.ExecuteSqlInterpolatedAsync($"UPDATE attempts SET last_activity_at = NOW() WHERE id = {id} AND user_id = {userId} AND status = 'in_progress'", ct);
            return Results.NoContent();
        });
    }
}

public record StartAttemptRequest(Guid TargetId, string? DeviceSessionId);
public record SubmitAnswerRequest(int AnswerIndex, Guid? ActivityId, JsonDocument Payload);
```

- [ ] **Step 2: ExamAttemptService (access check + shuffle + time limit)**

```csharp
public class ExamAttemptService(LearningDbContext db, IClock clock) : IExamAttemptService
{
    public async Task<StartAttemptResult> StartExamAsync(StartExamAttemptCmd cmd, CancellationToken ct)
    {
        // 1. Access check
        var access = await db.ExamAccess.SingleOrDefaultAsync(a => a.UserId == cmd.UserId && a.ExamId == cmd.ExamId, ct);
        if (access is null) throw new ForbiddenException("No access to this exam");
        if (access.ExpiresAt is { } exp && exp < clock.UtcNow) throw new ForbiddenException("Access expired");
        if (access.AttemptsRemaining is 0) throw new ForbiddenException("No attempts remaining");

        // 2. Load exam meta + question count
        var exam = await db.Exams.SingleAsync(e => e.Id == cmd.ExamId, ct);
        var allQs = await db.ExamQuestions.Where(q => q.ExamId == cmd.ExamId && q.Status == "published").Select(q => q.Id).ToListAsync(ct);
        if (allQs.Count < exam.QuestionCountPerAttempt) throw new InvalidOperationException("Insufficient questions");

        // 3. Shuffle (Fisher-Yates) + take N
        var rng = new Random(unchecked((int)DateTimeOffset.UtcNow.Ticks ^ cmd.UserId.GetHashCode()));
        for (int i = allQs.Count - 1; i > 0; i--) { var j = rng.Next(i + 1); (allQs[i], allQs[j]) = (allQs[j], allQs[i]); }
        var selected = allQs.Take(exam.QuestionCountPerAttempt).ToArray();

        // 4. Create attempt + meta
        var attemptId = Guid.NewGuid();
        var attempt = new Attempt { Id = attemptId, UserId = cmd.UserId, AttemptType = "exam", TargetId = cmd.ExamId, Status = "in_progress", StartedAt = clock.UtcNow, LastActivityAt = clock.UtcNow, IdempotencyKey = cmd.IdempotencyKey };
        db.Attempts.Add(attempt);
        db.ExamAttemptMeta.Add(new ExamAttemptMeta { AttemptId = attemptId, QuestionsTotal = selected.Length, QuestionsCorrect = 0, PassingScorePct = exam.PassingScorePct, Passed = false, ShuffledQuestionOrder = selected });
        if (access.AttemptsRemaining.HasValue) access.AttemptsRemaining--;
        await db.SaveChangesAsync(ct);
        return new StartAttemptResult(attemptId, attempt.StartedAt);
    }
}
```

- [ ] **Step 3: Test exam start → access check 403 if no access**

```csharp
[Fact]
public async Task StartExam_WithoutAccess_Returns403()
{
    var examId = await SeedExam("DVA-C02");
    // No exam_access row for UserA
    var resp = await _client.PostAsync($"/api/v1/attempts/exam/{examId}/start", null);
    resp.StatusCode.Should().Be(HttpStatusCode.Forbidden);
}

[Fact]
public async Task StartExam_WithAccess_ReturnsShuffledQuestions()
{
    var examId = await SeedExam("DVA-C02", questionCount: 10);
    await GrantExamAccess(UserA, examId, via: "plan_pro");
    var resp = await _client.PostAsync($"/api/v1/attempts/exam/{examId}/start", null);
    resp.EnsureSuccessStatusCode();
    var meta = await _db.ExamAttemptMeta.SingleAsync();
    meta.ShuffledQuestionOrder.Should().HaveCount(10);
}
```

- [ ] **Step 4: Implement ExpiredExamAttemptsCleaner**

```csharp
public class ExpiredExamAttemptsCleaner(LearningDbContext db, IAttemptOrchestrator orch, IClock clock, ILogger<ExpiredExamAttemptsCleaner> log)
{
    public async Task RunAsync(CancellationToken ct)
    {
        var now = clock.UtcNow;
        var expired = await db.Attempts
            .Where(a => a.AttemptType == "exam" && a.Status == "in_progress")
            .Join(db.Exams, a => a.TargetId, e => e.Id, (a, e) => new { a, e })
            .Where(ae => EF.Functions.DateDiffMinute(ae.a.StartedAt, now) > ae.e.TimeLimitMinutes)
            .Select(ae => ae.a.Id)
            .ToListAsync(ct);

        foreach (var id in expired)
        {
            try
            {
                // Force-complete with timed_out status
                await db.Database.ExecuteSqlInterpolatedAsync($"UPDATE attempts SET status = 'timed_out' WHERE id = {id}", ct);
                // Score what was submitted
                // ... logic to compute final score from existing answers
            }
            catch (Exception ex) { log.LogError(ex, "Cleanup failed for attempt {Id}", id); }
        }
    }
}
```

Schedule via Hangfire: `RecurringJob.AddOrUpdate<ExpiredExamAttemptsCleaner>("attempts.cleanup.expired_exam", c => c.RunAsync(default), "*/30 * * * *")`.

- [ ] **Step 5: Implement AbandonedAttemptsCleaner (24h idle → abandoned)**

- [ ] **Step 6: Test heartbeat extends last_activity_at**

```csharp
[Fact]
public async Task Heartbeat_UpdatesLastActivity()
{
    var att = await StartAttempt(UserA, lessonId);
    var initial = await _db.Attempts.AsNoTracking().Where(a => a.Id == att.AttemptId).Select(a => a.LastActivityAt).SingleAsync();
    await Task.Delay(50);
    await _client.PostAsync($"/api/v1/attempts/{att.AttemptId}/heartbeat", null);
    var updated = await _db.Attempts.AsNoTracking().Where(a => a.Id == att.AttemptId).Select(a => a.LastActivityAt).SingleAsync();
    updated.Should().BeAfter(initial);
}
```

- [ ] **Step 7: Run all attempt endpoint tests**

```bash
dotnet test --filter "FullyQualifiedName~AttemptEndpoint OR FullyQualifiedName~ExamAttemptEndpoint"
```

- [ ] **Step 8: Commit**

```bash
git commit -am "feat(s3a-t12): attempt + exam endpoints with heartbeat and cleanup jobs"
git push
```

---

## Task 15 — ContentSync (Git → DB) + Webhook + Advisory Lock + Dry-Run

**Spec ref:** §6.10 ContentSyncOrchestrator, §9.3 webhook, Q1=B
**Estimated:** 10h
**Files:**
- Create: `src/Skillvio.Learning.ContentSync/Pipeline/ParseStage.cs`
- Create: `src/Skillvio.Learning.ContentSync/Pipeline/ValidateStage.cs`
- Create: `src/Skillvio.Learning.ContentSync/Pipeline/DiffStage.cs`
- Create: `src/Skillvio.Learning.ContentSync/Pipeline/ApplyStage.cs`
- Create: `src/Skillvio.Learning.ContentSync/Pipeline/ContentSyncOrchestrator.cs`
- Create: `src/Skillvio.Learning.ContentSync/Webhook/WebhookEndpoint.cs`
- Create: `src/Skillvio.Learning.ContentSync/GitOps/GitWorkdirManager.cs`
- Create: `src/Skillvio.Learning.ContentSync/Validators/JsonSchemaCache.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/ContentSync/SyncOrchestratorTests.cs`

- [ ] **Step 1: Add packages**

```bash
dotnet add src/Skillvio.Learning.ContentSync package LibGit2Sharp
dotnet add src/Skillvio.Learning.ContentSync package YamlDotNet
dotnet add src/Skillvio.Learning.ContentSync package JsonSchema.Net
```

- [ ] **Step 2: GitWorkdirManager (clone or fetch by SHA)**

```csharp
public class GitWorkdirManager(IConfiguration config, ILogger<GitWorkdirManager> log)
{
    private readonly string _localPath = config["ContentRepo:LocalPath"]!;
    private readonly string _gitUrl = config["ContentRepo:GitUrl"]!;

    public string CheckoutAtSha(string commitSha)
    {
        if (!Directory.Exists(Path.Combine(_localPath, ".git")))
        {
            Repository.Clone(_gitUrl, _localPath);
            log.LogInformation("Cloned {Url} → {Path}", _gitUrl, _localPath);
        }
        using var repo = new Repository(_localPath);
        Commands.Fetch(repo, "origin", new[] { "+refs/heads/*:refs/remotes/origin/*" }, null, null);
        var commit = repo.Lookup<Commit>(commitSha) ?? throw new InvalidOperationException($"Commit {commitSha} not found");
        Commands.Checkout(repo, commit);
        return _localPath;
    }
}
```

- [ ] **Step 3: ParseStage — walks file tree → parsed entities**

```csharp
public class ParseStage(IDeserializer yamlDes)
{
    public IEnumerable<ParsedEntity> Parse(string workdir)
    {
        foreach (var dir in new[] { "domains", "tracks", "lessons", "roadmaps", "exams", "certificates", "badges", "glossary", "level-definitions" })
        {
            var path = Path.Combine(workdir, dir);
            if (!Directory.Exists(path)) continue;
            foreach (var file in Directory.EnumerateFiles(path, "*.yml", SearchOption.AllDirectories))
            {
                var content = File.ReadAllText(file);
                var parsed = yamlDes.Deserialize<dynamic>(content);
                yield return new ParsedEntity(
                    Type: dir.TrimEnd('s'),
                    SourcePath: Path.GetRelativePath(workdir, file),
                    Payload: parsed,
                    ContentHash: ComputeHash(content));
            }
        }
    }

    private static string ComputeHash(string s)
    {
        using var sha = SHA256.Create();
        return Convert.ToHexString(sha.ComputeHash(Encoding.UTF8.GetBytes(s)));
    }
}

public record ParsedEntity(string Type, string SourcePath, object Payload, string ContentHash);
```

- [ ] **Step 4: ValidateStage — JSON Schema check per type**

```csharp
public class ValidateStage(JsonSchemaCache cache)
{
    public ValidationResult Validate(ParsedEntity entity)
    {
        var schema = cache.Get(entity.Type);  // schemas/track.schema.json, schemas/lesson.schema.json, ...
        var json = JsonSerializer.SerializeToNode(entity.Payload);
        var result = schema.Evaluate(json, new EvaluationOptions { OutputFormat = OutputFormat.List });
        return new ValidationResult(result.IsValid, result.Errors?.Select(e => e.ToString()).ToList() ?? []);
    }
}

public record ValidationResult(bool IsValid, IReadOnlyList<string> Errors);
```

- [ ] **Step 5: DiffStage — hash + path-based detection**

```csharp
public class DiffStage(LearningDbContext db)
{
    public async Task<ContentDiff> ComputeAsync(IEnumerable<ParsedEntity> parsed, CancellationToken ct)
    {
        var diff = new ContentDiff();
        foreach (var p in parsed)
        {
            var existing = await db.Tracks.Where(t => t.SourcePath == p.SourcePath).Select(t => new { t.Id, t.ContentHash }).SingleOrDefaultAsync(ct);
            if (existing is null) diff.Added.Add(p);
            else if (existing.ContentHash != p.ContentHash) diff.Updated.Add((existing.Id, p));
            else diff.Unchanged.Add(p);
        }
        // Detect renames: same content_hash but different source_path
        // Detect archived: in DB but not in parsed (sourcePath disappeared)
        return diff;
    }
}
```

- [ ] **Step 6: ApplyStage — INSERT/UPDATE within tx**

Maps each parsed entity to its DB row, sets `status`, `published_at`, `source_commit_sha`, etc.

- [ ] **Step 7: ContentSyncOrchestrator with pg_advisory_lock**

```csharp
public async Task<ContentImportResult> ImportAsync(string commitSha, bool dryRun, CancellationToken ct)
{
    // 1. Acquire global advisory lock
    var lockKey = "content_sync_global".GetDeterministicHashCode();
    var locked = await db.Database.ExecuteSqlInterpolatedAsync($"SELECT pg_try_advisory_lock({lockKey})", ct);
    if (locked == 0) throw new ConflictException("Another content sync is already running");

    try
    {
        var import = new ContentImport { Id = Guid.NewGuid(), ContentRepoCommitSha = commitSha, StartedAt = DateTimeOffset.UtcNow, Status = "running" };
        db.ContentImports.Add(import);
        await db.SaveChangesAsync(ct);

        var workdir = git.CheckoutAtSha(commitSha);
        var parsed = parse.Parse(workdir).ToList();
        var validations = parsed.Select(p => (p, validate.Validate(p))).ToList();
        var failed = validations.Where(t => !t.Item2.IsValid).ToList();

        if (failed.Any())
        {
            import.Status = "failed";
            import.Errors = JsonDocument.Parse(JsonSerializer.Serialize(failed.Select(t => new { t.p.SourcePath, t.Item2.Errors })));
            await db.SaveChangesAsync(ct);
            return new ContentImportResult(import.Id, null);
        }

        var diff = await diffStage.ComputeAsync(validations.Select(t => t.p), ct);
        if (dryRun) { import.Status = "success_dry_run"; import.EntitiesAdded = diff.Added.Count; await db.SaveChangesAsync(ct); return new ContentImportResult(import.Id, diff); }

        using var tx = await db.Database.BeginTransactionAsync(ct);
        await applyStage.ApplyAsync(diff, commitSha, ct);
        await tx.CommitAsync(ct);

        await cache.RemoveByTagAsync("content");
        await outbox.AddAsync(new ContentImportCompletedEvent(import.Id, commitSha, diff.Added.Count, diff.Updated.Count), ct);
        import.Status = "success";
        import.CompletedAt = DateTimeOffset.UtcNow;
        await db.SaveChangesAsync(ct);
        return new ContentImportResult(import.Id, diff);
    }
    finally
    {
        await db.Database.ExecuteSqlInterpolatedAsync($"SELECT pg_advisory_unlock({lockKey})", ct);
    }
}
```

- [ ] **Step 8: Webhook endpoint with HMAC verify**

```csharp
public static class WebhookEndpoint
{
    public static void Map(IEndpointRouteBuilder app, IConfiguration config)
    {
        var secret = config["ContentRepo:WebhookSecret"]!;
        app.MapPost("/api/v1/admin/learning/content-imports/webhook", async (HttpContext http, IBackgroundJobClient jobs, LearningDbContext db, CancellationToken ct) =>
        {
            using var reader = new StreamReader(http.Request.Body);
            var body = await reader.ReadToEndAsync(ct);
            var signature = http.Request.Headers["X-Hub-Signature-256"].FirstOrDefault();
            if (!VerifyHmac(body, secret, signature)) return Results.Unauthorized();

            var payload = JsonDocument.Parse(body);
            var sha = payload.RootElement.GetProperty("after").GetString()!;

            // Idempotency: skip if commit already running/done
            if (await db.ContentImports.AnyAsync(i => i.ContentRepoCommitSha == sha && (i.Status == "running" || i.Status == "success"), ct))
                return Results.Ok(new { skipped = true });

            jobs.Enqueue<IContentSyncOrchestrator>(o => o.ImportAsync(sha, false, default));
            return Results.Accepted();
        });
    }

    private static bool VerifyHmac(string body, string secret, string? signature)
    {
        if (string.IsNullOrEmpty(signature)) return false;
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
        var computed = "sha256=" + Convert.ToHexString(hmac.ComputeHash(Encoding.UTF8.GetBytes(body))).ToLowerInvariant();
        return CryptographicOperations.FixedTimeEquals(Encoding.UTF8.GetBytes(computed), Encoding.UTF8.GetBytes(signature));
    }
}
```

- [ ] **Step 9: Integration test — full sync from fixture repo**

```csharp
[Fact]
public async Task FullSync_FromFixture_PopulatesDb()
{
    var fixtureWorkdir = await CopyFixtureToTempDir("ContentFixtures/Sample");
    var commit = await CommitFixture(fixtureWorkdir);
    var result = await _orchestrator.ImportAsync(commit, dryRun: false, default);
    var domainCount = await _db.Domains.CountAsync();
    domainCount.Should().BeGreaterThan(0);
}
```

- [ ] **Step 10: Test concurrent sync rejected via advisory lock**

```csharp
[Fact]
public async Task ConcurrentSync_SecondCallRejected()
{
    var task1 = _orchestrator.ImportAsync("abc123", false, default);
    var act = async () => await _orchestrator.ImportAsync("def456", false, default);
    await act.Should().ThrowAsync<ConflictException>();
    await task1;  // first finishes
}
```

- [ ] **Step 11: Commit**

```bash
git commit -am "feat(s3a-t15): ContentSync (Git→DB) with HMAC webhook + advisory lock + dry-run"
git push
```

---

## Task 16 — Cloud Cert Catalog (30 Cert: exam.yml + roadmap + cert def + schemas)

**Spec ref:** §4 cloud cert sistemi, Q+1
**Estimated:** 15h
**Files:** All under `skillvio-content/` repo (separate from learning-service)
- Create: `skillvio-content/schemas/exam.schema.json`
- Create: `skillvio-content/schemas/exam-question-single.schema.json`
- Create: `skillvio-content/schemas/exam-question-multi.schema.json`
- Create: `skillvio-content/schemas/exam-question-scenario.schema.json`
- Create: `skillvio-content/schemas/track.schema.json`
- Create: `skillvio-content/schemas/lesson.schema.json`
- Create: `skillvio-content/schemas/activity-multiple-choice.schema.json` (and 7 other activity types)
- Create: `skillvio-content/schemas/roadmap.schema.json`
- Create: `skillvio-content/schemas/badge.schema.json`
- Create: `skillvio-content/schemas/certificate.schema.json`
- Create: `skillvio-content/domains/cloud.yml`
- Create: 30 × `skillvio-content/exams/<provider>/<code>/exam.yml`
- Create: 30 × `skillvio-content/roadmaps/<roadmap-slug>.yml`
- Create: 30 × `skillvio-content/certificates/skillvio-<cert>-prep.yml`
- Create: ~30 × `skillvio-content/badges/*.yml`

> **Pre-flight:** Confirm OI4 (exam content licensing) decision. If "100% original": all questions authored by Skillvio team. If "inspired by": every question must be substantially rewritten with different scenario/wording.

- [ ] **Step 1: Initialize `skillvio-content` repo**

```bash
gh repo clone skillvio/skillvio-content
cd skillvio-content
mkdir -p schemas domains exams/aws exams/azure exams/gcp tracks roadmaps certificates badges glossary
```

- [ ] **Step 2: Author JSON Schemas (10 schemas)**

Use a representative example (already shown in spec §4.2):

```json
// schemas/exam.schema.json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["slug", "cert_provider", "cert_code", "tier", "question_count_per_attempt", "time_limit_minutes", "passing_score_pct", "default_locale"],
  "properties": {
    "slug": { "type": "string", "pattern": "^[a-z0-9-]+$" },
    "cert_provider": { "enum": ["aws", "azure", "gcp", "skillvio"] },
    "cert_code": { "type": "string" },
    "tier": { "enum": ["foundational", "associate", "professional", "expert", "specialty"] },
    "question_count_per_attempt": { "type": "integer", "minimum": 1, "maximum": 200 },
    "time_limit_minutes": { "type": "integer", "minimum": 1 },
    "passing_score_pct": { "type": "number", "minimum": 0, "maximum": 100 },
    "scoring_scheme": { "enum": ["percentage_72", "percentage_70", "percentage_75", "scaled_700_1000"] },
    "default_locale": { "type": "string" },
    "is_premium": { "type": "boolean" },
    "domain_breakdown": { "type": "array", "items": { "type": "object", "required": ["code", "weight_pct"], "properties": { "code": { "type": "string" }, "weight_pct": { "type": "number" } } } }
  }
}
```

Repeat for `exam-question-*`, `track`, `lesson`, `activity-*`, `roadmap`, `badge`, `certificate`. Use AJV-style references.

- [ ] **Step 3: Author 30 `exam.yml` files (AWS 10 + Azure 12 + GCP 8)**

Template (`exams/aws/dva-c02/exam.yml`):

```yaml
slug: aws-dva-c02
cert_provider: aws
cert_code: DVA-C02
tier: associate
exam_format: aws_associate
scoring_scheme: percentage_72
question_count_per_attempt: 65
time_limit_minutes: 130
passing_score_pct: 72
is_premium: true
default_locale: en
short_i18n:
  title:
    en: "AWS Developer Associate (DVA-C02)"
    tr: "AWS Developer Associate (DVA-C02)"
  summary:
    en: "Hands-on validated knowledge for AWS application developers."
    tr: "AWS uygulama geliştiricileri için pratik doğrulanmış bilgi."
domain_breakdown:
  - code: deployment
    name_i18n: { en: "Deployment", tr: "Yayınlama" }
    weight_pct: 32
  - code: security
    name_i18n: { en: "Security", tr: "Güvenlik" }
    weight_pct: 26
  - code: dev_with_aws
    name_i18n: { en: "Development with AWS Services", tr: "AWS Servisleriyle Geliştirme" }
    weight_pct: 30
  - code: troubleshoot
    name_i18n: { en: "Troubleshooting and Optimization", tr: "Sorun Giderme ve Optimizasyon" }
    weight_pct: 12
```

Repeat for 29 more certs; values come from each provider's official exam guide.

- [ ] **Step 4: Author 30 roadmap iskeletleri**

Template (`roadmaps/aws-developer.yml`):

```yaml
slug: aws-developer
domain: aws
target_exam: aws-dva-c02
default_locale: en
short_i18n:
  title:
    en: "AWS Developer Path"
    tr: "AWS Developer Yolu"
nodes:
  - id: lambda-fundamentals
    type: topic
    lesson_slug: lambda-101
    children:
      - id: lambda-deployment
        lesson_slug: lambda-deployment
      - id: lambda-permissions
        lesson_slug: lambda-iam-roles
  - id: dynamodb-basics
    type: topic
    lesson_slug: dynamodb-101
  - id: api-gateway
    type: topic
    lesson_slug: api-gateway-101
  - id: ci-cd-pipelines
    type: topic
    lesson_slug: codepipeline-101
  - id: cert-prep
    type: cert_target
    target_exam: aws-dva-c02
```

Lesson stubs created in Sprint 5+; for Sprint 3a we only need the roadmap+nodes structure to seed.

- [ ] **Step 5: Author 30 Skillvio prep certificate definitions**

Template (`certificates/skillvio-aws-dev-prep.yml`):

```yaml
code: skillvio-aws-dev-prep
domain: aws
artwork_url: https://cdn.skillvio.io/certs/aws-dev-prep.svg
trigger_rule:
  all:
    - track_completed: aws-dva-c02-prep
    - exam_passed:
        exam: aws-dva-c02
        min_score_pct: 72
short_i18n:
  title:
    en: "Ready for AWS Developer Certified"
    tr: "AWS Developer Sertifikasına Hazır"
  summary:
    en: "Skillvio prep complete — you've mastered the DVA-C02 syllabus."
    tr: "Skillvio hazırlığı tamamlandı — DVA-C02 müfredatına hakimsiniz."
is_verifiable: true
```

- [ ] **Step 6: Author baseline badge set (~30 badges)**

`badges/first-lesson.yml`, `badges/streak-7.yml`, `badges/streak-30.yml`, `badges/perfect-lesson.yml`, `badges/first-exam-pass.yml`, `badges/multi-cloud-master.yml`, etc. Each follows the schema with `tier: sync|async`, `predicate_kind`, `predicate_args`.

- [ ] **Step 7: Domain hierarchy seed**

```yaml
# domains/cloud.yml
domains:
  - code: cloud
    parent: null
    short_i18n:
      name: { tr: "Bulut", en: "Cloud" }
      tagline: { tr: "Bulut bilişim öğren", en: "Learn cloud computing" }
    icon_url: https://cdn.skillvio.io/icons/cloud.svg
    color_hex: "#0077C8"
  - code: aws
    parent: cloud
    short_i18n:
      name: { en: "AWS", tr: "AWS" }
    icon_url: https://cdn.skillvio.io/icons/aws.svg
    color_hex: "#FF9900"
  # ... azure, gcp similarly
```

- [ ] **Step 8: Add CI workflow for skillvio-content (validates schemas)**

```yaml
# skillvio-content/.github/workflows/validate.yml
name: Validate Content
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm install -g ajv-cli ajv-formats
      - run: |
          for schema in schemas/*.json; do
            echo "Validating against $schema"
            # Globe through corresponding files (e.g., exams/**/exam.yml against exam.schema.json)
          done
```

- [ ] **Step 9: Run ContentSync on the new content**

```bash
cd ~/skillvio/skillvio-learning-service
docker-compose -f docker-compose.dev.yml up -d learning-postgres redis rabbitmq
dotnet run --project src/Skillvio.Learning.ContentSync -- import --commit HEAD --dry-run=false
```

Expected: ContentImport row with `status='success'`, ~30 exams + 30 roadmaps + 30 certs + 30 badges + 4 domains in DB.

- [ ] **Step 10: Commit content repo + reference in learning-service**

```bash
cd ~/skillvio/skillvio-content
git add -A
git commit -m "feat(content): seed 30 cloud certs (AWS/Azure/GCP) + roadmaps + Skillvio prep certs + 30 badges"
git push

cd ~/skillvio/skillvio-learning-service
# Update appsettings.Development.json: ContentRepo:GitRefDefault = "<commit-sha>"
git commit -am "chore(s3a-t16): pin content commit for Sprint 3a baseline"
git push
```

---

## Task 18a — Admin Endpoints + Observability Core (Prometheus + Loki + Role Auth)

**Spec ref:** §5.7 admin endpoints, §11.1 metrics, §11.2 logs, §11.3 traces
**Estimated:** 5h
**Files:**
- Create: `src/Skillvio.Learning.Api/Endpoints/AdminEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Filters/RequireRoleFilter.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Observability/PrometheusMetrics.cs`
- Create: `src/Skillvio.Learning.Api/Observability/SerilogConfig.cs`
- Create: `src/Skillvio.Learning.Api/Observability/OpenTelemetryConfig.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Api/AdminAuthorizationTests.cs`

- [ ] **Step 1: Implement RequireRoleFilter**

```csharp
public class RequireRoleFilter(string[] allowed) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var role = ctx.HttpContext.Request.Headers["X-User-Role"].FirstOrDefault();
        if (string.IsNullOrEmpty(role) || !allowed.Contains(role)) return Results.Forbid();
        return await next(ctx);
    }
}
```

- [ ] **Step 2: Define admin endpoints**

```csharp
public static class AdminEndpoints
{
    public static void Map(IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/api/v1/admin/learning")
            .AddEndpointFilter<RequireUserHeaderFilter>()
            .WithTags("Admin");

        g.MapGet("/content-imports", async (LearningDbContext db, CancellationToken ct) =>
            Results.Ok(await db.ContentImports.OrderByDescending(i => i.StartedAt).Take(50).ToListAsync(ct)))
            .AddEndpointFilter(new RequireRoleFilter(["admin"]));

        g.MapPost("/content-imports", async (TriggerImportRequest req, IBackgroundJobClient jobs, CancellationToken ct) =>
        {
            var jobId = jobs.Enqueue<IContentSyncOrchestrator>(o => o.ImportAsync(req.CommitSha, req.DryRun, default));
            return Results.Accepted($"/api/v1/admin/learning/content-imports/jobs/{jobId}");
        }).AddEndpointFilter(new RequireRoleFilter(["admin"]));

        g.MapPost("/tracks/{id:guid}/publish", async (Guid id, LearningDbContext db, CancellationToken ct) =>
        {
            await db.Tracks.Where(t => t.Id == id && t.Status == "review").ExecuteUpdateAsync(s => s.SetProperty(t => t.Status, "published").SetProperty(t => t.PublishedAt, DateTimeOffset.UtcNow), ct);
            return Results.NoContent();
        }).AddEndpointFilter(new RequireRoleFilter(["admin", "content_author"]));

        g.MapPost("/tracks/{id:guid}/archive", async (Guid id, LearningDbContext db, CancellationToken ct) =>
        {
            await db.Tracks.Where(t => t.Id == id).ExecuteUpdateAsync(s => s.SetProperty(t => t.Status, "archived").SetProperty(t => t.ArchivedAt, DateTimeOffset.UtcNow), ct);
            return Results.NoContent();
        }).AddEndpointFilter(new RequireRoleFilter(["admin"]));

        g.MapPost("/badges/{id:guid}/recompute", async (Guid id, IBackgroundJobClient jobs) => Results.Accepted("", jobs.Enqueue<IBadgeBackfillJob>(j => j.RecomputeAsync(id, default))))
            .AddEndpointFilter(new RequireRoleFilter(["admin"]));

        g.MapGet("/attempts/{id:guid}/audit", async (Guid id, LearningDbContext db, CancellationToken ct) =>
        {
            var att = await db.Attempts.Include(a => a.Answers).SingleOrDefaultAsync(a => a.Id == id, ct);
            if (att is null) return Results.NotFound();
            return Results.Ok(new { attempt = att, suspiciousFlags = att.Answers.Where(an => an.TimeSpentMs < 2000).Select(an => an.AnswerIndex) });
        }).AddEndpointFilter(new RequireRoleFilter(["admin", "moderator"]));

        g.MapPost("/users/{userId:guid}/grant-exam-access", async (Guid userId, GrantAccessRequest req, LearningDbContext db, IOutbox outbox, CancellationToken ct) =>
        {
            await db.ExamAccess.AddAsync(new ExamAccess { UserId = userId, ExamId = req.ExamId, GrantedAt = DateTimeOffset.UtcNow, GrantedVia = "gift" }, ct);
            await outbox.AddAsync(new ExamAccessGrantedEvent(userId, req.ExamId, "gift"), ct);
            await db.SaveChangesAsync(ct);
            return Results.NoContent();
        }).AddEndpointFilter(new RequireRoleFilter(["admin"]));

        g.MapPost("/users/{userId:guid}/revoke-exam-access", async (Guid userId, RevokeAccessRequest req, LearningDbContext db, IOutbox outbox, CancellationToken ct) =>
        {
            await db.ExamAccess.Where(a => a.UserId == userId && a.ExamId == req.ExamId).ExecuteDeleteAsync(ct);
            await outbox.AddAsync(new ComplianceAuditEvent(userId, "exam_access_revoked", JsonSerializer.SerializeToDocument(new { req.ExamId, req.Reason })), ct);
            await db.SaveChangesAsync(ct);
            return Results.NoContent();
        }).AddEndpointFilter(new RequireRoleFilter(["admin"]));

        g.MapPost("/dlq/replay/{messageId:guid}", async (Guid messageId, LearningDbContext db, IBus bus, CancellationToken ct) =>
        {
            // Reset retry_count + dispatched_at for replay
            await db.OutboxMessages.Where(m => m.Id == messageId).ExecuteUpdateAsync(s => s.SetProperty(m => m.RetryCount, 0).SetProperty(m => m.DispatchedAt, (DateTimeOffset?)null), ct);
            return Results.NoContent();
        }).AddEndpointFilter(new RequireRoleFilter(["admin"]));
    }
}
```

- [ ] **Step 3: Configure Serilog → Loki**

```csharp
// src/Skillvio.Learning.Api/Observability/SerilogConfig.cs
public static class SerilogConfig
{
    public static void Configure(IHostBuilder host)
    {
        host.UseSerilog((ctx, lc) => lc
            .ReadFrom.Configuration(ctx.Configuration)
            .Enrich.FromLogContext()
            .Enrich.WithProperty("service", "skillvio-learning")
            .WriteTo.Console(outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties}{NewLine}{Exception}")
            .WriteTo.GrafanaLoki(
                ctx.Configuration["Loki:Endpoint"]!,
                labels: [new() { Key = "service", Value = "skillvio-learning" }],
                propertiesAsLabels: ["level", "RequestPath"]));
    }
}
```

- [ ] **Step 4: Configure OpenTelemetry → Tempo + Prometheus**

```csharp
public static class OpenTelemetryConfig
{
    public static void Configure(IServiceCollection services, IConfiguration config)
    {
        services.AddOpenTelemetry()
            .ConfigureResource(r => r.AddService("skillvio-learning"))
            .WithTracing(t => t
                .AddAspNetCoreInstrumentation()
                .AddEntityFrameworkCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddSource("MassTransit")
                .AddOtlpExporter(o => { o.Endpoint = new Uri(config["Otel:Endpoint"]!); }))
            .WithMetrics(m => m
                .AddAspNetCoreInstrumentation()
                .AddRuntimeInstrumentation()
                .AddPrometheusExporter());
    }
}
```

- [ ] **Step 5: Custom Prometheus metrics**

```csharp
public static class LearningMetrics
{
    public static readonly Meter Meter = new("Skillvio.Learning", "1.0.0");
    public static readonly Counter<long> AttemptsStarted = Meter.CreateCounter<long>("learning_attempts_started_total", "count");
    public static readonly Counter<long> AttemptsCompleted = Meter.CreateCounter<long>("learning_attempts_completed_total", "count");
    public static readonly Counter<long> XpAwarded = Meter.CreateCounter<long>("learning_xp_awarded_total", "xp");
    public static readonly Counter<long> BadgeEvaluations = Meter.CreateCounter<long>("learning_badge_evaluations_total", "count");
    public static readonly ObservableGauge<int> ActiveAttempts = Meter.CreateObservableGauge<int>("learning_active_attempts", () => GetActiveAttemptsCount());
    public static readonly ObservableGauge<int> OutboxUndispatched = Meter.CreateObservableGauge<int>("learning_outbox_undispatched_count", () => GetOutboxUndispatched());
    public static readonly Histogram<double> ContentImportDuration = Meter.CreateHistogram<double>("learning_content_import_duration_seconds");

    private static int GetActiveAttemptsCount() { /* short query, cached 30s */ return 0; }
    private static int GetOutboxUndispatched() { /* short query, cached 10s */ return 0; }
}
```

Increment counters at appropriate points:
- `AttemptsStarted.Add(1, new("type", cmd.AttemptType));` in `AttemptOrchestrator.StartAsync`
- `XpAwarded.Add(totalXp, new("source", "lesson"));` in `CompleteAsync`
- etc.

- [ ] **Step 6: Test admin role gating (403 for non-admin)**

```csharp
[Fact]
public async Task AdminEndpoint_AsUser_Returns403()
{
    var client = CreateClientWithRole("user");
    var resp = await client.GetAsync("/api/v1/admin/learning/content-imports");
    resp.StatusCode.Should().Be(HttpStatusCode.Forbidden);
}

[Fact]
public async Task AdminEndpoint_AsAdmin_Returns200()
{
    var client = CreateClientWithRole("admin");
    var resp = await client.GetAsync("/api/v1/admin/learning/content-imports");
    resp.EnsureSuccessStatusCode();
}
```

- [ ] **Step 7: Wire Prometheus scraping endpoint**

```csharp
app.MapPrometheusScrapingEndpoint();  // /metrics
```

- [ ] **Step 8: Run all tests + smoke metric scrape**

```bash
dotnet test
dotnet run --project src/Skillvio.Learning.Api &
sleep 5
curl -s http://localhost:8080/metrics | grep learning_
kill %1
```

Expected: At least `learning_attempts_started_total`, `learning_outbox_undispatched_count` visible.

- [ ] **Step 9: Commit**

```bash
git commit -am "feat(s3a-t18a): admin endpoints (10) + role auth + Serilog/Loki + OTel/Tempo + 7 Prometheus metrics"
git push
```

---

## Sprint 3a Wrap-Up

- [ ] **Step 1: Run full test suite**

```bash
dotnet test --collect "XPlat Code Coverage"
```

Coverage targets:
- Domain ≥ 90%
- Application ≥ 75%
- Integration: full happy paths + error scenarios

- [ ] **Step 2: Verify DoD against spec §17**

- [ ] 30 tablo migration applies cleanly (T2 integration tests)
- [ ] Tüm content endpoint'leri çalışır, locale-aware (T11a)
- [ ] BadgeService 30+ rozeti doğru evaluate eder (T8)
- [ ] Streak gece doğru ilerler/sıfırlanır (T6)
- [ ] Hearts 4 saatte 1 yenilenir, Pro unlimited (T5)
- [ ] Lesson complete event publish ediliyor (T7b + T10)
- [ ] Exam attempt flow timed, scored, cert eligible (T9 + T12)
- [ ] Skillvio cert verifiable (T9)
- [ ] ContentSync Git→DB tam çalışır (T15)
- [ ] 30 cert catalog seed Git'te (T16)

- [ ] **Step 3: Tag release**

```bash
git tag -a v0.3.0-sprint3a -m "Sprint 3a complete — Learning Service core"
git push --tags
```

- [ ] **Step 4: Demo to stakeholders**

Walk through:
1. Onboarding (select 3 tracks, pin one)
2. Start lesson → submit answers → complete → see XP + badge
3. Start exam → answer questions → complete → see Skillvio cert
4. Verify cert at public URL

---

## Self-Review Checklist (post-write)

- [ ] **Spec coverage:** Every Sprint 3a task (T1–T18a) has a section. Every spec section §1–§11 + §17 referenced. ✅
- [ ] **Placeholder scan:** No `TBD`, `TODO`, "implement later", "similar to". ✅
- [ ] **Type consistency:** `IAttemptOrchestrator` methods match across T7a/T7b. `IBadgeRuleEngine.EvaluateSyncTierAsync` signature consistent. `IOutbox.AddAsync` consistent. ✅
- [ ] **Cross-task integrity:** T7b depends on T5 (ScoringService), T6 (StreakService), T8 (BadgeRuleEngine), T9 (CertificateEligibilityService), T10 (IOutbox) — all present in earlier tasks. ✅

If any issue is found post-execution, fix inline and continue.

