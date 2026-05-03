# ADR 0004 — Postgres + Redis

- **Status:** Accepted
- **Date:** 2026-05-04

## Karar

- **Postgres 16** (her servis kendi database)
- **Redis 7** (cache + session + rate limit + RabbitMQ alternatifi BullMQ)

## Postgres Gerekçe

- ACID
- JSONB (activity payloads, content metadata için ideal)
- Full-text search hazır (Türkçe tsvector için `unaccent` extension)
- Replikasyon olgun (logical replication)
- TR developer havuzu geniş
- pgvector eklenebilir (AI embedding ileride)

Alternatifler reddedildi:
- **MongoDB:** transactions zayıf, ilişkisel content zor
- **MySQL:** JSON desteği daha yeni, JSONB Postgres'te daha güçlü
- **CockroachDB:** ölçek için aşırı, başlangıçta gereksiz

## Redis Gerekçe

- Session storage (cookie session validation)
- AI response cache (Gemini yanıtları)
- Rate limit (token bucket)
- Distributed lock (idempotency için)
- Pub/sub (gerektiğinde, RabbitMQ light-weight alternatifi değil)

Alternatifler reddedildi:
- **Memcached:** persistence yok
- **KeyDB:** Redis fork, ekstra fayda yok

## DB-Per-Service Modeli

```
postgres:
  databases:
    identity_db    ← skillvio-identity-service
    learning_db    ← skillvio-learning-service
    lab_db         ← skillvio-lab-service
    notification_db ← skillvio-notification-service

redis:
  db 0  ← session
  db 1  ← cache
  db 2  ← rate limit
```

Her servis sadece kendi DB'sine bağlanır. Cross-service veri sadece event-driven.

## Production

- Managed: **Neon** (Postgres serverless) veya **Supabase**
- Self-hosted: **Hetzner Cloud + pgBackRest**
- Replikasyon: read replica `learning_db` (en çok okuma var)
- Backup: günlük pg_dump, S3'e yükle
