# Sprint 5 — AI Service + Event Bus + Notification

**Süre:** 2 hafta
**Hedef:** Gemini server-side, RabbitMQ aktif, notification service skeleton.

## Çıktılar
- `skillvio-ai-service` (Gemini orchestration + Redis cache)
- `skillvio-notification-service` (email)
- Event bus (RabbitMQ) tüm servisler arasında akıyor

## Görevler

### S5.T1 — AI Service
- `Skillvio.Ai.Api` — Minimal API
- `IGeminiClient` (mockable)
- Endpoints:
  - `POST /api/ai/explain` (soru → açıklama)
  - `POST /api/ai/tutor` (sohbet, "daha basit anlat")
  - `POST /api/ai/translate` (admin için içerik çevirisi)
- Per-user rate limit (Redis token bucket)
- Server-side cache (qid + locale, 30 gün TTL)
- Fallback chain modeller: gemini-2.0-flash-lite → gemini-2.5-flash-lite → ...

### S5.T2 — RabbitMQ kurulum
- skillvio-infra docker-compose'a eklendi
- MassTransit configured per service
- Sayar: management UI 15672

### S5.T3 — Notification Service
- `Skillvio.Notification.Api`
- DB: `notification_log`, `email_templates`, `user_preferences`
- Razor templates (Welcome, BadgeEarned, CertificationVerified)
- SMTP entegrasyonu (Mailtrap dev, SendGrid prod)
- Endpoints (admin-only):
  - `GET /api/notifications/log`
  - `POST /api/notifications/test` (template preview)

### S5.T4 — Event consumers
- Notification:
  - `UserRegisteredEvent` → welcome email
  - `BadgeEarnedEvent` → email (kullanıcı tercihine göre)
  - `CertificationVerifiedEvent` → email
- Learning:
  - `CertificationVerifiedEvent` → multi-cloud / cloud_first_pass badge eval
  - `LabSubmittedEvent` → XP credit

### S5.T5 — Event reliability
- Outbox pattern (event publish başarısız olursa retry)
- Dead letter queue
- Idempotency: consumer event ID'sini kontrol eder

## Tamamlanma Kriteri
- [ ] Yeni kullanıcı signup → welcome email gelir
- [ ] Badge kazanılınca email tetiklenir
- [ ] AI explain endpoint çalışır + cache hit/miss metrik
- [ ] RabbitMQ DLQ aktif, retry policy test edildi
