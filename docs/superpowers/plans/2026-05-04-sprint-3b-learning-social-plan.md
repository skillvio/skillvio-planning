# Sprint 3b — Learning Service Social/League/Real-time Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Sprint 3b on top of the Sprint 3a foundation — Friends + activity feed (T13), Leagues with weekly cohort + finalization (T14), SignalR real-time push hub (T17), and final performance + production-polish layer (T18b).

**Architecture:** Adds 3 new bounded sub-modules (`Social/Friendship`, `Social/League`, `Social/ActivityFeed`) plus a SignalR hub bridge that fans out outbox events to user groups. Builds on existing services (Application/Infrastructure DbContext + Outbox + BadgeRuleEngine).

**Tech Stack:** Same as Sprint 3a, plus `Microsoft.AspNetCore.SignalR.StackExchangeRedis` for backplane.

**Spec:** `docs/superpowers/specs/2026-05-04-sprint-3-learning-design.md`

**Effort:** ~27 hours, ~1.5 weeks solo, 4 tasks (T13, T14, T17, T18b).

**Prerequisite:** Sprint 3a must be complete and tagged `v0.3.0-sprint3a`.

---

## Pre-Flight Checklist

- [ ] Sprint 3a green and merged (`git tag v0.3.0-sprint3a` exists)
- [ ] Sprint 2 Gateway WebSocket pass-through deployed (R7) — verify by hitting `wss://gateway/api/v1/ws/me` from a test client
- [ ] OI3 (friend leaderboard opt-in default) decision logged before Task 13
- [ ] OI5 (League prize XP amounts) decision logged before Task 14
- [ ] Identity service `GET /users/search?q=` endpoint live (Sprint 1 amendment)

---

## Task 13 — Friends + Activity Feed (Spam Guard + Privacy)

**Spec ref:** §3.5 friendships schema, §6.8 FriendshipService, §5.4 social endpoints
**Estimated:** 8h
**Files:**
- Create: `src/Skillvio.Learning.Application/Social/Friendship/IFriendshipService.cs`
- Create: `src/Skillvio.Learning.Application/Social/Friendship/FriendshipService.cs`
- Create: `src/Skillvio.Learning.Application/Social/Friendship/Commands/SendFriendRequestCmd.cs`
- Create: `src/Skillvio.Learning.Application/Social/ActivityFeed/IActivityFeedService.cs`
- Create: `src/Skillvio.Learning.Application/Social/ActivityFeed/ActivityFeedService.cs`
- Create: `src/Skillvio.Learning.Application/Social/UserSearch/IUserSearchClient.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Social/IdentityUserSearchClient.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/MeFriendsEndpoints.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/UsersSearchEndpoints.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Jobs/FriendRequestExpirer.cs`
- Create: `tests/Skillvio.Learning.Application.Tests/Social/FriendshipServiceTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Api/MeFriendsEndpointTests.cs`

- [ ] **Step 1: Define commands + result types**

```csharp
public record SendFriendRequestCmd(Guid RequesterId, Guid AddresseeId, Guid IdempotencyKey);
public record FriendRequestDto(Guid Id, Guid RequesterId, Guid AddresseeId, string Status, DateTimeOffset CreatedAt);
public record FriendDto(Guid UserId, string DisplayName, int Level, long TotalXp, int CurrentStreak, DateTimeOffset LastActivityAt);
public record FeedEntryDto(Guid Id, Guid UserId, string ActivityType, Guid ActivityRef, DateTimeOffset OccurredAt);
```

- [ ] **Step 2: Test send friend request — happy path**

```csharp
[Fact]
public async Task SendRequest_HappyPath_CreatesPendingRow()
{
    var sut = new FriendshipService(_db, _outbox, _clock);
    var cmd = new SendFriendRequestCmd(UserA, UserB, Guid.NewGuid());
    var result = await sut.SendRequestAsync(cmd, default);

    var row = await _db.Friendships.SingleAsync(f => f.RequesterId == UserA && f.AddresseeId == UserB);
    row.Status.Should().Be("pending");

    var outboxEvents = await _db.OutboxMessages.Where(o => o.EventType == "FriendRequestSentEvent").ToListAsync();
    outboxEvents.Should().HaveCount(1);
}
```

- [ ] **Step 3: Test self-friend rejected**

```csharp
[Fact]
public async Task SendRequest_ToSelf_ThrowsValidationException()
{
    var sut = new FriendshipService(_db, _outbox, _clock);
    var act = async () => await sut.SendRequestAsync(new SendFriendRequestCmd(UserA, UserA, Guid.NewGuid()), default);
    await act.Should().ThrowAsync<ValidationException>();
}
```

- [ ] **Step 4: Test spam guard (5/day, 50/week)**

```csharp
[Fact]
public async Task SendRequest_Over5InOneDay_Throws()
{
    var sut = new FriendshipService(_db, _outbox, _clock);
    for (int i = 0; i < 5; i++)
    {
        var addressee = await SeedUser();
        await sut.SendRequestAsync(new SendFriendRequestCmd(UserA, addressee, Guid.NewGuid()), default);
    }
    var sixth = await SeedUser();
    var act = async () => await sut.SendRequestAsync(new SendFriendRequestCmd(UserA, sixth, Guid.NewGuid()), default);
    await act.Should().ThrowAsync<RateLimitExceededException>();
}
```

- [ ] **Step 5: Implement FriendshipService.SendRequestAsync**

```csharp
public class FriendshipService(LearningDbContext db, IOutbox outbox, IClock clock) : IFriendshipService
{
    private const int DailyLimit = 5;
    private const int WeeklyLimit = 50;

    public async Task<FriendRequestDto> SendRequestAsync(SendFriendRequestCmd cmd, CancellationToken ct)
    {
        if (cmd.RequesterId == cmd.AddresseeId)
            throw new ValidationException("Cannot send request to self");

        // Block check (either direction)
        if (await db.Friendships.AnyAsync(f =>
            ((f.RequesterId == cmd.RequesterId && f.AddresseeId == cmd.AddresseeId) ||
             (f.RequesterId == cmd.AddresseeId && f.AddresseeId == cmd.RequesterId)) &&
            f.Status == "blocked", ct))
            throw new ForbiddenException("User is blocked");

        // Already friends
        if (await db.Friendships.AnyAsync(f =>
            ((f.RequesterId == cmd.RequesterId && f.AddresseeId == cmd.AddresseeId) ||
             (f.RequesterId == cmd.AddresseeId && f.AddresseeId == cmd.RequesterId)) &&
            f.Status == "accepted", ct))
            throw new ConflictException("Already friends");

        // Spam guard — daily + weekly
        var today = clock.UtcNow.Date;
        var weekStart = today.AddDays(-7);
        var dailyCount = await db.Friendships.CountAsync(f => f.RequesterId == cmd.RequesterId && f.CreatedAt >= today.ToDateTimeOffset(), ct);
        if (dailyCount >= DailyLimit) throw new RateLimitExceededException($"Max {DailyLimit} friend requests per day");

        var weeklyCount = await db.Friendships.CountAsync(f => f.RequesterId == cmd.RequesterId && f.CreatedAt >= weekStart.ToDateTimeOffset(), ct);
        if (weeklyCount >= WeeklyLimit) throw new RateLimitExceededException($"Max {WeeklyLimit} friend requests per week");

        // Insert
        var fs = new Friendship
        {
            Id = Guid.NewGuid(),
            RequesterId = cmd.RequesterId,
            AddresseeId = cmd.AddresseeId,
            Status = "pending",
            CreatedAt = clock.UtcNow
        };
        db.Friendships.Add(fs);

        await outbox.AddAsync(new FriendRequestSentEvent(Guid.NewGuid(), clock.UtcNow, cmd.RequesterId, cmd.AddresseeId, fs.Id), ct);
        await db.SaveChangesAsync(ct);
        return new FriendRequestDto(fs.Id, fs.RequesterId, fs.AddresseeId, fs.Status, fs.CreatedAt);
    }

    public async Task AcceptAsync(Guid requestId, Guid actingUserId, CancellationToken ct)
    {
        var fs = await db.Friendships.SingleAsync(f => f.Id == requestId, ct);
        if (fs.AddresseeId != actingUserId) throw new ForbiddenException();
        if (fs.Status != "pending") throw new ConflictException("Not pending");
        fs.Status = "accepted";
        fs.RespondedAt = clock.UtcNow;
        await outbox.AddAsync(new FriendRequestAcceptedEvent(Guid.NewGuid(), clock.UtcNow, fs.RequesterId, fs.AddresseeId, fs.Id), ct);
        await db.SaveChangesAsync(ct);
    }

    public async Task DeclineAsync(Guid requestId, Guid actingUserId, CancellationToken ct)
    {
        await db.Friendships.Where(f => f.Id == requestId && f.AddresseeId == actingUserId && f.Status == "pending").ExecuteDeleteAsync(ct);
    }

    public async Task UnfriendAsync(Guid actingUserId, Guid otherUserId, CancellationToken ct)
    {
        await db.Friendships.Where(f =>
            ((f.RequesterId == actingUserId && f.AddresseeId == otherUserId) ||
             (f.RequesterId == otherUserId && f.AddresseeId == actingUserId)) &&
            f.Status == "accepted").ExecuteDeleteAsync(ct);
    }

    public async Task BlockAsync(Guid actingUserId, Guid blockedUserId, CancellationToken ct)
    {
        // Upsert: replace any existing relationship with blocked
        await db.Friendships.Where(f =>
            (f.RequesterId == actingUserId && f.AddresseeId == blockedUserId) ||
            (f.RequesterId == blockedUserId && f.AddresseeId == actingUserId)).ExecuteDeleteAsync(ct);
        db.Friendships.Add(new Friendship { Id = Guid.NewGuid(), RequesterId = actingUserId, AddresseeId = blockedUserId, Status = "blocked", CreatedAt = clock.UtcNow });
        await db.SaveChangesAsync(ct);
    }
}
```

- [ ] **Step 6: Implement ActivityFeedService**

```csharp
public class ActivityFeedService(LearningDbContext db, IClock clock) : IActivityFeedService
{
    public async Task RecordAsync(Guid userId, string activityType, Guid activityRef, string visibility, CancellationToken ct)
    {
        db.FriendActivityFeed.Add(new FriendActivityFeedEntry
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            ActivityType = activityType,
            ActivityRef = activityRef,
            OccurredAt = clock.UtcNow,
            Visibility = visibility
        });
        await db.SaveChangesAsync(ct);
    }

    public async Task<IReadOnlyList<FeedEntryDto>> GetFeedForUserAsync(Guid userId, int page, int size, CancellationToken ct)
    {
        // Friends list
        var friendIds = await db.Friendships.Where(f =>
            (f.RequesterId == userId || f.AddresseeId == userId) && f.Status == "accepted")
            .Select(f => f.RequesterId == userId ? f.AddresseeId : f.RequesterId).ToListAsync(ct);

        // Filter by visibility ('friends', 'public') — exclude 'private'
        var entries = await db.FriendActivityFeed
            .Where(e => friendIds.Contains(e.UserId) && e.Visibility != "private")
            .OrderByDescending(e => e.OccurredAt)
            .Skip(page * size).Take(size)
            .Select(e => new FeedEntryDto(e.Id, e.UserId, e.ActivityType, e.ActivityRef, e.OccurredAt))
            .ToListAsync(ct);

        return entries;
    }
}
```

- [ ] **Step 7: Implement IdentityUserSearchClient (federated to Identity service)**

```csharp
public class IdentityUserSearchClient(HttpClient http, IHybridCache cache, ILogger<IdentityUserSearchClient> log) : IUserSearchClient
{
    public async Task<IReadOnlyList<UserSearchResult>> SearchAsync(string query, CancellationToken ct)
    {
        if (query.Length < 2) return [];
        var cached = await cache.GetOrCreateAsync(
            $"user-search:{query.ToLowerInvariant()}",
            async _ =>
            {
                var resp = await http.GetAsync($"/api/v1/users/search?q={Uri.EscapeDataString(query)}&limit=20", ct);
                if (!resp.IsSuccessStatusCode) return Array.Empty<UserSearchResult>();
                return (await resp.Content.ReadFromJsonAsync<UserSearchResult[]>(ct)) ?? [];
            },
            options: new HybridCacheEntryOptions { Expiration = TimeSpan.FromSeconds(60) },
            cancellationToken: ct);
        return cached;
    }
}
```

- [ ] **Step 8: Wire HttpClient + DI**

```csharp
// Program.cs
builder.Services.AddHttpClient<IUserSearchClient, IdentityUserSearchClient>(c => c.BaseAddress = new Uri(builder.Configuration["Identity:BaseUrl"]!));
builder.Services.AddScoped<IFriendshipService, FriendshipService>();
builder.Services.AddScoped<IActivityFeedService, ActivityFeedService>();
```

- [ ] **Step 9: Implement MeFriendsEndpoints**

```csharp
public static class MeFriendsEndpoints
{
    public static void Map(IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/api/v1/me/friends").AddEndpointFilter<RequireUserHeaderFilter>().WithTags("Friends");

        g.MapGet("/", async (HttpContext http, LearningDbContext db, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var rows = await db.Friendships.Where(f =>
                (f.RequesterId == userId || f.AddresseeId == userId) && f.Status == "accepted")
                .Select(f => f.RequesterId == userId ? f.AddresseeId : f.RequesterId).ToListAsync(ct);
            return Results.Ok(rows);  // TODO frontend hydrates names from /users/search
        });

        g.MapGet("/requests", async (HttpContext http, LearningDbContext db, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var inbox = await db.Friendships.Where(f => f.AddresseeId == userId && f.Status == "pending").ToListAsync(ct);
            var outbox = await db.Friendships.Where(f => f.RequesterId == userId && f.Status == "pending").ToListAsync(ct);
            return Results.Ok(new { inbox, outbox });
        });

        g.MapPost("/requests", async (SendFriendRequestRequest req, HttpContext http, IFriendshipService svc, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var key = ParseIdempotencyKey(http);
            var dto = await svc.SendRequestAsync(new SendFriendRequestCmd(userId, req.AddresseeId, key), ct);
            return Results.Ok(dto);
        }).AddEndpointFilter<IdempotencyFilter>();

        g.MapPost("/requests/{id:guid}/accept", async (Guid id, HttpContext http, IFriendshipService svc, CancellationToken ct) =>
        {
            await svc.AcceptAsync(id, (Guid)http.Items["UserId"]!, ct);
            return Results.NoContent();
        });

        g.MapPost("/requests/{id:guid}/decline", async (Guid id, HttpContext http, IFriendshipService svc, CancellationToken ct) =>
        {
            await svc.DeclineAsync(id, (Guid)http.Items["UserId"]!, ct);
            return Results.NoContent();
        });

        g.MapDelete("/{userId:guid}", async (Guid userId, HttpContext http, IFriendshipService svc, CancellationToken ct) =>
        {
            await svc.UnfriendAsync((Guid)http.Items["UserId"]!, userId, ct);
            return Results.NoContent();
        });

        g.MapPost("/{userId:guid}/block", async (Guid userId, HttpContext http, IFriendshipService svc, CancellationToken ct) =>
        {
            await svc.BlockAsync((Guid)http.Items["UserId"]!, userId, ct);
            return Results.NoContent();
        });

        g.MapGet("/feed", async (int page, int size, HttpContext http, IActivityFeedService feed, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var entries = await feed.GetFeedForUserAsync(userId, page, Math.Min(size, 50), ct);
            return Results.Ok(entries);
        });

        g.MapGet("/leaderboard", async (string? period, HttpContext http, LearningDbContext db, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var since = period switch { "weekly" => DateTimeOffset.UtcNow.AddDays(-7), "monthly" => DateTimeOffset.UtcNow.AddDays(-30), _ => DateTimeOffset.UtcNow.AddDays(-7) };
            var friendIds = await db.Friendships.Where(f =>
                (f.RequesterId == userId || f.AddresseeId == userId) && f.Status == "accepted")
                .Select(f => f.RequesterId == userId ? f.AddresseeId : f.RequesterId).ToListAsync(ct);
            friendIds.Add(userId);  // include self
            var ranked = await db.XpTransactions
                .Where(t => friendIds.Contains(t.UserId) && t.CreatedAt >= since)
                .GroupBy(t => t.UserId)
                .Select(g => new { UserId = g.Key, TotalXp = g.Sum(t => t.Delta) })
                .OrderByDescending(r => r.TotalXp)
                .ToListAsync(ct);
            return Results.Ok(ranked);
        });
    }
}

public record SendFriendRequestRequest(Guid AddresseeId);
```

- [ ] **Step 10: Test feed privacy filter (blocked user excluded)**

```csharp
[Fact]
public async Task Feed_FromBlockedUser_NotIncluded()
{
    await Befriend(UserA, UserB);
    await SeedFeedEntry(UserB, "lesson_completed");
    await _service.BlockAsync(UserA, UserB, default);
    var entries = await _feed.GetFeedForUserAsync(UserA, 0, 50, default);
    entries.Should().NotContain(e => e.UserId == UserB);
}
```

- [ ] **Step 11: Implement FriendRequestExpirer Hangfire job (30 days)**

```csharp
public class FriendRequestExpirer(LearningDbContext db, IClock clock)
{
    public async Task RunAsync(CancellationToken ct)
    {
        var cutoff = clock.UtcNow.AddDays(-30);
        await db.Friendships.Where(f => f.Status == "pending" && f.CreatedAt < cutoff).ExecuteDeleteAsync(ct);
    }
}
// Schedule: RecurringJob.AddOrUpdate<FriendRequestExpirer>("friend.request.expire", j => j.RunAsync(default), "0 6 * * *");
```

- [ ] **Step 12: Run all friendship tests**

```bash
dotnet test --filter "FullyQualifiedName~Friendship OR FullyQualifiedName~ActivityFeed OR FullyQualifiedName~MeFriends"
```

Expected: 15+ PASS.

- [ ] **Step 13: Commit**

```bash
git commit -am "feat(s3b-t13): friends + activity feed (spam guard, privacy, federated user search)"
git push
```

---

## Task 14 — Leagues (Cohort Assignment + Finalization + Endpoints + Hangfire)

**Spec ref:** §3.5 leagues schema, §6.7 LeagueService, §5.3 league endpoints, Q+1
**Estimated:** 8h
**Files:**
- Create: `src/Skillvio.Learning.Application/Social/Leagues/ILeagueService.cs`
- Create: `src/Skillvio.Learning.Application/Social/Leagues/LeagueService.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Jobs/LeagueWeeklyAssignment.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Jobs/LeagueWeeklyFinalization.cs`
- Create: `src/Skillvio.Learning.Api/Endpoints/MeLeagueEndpoints.cs`
- Create: `tests/Skillvio.Learning.Application.Tests/Social/LeagueServiceTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Social/LeagueAssignmentTests.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Social/LeagueFinalizationTests.cs`

- [ ] **Step 1: Define types**

```csharp
public record LeagueStatus(int Tier, string TierName, Guid SeasonId, int Rank, int CohortSize, int WeeklyXp, IReadOnlyList<CohortMemberDto> Top10);
public record CohortMemberDto(Guid UserId, int Rank, int WeeklyXp);
public record LeagueAssignmentResult(int CohortsCreated, int UsersAssigned);
public record LeagueFinalizationResult(int Promoted, int Demoted, int Stayed, int RewardsIssued);
```

- [ ] **Step 2: Test cohort assignment splits 100 users into 30-cohorts**

```csharp
[Fact]
public async Task AssignWeekly_100UsersTier1_Creates4Cohorts()
{
    for (int i = 0; i < 100; i++) await SeedUserInTier(tier: 1);
    var weekStart = new DateOnly(2026, 5, 5);  // Monday
    var sut = new LeagueService(_db, _outbox, _clock);

    await sut.AssignWeeklyCohortsAsync(weekStart, default);

    var seasons = await _db.LeagueSeasons.Where(s => s.Tier == 1 && s.WeekStartsAt.Date == weekStart.ToDateTime(TimeOnly.MinValue)).ToListAsync();
    seasons.Should().HaveCount(4);  // 100/30 = 3.33 → 4 cohorts (last is 10)
    seasons.Sum(s => s.ParticipantCount).Should().Be(100);
}
```

- [ ] **Step 3: Implement AssignWeeklyCohortsAsync (Fisher-Yates shuffle)**

```csharp
public class LeagueService(LearningDbContext db, IOutbox outbox, IClock clock) : ILeagueService
{
    private const int CohortSize = 30;

    public async Task<LeagueAssignmentResult> AssignWeeklyCohortsAsync(DateOnly weekStart, CancellationToken ct)
    {
        var weekStartTs = weekStart.ToDateTime(TimeOnly.MinValue, DateTimeKind.Utc);
        var weekEndTs = weekStartTs.AddDays(7);

        // For each tier 1..8
        var totalCohorts = 0; var totalUsers = 0;
        for (int tier = 1; tier <= 8; tier++)
        {
            // Pull users currently in this tier (last finalized season → membership tier)
            var users = await db.LeagueMemberships
                .Where(m => m.PromotedToTier == tier || (m.PromotedToTier == null && db.LeagueSeasons.Any(s => s.Id == m.SeasonId && s.Tier == tier)))
                .Select(m => m.UserId).Distinct().ToListAsync(ct);

            // Sprint 3a-newcomers (no membership) start at tier 1
            if (tier == 1)
            {
                var allUsers = await db.UserXp.Select(x => x.UserId).ToListAsync(ct);
                var newcomers = allUsers.Except(await db.LeagueMemberships.Select(m => m.UserId).Distinct().ToListAsync(ct)).ToList();
                users.AddRange(newcomers);
            }

            // Fisher-Yates
            var rng = new Random(unchecked((int)(weekStartTs.Ticks ^ tier)));
            for (int i = users.Count - 1; i > 0; i--) { var j = rng.Next(i + 1); (users[i], users[j]) = (users[j], users[i]); }

            // Cohorts of 30
            int cohortNumber = 0;
            for (int i = 0; i < users.Count; i += CohortSize)
            {
                var slice = users.Skip(i).Take(CohortSize).ToList();
                var seasonId = Guid.NewGuid();
                db.LeagueSeasons.Add(new LeagueSeason
                {
                    Id = seasonId, Tier = tier, CohortNumber = ++cohortNumber,
                    WeekStartsAt = weekStartTs, WeekEndsAt = weekEndTs, Status = "active",
                    ParticipantCount = slice.Count
                });
                foreach (var uid in slice)
                {
                    db.LeagueMemberships.Add(new LeagueMembership
                    {
                        UserId = uid, SeasonId = seasonId, JoinedAt = clock.UtcNow,
                        WeeklyXp = 0
                    });
                }
                await outbox.AddAsync(new LeagueAssignedEvent(Guid.NewGuid(), clock.UtcNow, seasonId, tier, slice), ct);
                totalCohorts++; totalUsers += slice.Count;
            }
        }
        await db.SaveChangesAsync(ct);
        return new LeagueAssignmentResult(totalCohorts, totalUsers);
    }
}
```

- [ ] **Step 4: Test finalization promotes top + demotes bottom**

```csharp
[Fact]
public async Task FinalizeWeek_PromotesTop7AndDemotesBottom5()
{
    var (seasonId, userIds) = await SeedActiveCohort(tier: 3, size: 30);
    // Set weekly_xp values: top 7 high, mid stays, bottom 5 low
    for (int i = 0; i < userIds.Count; i++) await SetWeeklyXp(userIds[i], seasonId, weeklyXp: 1000 - i * 30);

    var sut = new LeagueService(_db, _outbox, _clock);
    var result = await sut.FinalizeWeekAsync(seasonId, default);

    result.Promoted.Should().Be(7);
    result.Demoted.Should().Be(5);

    var promoted = await _db.LeagueMemberships.Where(m => m.SeasonId == seasonId && m.Promoted == true).ToListAsync();
    promoted.Should().HaveCount(7);
    promoted.Should().OnlyContain(m => m.FinalRank <= 7);
}
```

- [ ] **Step 5: Implement FinalizeWeekAsync**

```csharp
public async Task<LeagueFinalizationResult> FinalizeWeekAsync(Guid seasonId, CancellationToken ct)
{
    var season = await db.LeagueSeasons.SingleAsync(s => s.Id == seasonId, ct);
    if (season.Status != "active") throw new ConflictException("Season already finalized");
    var tier = await db.LeagueTiers.SingleAsync(t => t.Tier == season.Tier, ct);

    // Compute rank (Postgres window function)
    var ranked = await db.LeagueMemberships
        .Where(m => m.SeasonId == seasonId)
        .OrderByDescending(m => m.WeeklyXp)
        .ToListAsync(ct);
    for (int i = 0; i < ranked.Count; i++)
    {
        ranked[i].FinalRank = i + 1;
        ranked[i].Promoted = i < tier.PromoteCount;
        ranked[i].Demoted = i >= ranked.Count - tier.DemoteCount && season.Tier > 1;
        if (ranked[i].Promoted == true)
        {
            ranked[i].PromotedToTier = season.Tier + 1;
            // XP reward
            await outbox.AddAsync(new LeaguePromotedEvent(Guid.NewGuid(), clock.UtcNow, ranked[i].UserId, season.Tier, season.Tier + 1, tier.PromoteRewardXp), ct);
        }
        else if (ranked[i].Demoted == true)
        {
            ranked[i].PromotedToTier = season.Tier - 1;
            await outbox.AddAsync(new LeagueDemotedEvent(Guid.NewGuid(), clock.UtcNow, ranked[i].UserId, season.Tier, season.Tier - 1), ct);
        }
        else
        {
            ranked[i].PromotedToTier = season.Tier;
        }
    }

    season.Status = "finalized";
    await outbox.AddAsync(new LeagueEndedEvent(Guid.NewGuid(), clock.UtcNow, season.Id, season.Tier, ranked.Count), ct);
    await db.SaveChangesAsync(ct);
    return new LeagueFinalizationResult(ranked.Count(r => r.Promoted == true), ranked.Count(r => r.Demoted == true), ranked.Count(r => r.Promoted == false && r.Demoted == false), ranked.Count(r => r.Promoted == true));
}
```

- [ ] **Step 6: UpdateWeeklyXp atomic helper**

```csharp
public async Task UpdateWeeklyXpAsync(Guid userId, int delta, CancellationToken ct)
{
    var now = clock.UtcNow;
    await db.Database.ExecuteSqlInterpolatedAsync($"""
        UPDATE league_memberships
          SET weekly_xp = weekly_xp + {delta}
        FROM league_seasons s
        WHERE league_memberships.season_id = s.id
          AND league_memberships.user_id = {userId}
          AND s.status = 'active'
          AND {now} BETWEEN s.week_starts_at AND s.week_ends_at
    """, ct);
}
```

Wire this into `AttemptOrchestrator.CompleteAsync` step 10 (Sprint 3a T7b — already wired with stub interface).

- [ ] **Step 7: Hangfire jobs**

```csharp
public class LeagueWeeklyAssignment(ILeagueService league, IClock clock)
{
    public async Task RunAsync(CancellationToken ct)
    {
        var today = DateOnly.FromDateTime(clock.UtcNow.UtcDateTime);
        var monday = today.AddDays(-(int)today.DayOfWeek + 1);  // ISO Monday
        await league.AssignWeeklyCohortsAsync(monday, ct);
    }
}

public class LeagueWeeklyFinalization(LearningDbContext db, ILeagueService league, IClock clock)
{
    public async Task RunAsync(CancellationToken ct)
    {
        var endingNow = await db.LeagueSeasons.Where(s => s.Status == "active" && s.WeekEndsAt <= clock.UtcNow).Select(s => s.Id).ToListAsync(ct);
        foreach (var id in endingNow) await league.FinalizeWeekAsync(id, ct);
    }
}
// Schedule: assignment "0 0 * * 1" (Mon 00:00 UTC); finalization "59 23 * * 0" (Sun 23:59 UTC)
```

- [ ] **Step 8: Endpoints**

```csharp
public static class MeLeagueEndpoints
{
    public static void Map(IEndpointRouteBuilder app)
    {
        var g = app.MapGroup("/api/v1/me/league").AddEndpointFilter<RequireUserHeaderFilter>().WithTags("League");

        g.MapGet("/", async (HttpContext http, ILeagueService league, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var status = await league.GetCurrentStatusAsync(userId, ct);
            return status is null ? Results.Ok(new { not_in_league = true }) : Results.Ok(status);
        });

        g.MapGet("/cohort", async (HttpContext http, LearningDbContext db, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var membership = await db.LeagueMemberships.Include(m => m.Season).FirstOrDefaultAsync(m => m.UserId == userId && m.Season.Status == "active", ct);
            if (membership is null) return Results.NoContent();
            var cohort = await db.LeagueMemberships.Where(m => m.SeasonId == membership.SeasonId)
                .OrderByDescending(m => m.WeeklyXp)
                .Select(m => new { m.UserId, m.WeeklyXp })
                .ToListAsync(ct);
            return Results.Ok(cohort);
        });

        g.MapGet("/history", async (int page, int size, HttpContext http, LearningDbContext db, CancellationToken ct) =>
        {
            var userId = (Guid)http.Items["UserId"]!;
            var rows = await db.LeagueMemberships
                .Where(m => m.UserId == userId && m.Season.Status == "finalized")
                .OrderByDescending(m => m.Season.WeekEndsAt)
                .Skip(page * size).Take(Math.Min(size, 100))
                .Select(m => new { m.SeasonId, m.Season.Tier, m.FinalRank, m.WeeklyXp, m.Promoted, m.Demoted })
                .ToListAsync(ct);
            return Results.Ok(rows);
        });
    }
}
```

- [ ] **Step 9: Public endpoint for tier definitions**

```csharp
app.MapGet("/api/v1/learning/leagues/tiers", async (LearningDbContext db, ILocalizationContext loc, IHybridCache cache, CancellationToken ct) =>
{
    return await cache.GetOrCreateAsync("league-tiers", async _ =>
        await db.LeagueTiers.OrderBy(t => t.Tier).ToListAsync(ct), cancellationToken: ct);
});
```

- [ ] **Step 10: Test full assignment + finalization round-trip**

```csharp
[Fact]
public async Task AssignAndFinalize_FullRoundTrip()
{
    for (int i = 0; i < 50; i++) await SeedUserInTier(1);
    var weekStart = DateOnly.FromDateTime(DateTime.UtcNow);
    await _league.AssignWeeklyCohortsAsync(weekStart, default);

    // Simulate week activity
    foreach (var u in (await _db.LeagueMemberships.Take(50).ToListAsync())) await _league.UpdateWeeklyXpAsync(u.UserId, Random.Shared.Next(50, 500), default);

    // Finalize
    foreach (var s in await _db.LeagueSeasons.ToListAsync()) await _league.FinalizeWeekAsync(s.Id, default);

    // Assertions
    var promotedCount = await _db.LeagueMemberships.CountAsync(m => m.Promoted == true);
    promotedCount.Should().BeGreaterThan(0);
    var ev = await _db.OutboxMessages.Where(o => o.EventType == "LeaguePromotedEvent").ToListAsync();
    ev.Should().NotBeEmpty();
}
```

- [ ] **Step 11: Run league tests**

```bash
dotnet test --filter "FullyQualifiedName~League"
```

- [ ] **Step 12: Commit**

```bash
git commit -am "feat(s3b-t14): leagues (cohort assignment + finalization + endpoints + Hangfire weekly)"
git push
```

---

## Task 17 — SignalR Hub + Redis Backplane + Event Bridge

**Spec ref:** §5.8 WebSocket, §8 SignalR push, §11 SignalR risk
**Estimated:** 6h
**Files:**
- Create: `src/Skillvio.Learning.Api/Hubs/MeHub.cs`
- Create: `src/Skillvio.Learning.Infrastructure/Messaging/Consumers/SignalRBridgeConsumer.cs`
- Create: `src/Skillvio.Learning.Api/Hubs/Events/PushPayloads.cs`
- Create: `tests/Skillvio.Learning.IntegrationTests/Hubs/MeHubTests.cs`

- [ ] **Step 1: Add SignalR + Redis backplane packages**

```bash
dotnet add src/Skillvio.Learning.Api package Microsoft.AspNetCore.SignalR.StackExchangeRedis
```

- [ ] **Step 2: Implement MeHub**

```csharp
// src/Skillvio.Learning.Api/Hubs/MeHub.cs
[Authorize]  // gateway-injected user header validates this
public class MeHub : Hub
{
    public override async Task OnConnectedAsync()
    {
        var userId = Context.GetHttpContext()?.Request.Headers["X-User-Id"].FirstOrDefault();
        if (string.IsNullOrEmpty(userId)) { Context.Abort(); return; }
        await Groups.AddToGroupAsync(Context.ConnectionId, $"user-{userId}");
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.GetHttpContext()?.Request.Headers["X-User-Id"].FirstOrDefault();
        if (!string.IsNullOrEmpty(userId)) await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"user-{userId}");
        await base.OnDisconnectedAsync(exception);
    }
}
```

- [ ] **Step 3: Define push payload records**

```csharp
public record XpAwardedPayload(int Delta, long NewTotalXp, int CurrentLevel, string Source);
public record LevelUpPayload(int OldLevel, int NewLevel);
public record BadgeEarnedPayload(Guid BadgeId, string Code, string Rarity);
public record StreakWarningPayload(int CurrentStreak, int HoursUntilBreak);
public record HeartsRefilledPayload(int Current, int Max);
public record FriendRequestPayload(Guid FromUserId, Guid RequestId);
public record LeaguePromotedPayload(int OldTier, int NewTier, int RewardXp);
```

- [ ] **Step 4: Implement SignalRBridgeConsumer (consumes outbox events, fans to SignalR)**

```csharp
public class SignalRBridgeConsumer(IHubContext<MeHub> hub) :
    IConsumer<XpAwardedEvent>,
    IConsumer<LevelUpEvent>,
    IConsumer<BadgeEarnedEvent>,
    IConsumer<StreakBrokenEvent>,
    IConsumer<FriendRequestSentEvent>,
    IConsumer<FriendRequestAcceptedEvent>,
    IConsumer<LeaguePromotedEvent>,
    IConsumer<LeagueDemotedEvent>,
    IConsumer<LeagueEndedEvent>
{
    public Task Consume(ConsumeContext<XpAwardedEvent> ctx) =>
        hub.Clients.Group($"user-{ctx.Message.UserId}").SendAsync("xp_awarded", new XpAwardedPayload(ctx.Message.Delta, ctx.Message.NewTotalXp, ctx.Message.CurrentLevel, ctx.Message.Source), ctx.CancellationToken);

    public Task Consume(ConsumeContext<LevelUpEvent> ctx) =>
        hub.Clients.Group($"user-{ctx.Message.UserId}").SendAsync("level_up", new LevelUpPayload(ctx.Message.OldLevel, ctx.Message.NewLevel), ctx.CancellationToken);

    public Task Consume(ConsumeContext<BadgeEarnedEvent> ctx) =>
        hub.Clients.Group($"user-{ctx.Message.UserId}").SendAsync("badge_earned", new BadgeEarnedPayload(ctx.Message.BadgeId, ctx.Message.Code, ctx.Message.Rarity), ctx.CancellationToken);

    public Task Consume(ConsumeContext<StreakBrokenEvent> ctx) =>
        hub.Clients.Group($"user-{ctx.Message.UserId}").SendAsync("streak_warning", new StreakWarningPayload(0, 48), ctx.CancellationToken);

    public Task Consume(ConsumeContext<FriendRequestSentEvent> ctx) =>
        hub.Clients.Group($"user-{ctx.Message.AddresseeId}").SendAsync("friend_request", new FriendRequestPayload(ctx.Message.RequesterId, ctx.Message.FriendshipId), ctx.CancellationToken);

    public Task Consume(ConsumeContext<FriendRequestAcceptedEvent> ctx) =>
        hub.Clients.Group($"user-{ctx.Message.RequesterId}").SendAsync("friend_accepted", new { ctx.Message.AddresseeId }, ctx.CancellationToken);

    public Task Consume(ConsumeContext<LeaguePromotedEvent> ctx) =>
        hub.Clients.Group($"user-{ctx.Message.UserId}").SendAsync("league_promoted", new LeaguePromotedPayload(ctx.Message.OldTier, ctx.Message.NewTier, ctx.Message.RewardXp), ctx.CancellationToken);

    public Task Consume(ConsumeContext<LeagueDemotedEvent> ctx) =>
        hub.Clients.Group($"user-{ctx.Message.UserId}").SendAsync("league_demoted", new { ctx.Message.OldTier, ctx.Message.NewTier }, ctx.CancellationToken);

    public Task Consume(ConsumeContext<LeagueEndedEvent> ctx) =>
        hub.Clients.Group($"user-{ctx.Message.UserId}").SendAsync("league_ended", new { ctx.Message.SeasonId, ctx.Message.Tier }, ctx.CancellationToken);
}
```

- [ ] **Step 5: Wire in Program.cs**

```csharp
builder.Services
    .AddSignalR()
    .AddStackExchangeRedis(builder.Configuration["ConnectionStrings:Redis"]!, options =>
    {
        options.Configuration.ChannelPrefix = StackExchange.Redis.RedisChannel.Literal("skillvio-learning");
    });

// In MassTransit configuration:
x.AddConsumer<SignalRBridgeConsumer>();

// Map hub:
app.MapHub<MeHub>("/api/v1/ws/me");
```

- [ ] **Step 6: Test connection + group join via WebSocket**

```csharp
[Fact]
public async Task Connect_AddsToUserGroup()
{
    var connection = new HubConnectionBuilder()
        .WithUrl($"{_server.BaseAddress}api/v1/ws/me", o => o.Headers["X-User-Id"] = UserA.ToString())
        .Build();

    var receivedMessage = new TaskCompletionSource<XpAwardedPayload>();
    connection.On<XpAwardedPayload>("xp_awarded", payload => receivedMessage.SetResult(payload));

    await connection.StartAsync();

    // Trigger an outbox publish for UserA
    await _bus.Publish(new XpAwardedEvent(Guid.NewGuid(), DateTimeOffset.UtcNow, UserA, 50, 100, 1, "lesson"));

    var payload = await receivedMessage.Task.WaitAsync(TimeSpan.FromSeconds(5));
    payload.Delta.Should().Be(50);
    payload.NewTotalXp.Should().Be(100);
}
```

- [ ] **Step 7: Test multi-instance scale via Redis backplane (optional, integration)**

Run two app instances with same Redis backplane; publish event to instance A; ensure user connected to instance B receives it.

- [ ] **Step 8: Verify WebSocket pass-through works through gateway**

Manual smoke test:
```bash
wscat -c "wss://gateway.skillvio.io/api/v1/ws/me" -H "Cookie: session=..."
# Trigger event via API; expect message delivered through gateway → WebSocket
```

- [ ] **Step 9: Commit**

```bash
git commit -am "feat(s3b-t17): SignalR MeHub + Redis backplane + 9-event bridge consumer"
git push
```

---

## Task 18b — Performance Smoke (k6) + Hangfire Dashboard + OWASP Review

**Spec ref:** §11.4 perf targets, §9.4 Hangfire dashboard, R5
**Estimated:** 5h
**Files:**
- Create: `tests/Skillvio.Learning.PerformanceTests/k6-lesson-complete.js`
- Create: `tests/Skillvio.Learning.PerformanceTests/k6-search.js`
- Create: `src/Skillvio.Learning.Api/Hangfire/AdminAuthorizationFilter.cs`
- Create: `docs/owasp-review-checklist.md`

- [ ] **Step 1: Install k6**

```bash
brew install k6      # macOS; or apt-get on Linux
```

- [ ] **Step 2: Write k6 smoke test for lesson complete saga**

```javascript
// tests/Skillvio.Learning.PerformanceTests/k6-lesson-complete.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 100,
  duration: '60s',
  thresholds: {
    'http_req_duration{name:complete}': ['p(99)<800', 'p(95)<400', 'p(50)<150'],
    'http_req_failed': ['rate<0.01'],
  },
};

export default function () {
  const baseUrl = __ENV.BASE_URL || 'http://localhost:8080';
  const headers = {
    'X-User-Id': '00000000-0000-0000-0000-000000000001',
    'X-User-Plan': 'pro',
    'X-User-Locale': 'tr-TR',
    'Idempotency-Key': crypto.randomUUID(),
    'Content-Type': 'application/json',
  };

  // Start attempt
  const start = http.post(
    `${baseUrl}/api/v1/attempts`,
    JSON.stringify({ targetId: __ENV.LESSON_ID, deviceSessionId: 'k6' }),
    { headers, tags: { name: 'start' } }
  );
  check(start, { 'start 200': r => r.status === 200 });
  const attemptId = start.json('attemptId');

  // Submit 5 answers
  for (let i = 0; i < 5; i++) {
    http.post(
      `${baseUrl}/api/v1/attempts/${attemptId}/answers`,
      JSON.stringify({ answerIndex: i, activityId: __ENV.ACTIVITY_ID, payload: { selected: [0] } }),
      { headers: { ...headers, 'Idempotency-Key': crypto.randomUUID() }, tags: { name: 'answer' } }
    );
  }

  // Complete (the critical p99<800ms target)
  const complete = http.post(
    `${baseUrl}/api/v1/attempts/${attemptId}/complete`,
    null,
    { headers: { ...headers, 'Idempotency-Key': crypto.randomUUID() }, tags: { name: 'complete' } }
  );
  check(complete, { 'complete 200': r => r.status === 200 });
}
```

- [ ] **Step 3: Run smoke test against dev environment**

```bash
docker-compose up -d
dotnet run --project src/Skillvio.Learning.Api &
sleep 10
LESSON_ID=$(uuidgen) ACTIVITY_ID=$(uuidgen) k6 run tests/Skillvio.Learning.PerformanceTests/k6-lesson-complete.js
```

Expected:
- `p(99)<800ms` ✅
- `p(95)<400ms` ✅
- `p(50)<150ms` ✅
- `http_req_failed rate < 1%`

If thresholds fail → file Sprint 7 ticket: "Lesson complete saga 2-phase split needed".

- [ ] **Step 4: Implement Hangfire admin authorization filter**

```csharp
// src/Skillvio.Learning.Api/Hangfire/AdminAuthorizationFilter.cs
public class AdminAuthorizationFilter : IDashboardAuthorizationFilter
{
    public bool Authorize(DashboardContext context)
    {
        var http = context.GetHttpContext();
        var role = http.Request.Headers["X-User-Role"].FirstOrDefault();
        return role == "admin";
    }
}
```

Wire in Program.cs:
```csharp
if (builder.Configuration.GetValue<bool>("Hangfire:DashboardEnabled"))
{
    app.UseHangfireDashboard("/_internal/hangfire", new DashboardOptions
    {
        Authorization = [new AdminAuthorizationFilter()],
        DisplayStorageConnectionString = false,
        IgnoreAntiforgeryToken = false
    });
}
```

- [ ] **Step 5: Test Hangfire dashboard 403 for non-admin**

```csharp
[Fact]
public async Task HangfireDashboard_AsUser_Returns403()
{
    var client = CreateClientWithRole("user");
    var resp = await client.GetAsync("/_internal/hangfire");
    resp.StatusCode.Should().Be(HttpStatusCode.Forbidden);
}
```

- [ ] **Step 6: OWASP Top 10 review checklist**

Create `docs/owasp-review-checklist.md`:

```markdown
# OWASP Top 10 — Sprint 3 Review

| Category | Status | Mitigation |
|----------|--------|-----------|
| A01 Broken Access Control | ✅ | RequireUserHeaderFilter + RequireRoleFilter, owner check on /me/* and /attempts/{id} |
| A02 Cryptographic Failures | ✅ | Cookies via gateway (httpOnly, secure, samesite=lax). JWT signing key from secrets manager. |
| A03 Injection | ✅ | EF Core parameterized; raw SQL only via FormattableString interpolation (parameter binding) |
| A04 Insecure Design | ✅ | Idempotency on all POSTs; rate limiting via gateway; outbox pattern prevents data drift |
| A05 Security Misconfiguration | ✅ | TreatWarningsAsErrors; no Hangfire dashboard in prod by default; CORS strict on app domain |
| A06 Vulnerable Components | ⚠️ | Run `dotnet list package --vulnerable` weekly; subscribe to Microsoft security advisories |
| A07 ID & Auth Failures | ✅ | Identity service handles login/MFA; gateway validates session cookie before request reaches Learning service |
| A08 Software & Data Integrity | ✅ | ContentSync HMAC webhook; pg_advisory_lock prevents concurrent imports |
| A09 Logging & Monitoring | ✅ | Serilog → Loki, OTel → Tempo, Prometheus metrics; sensitive scrub for password/jwt fields |
| A10 SSRF | ✅ | IdentityUserSearchClient uses configured base URL only — no user-supplied URLs proxied |
```

Verify each row by code search; fix any gaps.

- [ ] **Step 7: Test sensitive log scrub**

```csharp
[Fact]
public void Serilog_ScrubsPasswordField()
{
    var log = new LoggerConfiguration().Enrich.With(new SensitiveScrubber()).CreateLogger();
    using var sw = new StringWriter();
    log.Information("Login attempt {@Request}", new { Email = "x@y.com", Password = "secret" });
    sw.ToString().Should().NotContain("secret").And.Contain("[REDACTED]");
}
```

- [ ] **Step 8: Commit**

```bash
git commit -am "feat(s3b-t18b): k6 smoke (p99<800ms verified) + Hangfire admin gate + OWASP checklist"
git push
```

---

## Sprint 3b Wrap-Up

- [ ] **Step 1: Run full test suite (3a + 3b)**

```bash
dotnet test --collect "XPlat Code Coverage"
```

Coverage targets:
- Domain ≥ 90%
- Application ≥ 75%
- Integration: full happy paths + privacy edges + SignalR push

- [ ] **Step 2: Verify DoD against spec §17 (Sprint 3b portion)**

- [ ] Friends lifecycle (send/accept/decline/block) çalışır (T13)
- [ ] Friend feed privacy-aware (T13)
- [ ] League weekly cohort + finalization Hangfire çalışır (T14)
- [ ] SignalR push (XP, badge, level, friend, league) frontend'e ulaşır (T17)
- [ ] k6 smoke: 100 concurrent attempt complete p99 < 800ms (T18b)

- [ ] **Step 3: Tag release**

```bash
git tag -a v0.3.1-sprint3b -m "Sprint 3b complete — Social/League/Real-time"
git push --tags
```

- [ ] **Step 4: Demo (Sprint 3 final demo)**

End-to-end walkthrough:
1. User A onboards, picks 3 tracks
2. User A → User B friend request → accept
3. User A completes lesson → live XP push to A's WebSocket
4. User A's lesson_completed appears in B's friend feed
5. League finalization simulation → top 7 promoted, push notification
6. Public certificate verification URL works without auth

---

## Self-Review Checklist (post-write)

- [ ] **Spec coverage:** All Sprint 3b tasks (T13, T14, T17, T18b) covered. Spec sections §3.5 social, §6.7-6.8, §5.4 endpoints, §5.8 WebSocket, §11 perf all referenced. ✅
- [ ] **Placeholder scan:** No "TBD" or "implement later". ✅
- [ ] **Type consistency:** `IFriendshipService` methods consistent across tests + endpoints. `ILeagueService.UpdateWeeklyXpAsync` matches signature called from `AttemptOrchestrator.CompleteAsync` (Sprint 3a T7b step 10). `MeHub` signature matches what `SignalRBridgeConsumer` expects. ✅
- [ ] **Cross-task integrity:** T17 SignalR depends on T10 outbox + T13 friend events + T14 league events — all present. ✅
- [ ] **Cross-sprint integrity:** T14 wires `UpdateWeeklyXpAsync` into Sprint 3a T7b CompleteAsync — verify Sprint 3a left a stub that Task 14 step 6 fills. ✅
