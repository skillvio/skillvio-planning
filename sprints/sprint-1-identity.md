# Sprint 1 — Identity Service

**Süre:** 2 hafta
**Hedef:** `skillvio-identity-service` tamamen .NET 8'de, mevcut frontend bu servisle çalışsın.

## Çıktılar

- Identity service çalışıyor (auth + users + certifications)
- Cookie-based session, Redis store
- Mevcut Node backend'i durduruldu, frontend yeni servisle konuşuyor
- xUnit + Testcontainers ile %70+ test coverage
- `UserRegisteredEvent`, `UserDeletedEvent` event'leri RabbitMQ'ya publish ediliyor

## Görevler

### S1.T1 — Solution kurulumu
```bash
cd skillvio-identity-service
dotnet new sln -n Skillvio.Identity
dotnet new webapi -n Skillvio.Identity.Api
dotnet new classlib -n Skillvio.Identity.Application
dotnet new classlib -n Skillvio.Identity.Domain
dotnet new classlib -n Skillvio.Identity.Infrastructure
dotnet new xunit -n Skillvio.Identity.Tests
dotnet sln add **/*.csproj
```

### S1.T2 — NuGet paketleri
- `Microsoft.EntityFrameworkCore.Design`
- `Npgsql.EntityFrameworkCore.PostgreSQL`
- `EFCore.NamingConventions` (snake_case)
- `Microsoft.AspNetCore.Identity.EntityFrameworkCore`
- `StackExchange.Redis`
- `Microsoft.AspNetCore.DataProtection.StackExchangeRedis`
- `FluentValidation.AspNetCore`
- `Swashbuckle.AspNetCore`
- `Serilog.AspNetCore` + `Serilog.Sinks.Seq`
- `OpenTelemetry.Extensions.Hosting`
- `MassTransit.AspNetCore` + `MassTransit.RabbitMQ`
- `BCrypt.Net-Next`

### S1.T3 — Domain modelleri
- `User` (Id Guid, Username, DisplayName, Email, PasswordHash, Role, AvatarColors,
  UiLocale, PreferredLocale, OnboardingCompleted, Goal, ExperienceLevel, …)
- `RefreshToken`
- `Certification` + `CertificationStatus` enum
- `UserInterest` (UserId + DomainId)
- Domain events: `UserRegistered`, `UserDeleted`, `CertificationVerified`

### S1.T4 — EF Core configurations + migration
- `IdentityDbContext` ile snake_case naming convention
- `IEntityTypeConfiguration<T>` per entity
- Initial migration: `Init`
- `users.role` text constraint (admin/user)

### S1.T5 — Application layer (services)
- `IAuthService` (Signup, Login, Logout, RefreshToken)
- `IUserService` (Update, Delete, GetMe)
- `ICertificationService` (Submit, List, ApproveReject)
- `IPasswordHasher` → BCrypt implementation
- FluentValidation rules (signup, login, profile patch)

### S1.T6 — API Controllers / Minimal APIs
- `POST /api/auth/signup`
- `POST /api/auth/login`
- `POST /api/auth/logout`
- `GET  /api/auth/me`
- `PATCH /api/users/me`
- `DELETE /api/users/me`
- `GET /api/certifications/me`
- `POST /api/certifications`
- `DELETE /api/certifications/{id}`
- `GET /api/certifications/all` (admin)
- `PATCH /api/certifications/{id}/status` (admin)
- `POST /api/onboarding`
- `GET /api/me/interests`

### S1.T7 — Cookie + Redis session
- `services.AddDataProtection().PersistKeysToStackExchangeRedis(...)`
- Cookie auth: `forge.sid` (HttpOnly, SameSite=Lax, 30 gün)
- Redis distributed cache + session store

### S1.T8 — Event bus publish
- MassTransit + RabbitMQ
- `UserRegisteredEvent`, `UserDeletedEvent` publish

### S1.T9 — Test
- xUnit + FluentAssertions
- WebApplicationFactory + Testcontainers (Postgres + Redis)
- Auth happy path: signup → login → me → logout
- 5 sad path: yanlış şifre, yetersiz validation, duplicate username, …

### S1.T10 — Dockerfile + CI
- Multi-stage Dockerfile
- GitHub Actions: `dotnet test`, `dotnet publish`, image push to GHCR
- `docker-compose.dev.yml` skillvio-infra'da identity'yi çalıştırır

### S1.T11 — Frontend bağlantı testi
- Mevcut Node backend'i durdur
- Frontend `apiClient.js` aynı URL pattern'ları
- Manual smoke test: signup → login → profile → logout

## Tamamlanma Kriteri

- [ ] Tüm endpoint'ler integration test ile geçer
- [ ] BCrypt+pepper password hash
- [ ] Argon2id alternative değerlendirildi (BCrypt yerine ileride)
- [ ] Swagger UI: `/swagger` çalışıyor
- [ ] Health check: `/health/live`, `/health/ready`
- [ ] Frontend tüm auth akışlarında çalışıyor
- [ ] CI yeşil, image GHCR'da
- [ ] CHANGELOG.md güncel

## Risk

- BCrypt JS migration (eski hash'ler) — kullanıcı login olduğunda yeniden hashle
  (BCrypt → Argon2id ileride)
- Session zamanlaması — Redis kapalıysa cookie geçersiz, fallback tasarımı

## Süre

| Görev | Süre |
|-------|------|
| Solution + paketler | 1 gün |
| Domain + EF | 2 gün |
| Application + API | 3 gün |
| Cookie + Redis | 1 gün |
| Event publish | 1 gün |
| Test | 2 gün |
| Docker + CI + smoke | 1 gün |
| **TOPLAM** | **11 gün (~2 hafta)** |
