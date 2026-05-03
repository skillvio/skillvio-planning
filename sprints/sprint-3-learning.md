# Sprint 3 — Learning Service

**Süre:** 2 hafta
**Hedef:** İçerik tarafı (track/unit/lesson/activity/roadmap/exam/attempt/badge) tamamen yeni servise.

## Çıktılar
- `skillvio-learning-service` — clean architecture
- i18n hazır: tüm `_translations` tabloları
- ScoringService + BadgeService domain bazlı
- Lesson complete + attempt save + streak/hearts merkezi

## Görevler

### S3.T1 — Solution + clean architecture
- Skillvio.Learning.Api / Application / Domain / Infrastructure / Tests

### S3.T2 — Schema + i18n
Migration: `Init`
- `domains`, `domain_translations`
- `tracks`, `track_translations`
- `units`, `unit_translations`
- `lessons`, `lesson_translations`
- `activities`, `activity_translations`
- `roadmaps`, `roadmap_translations`
- `roadmap_nodes`, `node_translations`
- `node_resources`
- `exams`, `exam_translations`
- `exam_questions`
- `attempts`, `lesson_progress`, `node_progress`, `unit_progress`
- `badges` (domain + cat + rarity), `user_badges`
- `day_streaks`, `hearts`
- `glossary_terms`, `glossary_translations`

### S3.T3 — Domain logic
- `ScoringService` — XP hesabı (eski Node `lib/scoring.js` portu)
- `BadgeService` — domain bazlı rozet eval (50+ rozet)
- `LessonProgressService`
- `LocalizedQueryService<T>` (locale fallback)

### S3.T4 — Endpoints
- `GET /api/learning/domains`
- `GET /api/learning/tracks?domain=cloud`
- `GET /api/learning/tracks/{id}` (locale-aware)
- `GET /api/learning/lessons/{id}`
- `POST /api/learning/lessons/{id}/complete`
- `GET /api/roadmaps?domain=...`
- `GET /api/roadmaps/{id}` + tree
- `GET /api/roadmaps/nodes/{id}`
- `POST /api/roadmaps/nodes/{id}/progress`
- `GET /api/me/streak`
- `GET /api/me/hearts`
- `GET /api/glossary?q=&category=`
- `GET /api/exams`, `GET /api/exams/{id}/questions`
- `POST /api/attempts`, `GET /api/attempts?exam_id=`

### S3.T5 — Event consume + publish
- Consume: `UserRegisteredEvent` → default streak/hearts
- Consume: `UserDeletedEvent` → tüm content data temizliği
- Publish: `LessonCompletedEvent`, `BadgeEarnedEvent`, `LevelUpEvent`,
  `AttemptCompletedEvent`

### S3.T6 — Hangfire jobs
- Hearts auto-refill (4 saatte 1)
- Day streak gece bitirme
- Eski draft attempt temizliği

### S3.T7 — Test
- xUnit + Testcontainers
- ScoringService unit tests (table-driven)
- BadgeService unit tests
- Integration: complete a lesson, badge earned, event published

### S3.T8 — Frontend bağlantı
- Gateway routes eklenir (`learn` cluster)
- Frontend dataSource zaten REST aynı path'ler

## Tamamlanma Kriteri
- [ ] Tüm content endpoint'leri çalışır, locale-aware
- [ ] BadgeService 50+ rozeti doğru evaluate eder (test ile kanıtlı)
- [ ] Streak gece doğru ilerler/sıfırlanır
- [ ] Hearts 4 saatte 1 yenilenir
- [ ] Lesson complete event publish ediliyor
- [ ] %75+ test coverage
