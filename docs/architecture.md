# Mimari — Yüksek Seviye

## Genel Bakış

Skillvio bir **microservices** sistemidir. Her servis kendi kod tabanına, kendi
DB şemasına ve kendi deploy döngüsüne sahiptir. Servisler arası iletişim:

- **Senkron (HTTP/REST)** — kullanıcı isteklerinde Gateway → Service
- **Asenkron (event bus)** — domain event'leri RabbitMQ üzerinden yayılır

## Mimari Çizimi

```
                              ┌──────────────────────┐
                              │      Frontend        │
                              │  (React + Vite + TS) │
                              └──────────┬───────────┘
                                         │ HTTPS, cookies
                                         ▼
                              ┌──────────────────────┐
                              │   API Gateway        │  ← YARP (.NET 8)
                              │   skillvio-gateway   │  • Routing
                              │                      │  • Auth check
                              │                      │  • Rate limit
                              │                      │  • Request logging
                              └──┬─────┬────┬────┬───┘
                                 │     │    │    │
              ┌──────────────────┘     │    │    └─────────────┐
              ▼                        ▼    ▼                  ▼
       ┌────────────┐          ┌────────────┐         ┌────────────┐
       │  identity  │          │  learning  │  ◄──►   │    lab     │
       │  service   │          │  service   │         │  service   │
       └─────┬──────┘          └──────┬─────┘         └──────┬─────┘
             │                        │                      │
             ▼                        ▼                      ▼
       ┌──────────┐             ┌──────────┐           ┌──────────┐
       │ identity │             │ learning │           │   lab    │
       │    DB    │             │    DB    │           │    DB    │
       └──────────┘             └──────────┘           └──────────┘

                ┌───────────────────────────────┐
                │   skillvio-ai-service         │  ← Gemini orchestration
                │   (Redis cache only)          │
                └───────────────────────────────┘

                ┌───────────────────────────────┐
                │  skillvio-notification-svc    │  ← email, push
                └───────────────────────────────┘

                              ┌──────────────────────┐
                              │   Event Bus          │
                              │   (RabbitMQ / NATS)  │
                              └──────────────────────┘
                              ▲▲▲ Tüm servisler subscribe/publish
```

## Servis Sorumlulukları

| Servis | Domain | DB | Boyut |
|--------|--------|-----|-------|
| **gateway** | — | — | XS — 1 dosya YARP config |
| **identity** | Users, Auth, Certifications | identity_db | M |
| **learning** | Tracks, Lessons, Roadmaps, Exams, Attempts, Badges | learning_db | L |
| **lab** | Editor labs, sandbox, code execution | lab_db + Redis | M |
| **ai** | Gemini orchestration | Redis (cache) | S |
| **notification** | Email, push, webhook | notification_db | S |

## Per-Servis Stack

Her servis aynı stack'i kullanır:

- **Runtime:** .NET 8
- **Web:** ASP.NET Core Minimal APIs / Controllers
- **DB ORM:** EF Core 8 + Npgsql + EFCore.NamingConventions (snake_case)
- **Auth:** Cookie-based session, validated by Gateway
- **Validation:** FluentValidation
- **Logging:** Serilog → Loki / Seq
- **Tracing:** OpenTelemetry
- **Background jobs:** Hangfire (cron, retry)
- **Tests:** xUnit + Testcontainers + WebApplicationFactory

## Veri İzolasyonu

Her servis **kendi DB'sine** sahiptir. Servisler arası veri paylaşımı:

- **Read:** API çağrısı (örn. learning service → identity service `/users/{id}`)
- **Write:** Asla cross-service direct write — sadece event yayılır

## Cross-Cutting Concerns

| Concern | Çözüm |
|---------|-------|
| **Auth** | Identity verir, Gateway doğrular, servisler `X-User-Id` header'a güvenir |
| **Logging** | Serilog → her servis aynı log formatı |
| **Tracing** | OpenTelemetry W3C Trace Context propagation |
| **Metrics** | Prometheus + Grafana (`/metrics` endpoint) |
| **Secrets** | `.env` dev, Azure KeyVault / AWS Secrets Manager prod |
| **Service discovery** | İlk faz Docker DNS, ileride Kubernetes |

## Deploy Modeli

### Development
- Tüm servisler tek `docker-compose.yml` ile (skillvio-infra repo'sunda)
- Her servis kendi build'i

### Production
- Kubernetes (skillvio-infra altında manifests)
- Her servis ayrı Deployment + Service + Ingress
- Helm chart (ileride)

## Olay (Event) Sözlüğü

Şu event'ler RabbitMQ üzerinden yayılır:

| Event | Yayan | Tüketen |
|-------|-------|---------|
| `UserRegisteredEvent` | identity | notification (welcome), learning (default state) |
| `UserDeletedEvent` | identity | tüm servisler (kullanıcı verisi temizliği) |
| `CertificationVerifiedEvent` | identity | learning (badge), notification (email) |
| `LessonCompletedEvent` | learning | notification (badge earned) |
| `BadgeEarnedEvent` | learning | notification |
| `LevelUpEvent` | learning | notification |
| `LabSubmittedEvent` | lab | learning (XP) |
| `AttemptCompletedEvent` | learning | notification (sertifika sınav sonucu) |

## Mikroservise Geçiş Sebebi

Modüler monolith yerine baştan microservice tercih edildi çünkü:

1. **Lab service yoğun CPU/IO işi** yapacak (kod çalıştırma) — ayrı host'ta scale olmalı
2. **AI service rate limit + maliyet kontrolü** ister — ayrı kuyruk + worker pool
3. **Identity service security boundary** — diğer servislere giremez
4. **Takım büyüdüğünde repo bazlı bölme** kolay olur
5. **Deploy bağımsızlığı** — bir servisin bug'ı diğerini düşürmez

## Sınırlar

- ❌ **Distributed transactions yok** — Saga pattern ya da eventually consistent
- ❌ **Cross-service join yok** — sorgular tek servis-içi, ihtiyaç olursa BFF
- ❌ **Shared DB yok** — her servis kendi DB'si
- ✅ **Shared contracts** — `skillvio-shared-contracts` NuGet paketi (DTO + event şemaları)

## Referanslar

- [microservices.io](https://microservices.io/patterns/index.html) — Chris Richardson
- [Microsoft microservices architecture](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/)
- [.NET YARP](https://microsoft.github.io/reverse-proxy/)
- [MassTransit (event bus)](https://masstransit.io/)
