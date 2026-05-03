# ADR 0002 — Microservices Mimarisi

- **Status:** Accepted
- **Date:** 2026-05-04

## Bağlam

Mevcut backend (Node.js + Express monolith) `backend/` klasöründe. Yeni mimaride
3 seçenek vardı:

1. **Modüler monolith** — tek deployable, bounded context'ler kod içinde
2. **Microservices** — her bounded context ayrı servis + DB
3. **Hibrit (start monolith → split later)** — Martin Fowler önerisi

## Karar

**Microservices** — 6 backend servisi:
- gateway, identity, learning, lab, ai, notification

## Gerekçe

1. **Lab service yoğun CPU/IO** yapacak (kod çalıştırma) — ayrı host'ta scale
2. **AI service rate limit + maliyet kontrolü** — ayrı kuyruk + worker pool
3. **Identity güvenlik sınırı** — diğer servislere giremez
4. **Takım büyüdüğünde** — repo bazlı bölünme doğal
5. **Deploy bağımsızlığı** — bir servisin bug'ı diğerini düşürmez
6. **Polyglot persistence** — her servis kendi DB optimizasyonu (ileride)

## Trade-off'lar

### Pozitif
- Bağımsız deploy
- Kaynak izolasyonu (CPU/RAM)
- Teknoloji esnekliği (gerekirse Lab service Go ile yazılabilir)
- Açık takım ownership

### Negatif (& hafifletme)
- **Distributed system karmaşıklığı** → OTel tracing zorunlu
- **Eventual consistency** → Saga pattern, idempotency
- **Network latency** → gateway'in iyi config'i + caching
- **Operational overhead** → Kubernetes + Helm chart ile standardize
- **Veri tutarlılığı** → event-driven, outbox pattern

## Sınırlar

- ❌ Distributed transactions yok
- ❌ Cross-service join yok (BFF veya event-driven aggregation)
- ❌ Shared DB yok
- ✅ Shared contracts (NuGet package)

## Microservice Olmama Sebebi (önceden tartışıldı, reddedildi)

Daha önce "modüler monolith → split later" önerisi vardı. Reddedildi çünkü:
1. Lab service trafiği baştan tahmin ediliyor (yüksek)
2. Microservice tooling olgun (Docker Compose dev, K8s prod)
3. Erken bölünme migration riskini düşürür
4. Takım tek kişi bile olsa, mimari servis-kendi-repo daha temiz

## İlk Faz Sınırlaması

İlk MVP'de:
- Tek Postgres instance, multi-database (`identity_db`, `learning_db`, …)
- Tek Redis instance, multi-database (DB index ile)
- Tek RabbitMQ broker
- Single-node Kubernetes (Hetzner CX21 yeter)

Trafiğe göre yatay ölçek planlanır.
