# ADR 0003 — .NET 8 + EF Core

- **Status:** Accepted
- **Date:** 2026-05-04

## Karar

Backend .NET 8 + EF Core 8 + Postgres.

## Gerekçe

| Kriter | .NET 8 | Node/Express | Go |
|--------|--------|--------------|-----|
| Tip güvenliği | ✅ | ❌ (TS bile runtime'a sızar) | ✅ |
| ORM | ✅ EF Core | 🟡 Drizzle/Prisma | 🟡 |
| Built-in DI | ✅ | ❌ | ❌ |
| Built-in Identity | ✅ ASP.NET Identity | ❌ | ❌ |
| Performance | ✅ | 🟡 | ✅ |
| TR developer havuzu | ✅ Geniş | ✅ | 🟡 Dar |
| LTS | ✅ Microsoft | 🟡 | 🟡 |
| Refactor güvenliği | ✅ | ❌ | ✅ |

.NET 8 LTS (Kasım 2026'a kadar destekli, sonra .NET 10 LTS migration).

## ORM Seçimi

**EF Core 8** seçildi. Alternatif: Dapper (raw SQL).

EF Core avantajları:
- LINQ ile tip-güvenli sorgu
- Migration sistemi olgun
- Lazy/eager loading kontrolü
- Snake_case naming convention paketi var

Dapper avantajı: performans (~%20 daha hızlı). Ama EF Core 8'de query-splitting,
ExecuteUpdateAsync gibi optimizasyonlarla fark daraldı. Karmaşık query'lerde
Dapper'a düşülebilir (`db.Database.SqlQueryRaw`).

## Versiyon Stratejisi

- .NET 8 (LTS — Kasım 2026)
- Yıllık LTS migrate (.NET 10 → 2027)
- C# 12 features kullan (primary constructor, collection expressions)
- nullable reference types enabled
