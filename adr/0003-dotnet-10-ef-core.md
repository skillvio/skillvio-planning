# ADR 0003 — .NET 10 + EF Core 10

- **Status:** Accepted (revised 2026-05-04)
- **Supersedes:** Initial draft (.NET 8) — .NET 10 LTS yayında ve daha doğru tercih
- **Date:** 2026-05-04

## Karar

Backend **.NET 10 (LTS) + EF Core 10 + Postgres 16**.

## Versiyon Takvimi (2024-2028)

| Version | Yayın | Tip | Destek Sonu |
|---------|-------|-----|-------------|
| .NET 8 | 2023-11 | LTS | 2026-11 (6 ay kaldı) |
| .NET 9 | 2024-11 | STS | **2026-05-12** (8 gün kaldı 💀) |
| **.NET 10** | 2025-11 | **LTS** ⭐ | **2028-11** (2.5 yıl) |
| .NET 11 | 2026-11 | STS | 2028-05 |

Skillvio bugün (2026-05-04) yeni başlıyor — sıfırdan **LTS-to-LTS** bağlanmak,
production'a çıkmadan destek sonu yaşamamak için **.NET 10 doğru tercih**.

## Gerekçe vs Diğer Stack'ler

| Kriter | .NET 10 | Node/Express | Go |
|--------|---------|--------------|-----|
| Tip güvenliği | ✅ | ❌ (TS bile runtime'a sızar) | ✅ |
| ORM | ✅ EF Core 10 | 🟡 Drizzle/Prisma | 🟡 |
| Built-in DI | ✅ | ❌ | ❌ |
| Built-in Identity | ✅ ASP.NET Identity | ❌ | ❌ |
| Performance | ✅ +%12 baseline | 🟡 | ✅ |
| TR developer havuzu | ✅ Geniş | ✅ | 🟡 Dar |
| LTS | ✅ 2028-11 | 🟡 | 🟡 |
| Refactor güvenliği | ✅ | ❌ | ✅ |
| Native AOT | ⭐⭐ Olgun | ❌ | ✅ Native |

## .NET 10'un Bizim İçin Avantajları

1. **EF Core 10 — `complex types` + JSON columns daha güçlü**
   i18n `_translations` pattern + JSONB column mapping daha temiz
2. **OpenAPI built-in** (`Microsoft.AspNetCore.OpenApi`) — `Swashbuckle`'dan kurtuluş, daha az dependency
3. **Native AOT olgun** — gateway service için ~30MB image, ~50ms cold start
4. **Performance +%12** — aynı donanımda %12 daha çok throughput
5. **HybridCache** — `Microsoft.Extensions.Caching.Hybrid` (L1+L2 built-in, Redis önünde memory cache)
6. **`AddOpenApi()` minimal** — Sprint 1 API doc kısmı kısalır
7. **C# 14 features** — `field` keyword, primary constructors, daha az boilerplate

## ORM Seçimi — EF Core 10

**EF Core 10** seçildi. Alternatif: Dapper (raw SQL).

EF Core 10 avantajları:
- LINQ ile tip-güvenli sorgu
- Migration sistemi olgun
- `complex types` + owned types — DDD value object'leri için ideal
- `ExecuteUpdateAsync` / `ExecuteDeleteAsync` — bulk ops
- Snake_case naming convention paketi çalışır
- JSONB column mapping native
- Lazy/eager loading kontrolü

Dapper avantajı (performans) EF Core 10'da query splitting + AOT ile çoğunlukla
kapandı. Karmaşık raw query gerekirse `db.Database.SqlQueryRaw` ile düşülebilir.

## Stack Bileşenleri

```
.NET 10                                  # SDK + runtime
├── ASP.NET Core 10                       # Web framework
│   ├── Minimal APIs
│   ├── Built-in OpenAPI
│   ├── HybridCache
│   └── Rate Limiting
├── EF Core 10
│   ├── Npgsql.EntityFrameworkCore.PostgreSQL
│   └── EFCore.NamingConventions (snake_case)
├── ASP.NET Core Identity (cookie + Redis session)
├── FluentValidation
├── MassTransit (RabbitMQ)
├── Serilog → OpenTelemetry sink
├── OpenTelemetry SDK (.NET native)
├── YARP (gateway service)
├── BCrypt.Net-Next
└── xUnit + Testcontainers + FluentAssertions
```

## Versiyon Stratejisi

- **.NET 10 LTS** (Kasım 2028'e kadar)
- Yıllık `.NET 11` STS skip → **.NET 12 LTS** (Kasım 2027) değerlendirilecek
- LTS-to-LTS migration zorunlu (3 yılda bir)
- C# 14 features kullan
- Nullable reference types **enabled**
- TreatWarningsAsErrors **enabled** (Application + Domain projelerinde)

## Risk & Hafifletme

| Risk | Olasılık | Hafifletme |
|------|----------|------------|
| Niş NuGet paketi .NET 10 desteklemiyor | Düşük | NuGet `>=net10.0` filter ile pre-check; fallback: `net8.0` target second |
| Tutorial / Stack Overflow daha çok .NET 8'de | Yüksek | C# 14 features minimal; çekirdek ASP.NET Core pattern aynı |
| AI assistant .NET 8 syntax veriyor | Orta | Prompt'ta `.NET 10` belirt; mapping yap |
| EF Core 10 breaking changes (vs 8/9) | Düşük | Sıfırdan başlıyoruz, irrelevant |

## Referanslar

- [.NET Release Cadence](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core)
- [.NET 10 What's New](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-10/)
- [EF Core 10 Release Notes](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew)
- [ASP.NET Core 10 New Features](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-10.0)
