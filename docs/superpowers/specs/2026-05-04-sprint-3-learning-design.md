# Sprint 3 — Learning Service Design Spec

- **Status:** Draft (awaiting user written-spec review)
- **Date:** 2026-05-04
- **Author:** Skillvio (brainstormed via Superpowers)
- **Spans:** Sprint 3a (4 hafta) + Sprint 3b (1.5 hafta)
- **Effort:** ~157 saat solo dev = ~5.5 hafta
- **Supersedes:** `sprints/sprint-3-learning.md` outline

> Bu doküman Sprint 3 implementasyonunun kararlarını ve sınırlarını çizer. 10 brainstorming sorusu + 4 ek genişletme ile şekillendi. Yazma planı (implementation plan) bu spec onaylandıktan sonra `writing-plans` skill'i ile üretilecek.

---

## 1. Genel Bakış

`skillvio-learning-service` öğrenme platformunun **kalbi**dir. Sorumlulukları:

- **İçerik:** Track / Unit / Lesson / Activity / Roadmap / Exam / Glossary / Badge / Certificate
- **İlerleme:** Attempt (lesson + exam), per-track progress, lesson progress
- **Gamification:** XP / Level / Hearts / Streak / Badge / League
- **Sosyal:** Friends / Activity feed / Friend leaderboard
- **Çoklu dil (i18n):** TR/EN/DE/AR + locale fallback chain
- **Cloud sertifika prep:** AWS/Azure/GCP 30 sertifika, paid exam simulator, Skillvio prep certificate
- **Cross-service:** Outbox-based event publish + idempotent consumer (Identity/Lab/Payment/AI)
- **Real-time:** WebSocket (SignalR) push for XP/badge/streak/friend/league events

### 1.1 Stack

- .NET 10 LTS + ASP.NET Core 10 Minimal APIs (ADR 0003)
- EF Core 10 + Npgsql + EFCore.NamingConventions (snake_case)
- HybridCache (built-in L1+L2)
- Postgres 16 + Redis 7 + RabbitMQ (Quorum Queues prod)
- MassTransit (Outbox + Consumer)
- Hangfire (Postgres storage)
- SignalR + Redis backplane
- Serilog → Loki, OpenTelemetry → Tempo, Prometheus
- FluentValidation, JsonSchema.Net (content validation)
- xUnit + Testcontainers + FluentAssertions

### 1.2 Brainstorming Karar Tablosu

| # | Konu | Karar |
|---|------|-------|
| Q1 | Content source of truth | Git-based (`skillvio-content` repo, Crowdin → PR) |
| Q2 | Tracks/Roadmaps/Lessons | Lesson shared (atomic), Roadmap node lesson'a link, **Exam bağımsız domain**, **Skillvio Certificate** |
| Q3 | Activity polymorphism | Hybrid: common cols + JSONB content + JSON Schema validation in pipeline |
| Q4 | XP scoring | Dynamic: base × accuracy × streak + perfect bonus, config-driven |
| Q5 | Hearts & Streak | Hybrid hearts (config per content) + Streak freeze (Pro 2/ay) + Repair (48h) |
| Q6 | Badge evaluation | Two-tier: sync hot-path + async worker, definitions in Git |
| Q7 | i18n storage | Hybrid: short→JSONB, long→`_translations`, fallback chain user→content→en→first |
| Q8 | Attempt persistence | Hybrid: in_progress create + debounced live-save + idempotent final commit |
| Q9 | Multi-track | Global hearts/streak/level + Per-track XP/progress, max 5 active |
| Q10 | Cross-service events | Outbox pattern + consumer idempotency (`processed_events`) + DLQ |
| +1 | Cloud cert catalog | AWS 10 + Azure 12 + GCP 8 = **30 sınav** |
| +2 | Yeni feature scope | Real-time (SignalR) + Social (Friends) + Leagues |
| +3 | Sprint bölünmesi | Sprint 3a (core, 4 hafta) + Sprint 3b (social/league, 1.5 hafta) |
| +4 | Görev granülaritesi | 18 task, T7/T11/T18 split, T16 = 15h |

---

## 2. Solution Yapısı (Clean Architecture)

```
skillvio-learning-service/
├── src/
│   ├── Skillvio.Learning.Domain/                  # Pure C#, no deps
│   │   ├── Content/
│   │   │   ├── Tracks/        (Track, Unit, UnitLesson)
│   │   │   ├── Lessons/       (Lesson, Activity)
│   │   │   ├── Roadmaps/      (Roadmap, RoadmapNode, NodeResource)
│   │   │   ├── Exams/         (Exam, ExamQuestion)
│   │   │   └── Glossary/      (GlossaryTerm)
│   │   ├── Learning/
│   │   │   ├── Attempts/      (Attempt, AttemptAnswer, ExamAttemptMeta)
│   │   │   ├── Progress/      (User*Progress)
│   │   │   └── ActiveSelections/
│   │   ├── Gamification/
│   │   │   ├── Xp/            (XpTransaction, ScoringRule)
│   │   │   ├── Hearts/
│   │   │   ├── Streaks/       (DayStreak, StreakRepair)
│   │   │   ├── Levels/
│   │   │   └── Badges/
│   │   ├── Certificates/      (CertificateDefinition, UserCertificate)
│   │   ├── Social/            (Friendship, ActivityFeed, LeagueSeason, LeagueMembership)
│   │   └── Common/            (ValueObjects, DomainEvents, Specifications, Abstractions)
│   │
│   ├── Skillvio.Learning.Application/
│   │   ├── Content/Queries/
│   │   ├── Progress/Commands/
│   │   ├── Gamification/      (ScoringService, HeartsService, StreakService, LevelService, BadgeRuleEngine)
│   │   ├── Certificates/      (CertificateEligibilityService)
│   │   ├── Social/            (FriendshipService, ActivityFeedService, LeagueService)
│   │   ├── Localization/      (LocalizedQueryService, FallbackResolver)
│   │   ├── Events/Inbound/
│   │   ├── Events/Outbound/
│   │   ├── Common/            (IdempotencyService, ContentVersionResolver)
│   │   └── Abstractions/      (IRepositories, IOutbox, IProcessedEvents, IClock, ILocalizationContext)
│   │
│   ├── Skillvio.Learning.Infrastructure/
│   │   ├── Persistence/       (LearningDbContext, Configurations, Migrations, Repositories)
│   │   ├── Messaging/         (MassTransit consumers, OutboxDispatcher hosted service)
│   │   ├── Jobs/              (Hangfire definitions)
│   │   ├── Localization/      (JsonbLocaleConverter)
│   │   └── Cache/             (HybridCache wrapper)
│   │
│   ├── Skillvio.Learning.Api/                     # ASP.NET Core 10 Minimal APIs
│   │   ├── Endpoints/         (Map*Endpoints classes)
│   │   ├── Filters/           (LocaleFilter, IdempotencyFilter, ContentVersionHeader)
│   │   ├── Validators/        (FluentValidation)
│   │   ├── Middleware/        (CorrelationId, RequestLogging)
│   │   ├── Hubs/              (MeHub : Hub) — SignalR
│   │   ├── Program.cs
│   │   └── appsettings.json
│   │
│   ├── Skillvio.Learning.ContentSync/             # Standalone console
│   │   ├── Pipeline/          (Parse → Validate → Diff → Apply)
│   │   ├── Validators/        (JSON Schema)
│   │   ├── Transformers/      (YAML/JSON → Domain entities)
│   │   ├── Webhook/           (HMAC verification + idempotency)
│   │   └── Program.cs
│   │
│   └── Skillvio.Learning.BadgeWorker/             # Standalone consumer (async tier)
│       └── Program.cs
│
├── tests/
│   ├── Skillvio.Learning.Domain.Tests/            # Mutation-test ready (pure functions)
│   ├── Skillvio.Learning.Application.Tests/
│   └── Skillvio.Learning.IntegrationTests/        # Testcontainers (Postgres + Redis + RabbitMQ)
│
├── docker/
│   ├── Dockerfile.api
│   ├── Dockerfile.contentsync
│   ├── Dockerfile.badgeworker
│   └── docker-compose.test.yml
│
└── skillvio-learning-service.sln
```

> **NOT:** Local `Skillvio.Learning.Contracts` project'i **YOKTUR**. Cross-service event ve DTO'lar `skillvio-shared-contracts` NuGet paketi üzerinden tüketilir.

### 2.1 Boundaries

- `Domain` ↔ hiçbir altyapıya bağımlı değil
- `Application` → `Domain`'e bağlı; abstract dependency'ler tanımlar
- `Infrastructure` → `Application` abstract'larını implement eder
- `Api`, `ContentSync`, `BadgeWorker` → composition root'lar
- `Application` → `Infrastructure`'a **direct reference yok**

---

## 3. Database Schema (Postgres 16, snake_case)

### 3.1 Standard kolonlar (her content tablosunda)

```
id                  uuid PK
status              text NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','review','published','archived'))
archived_at         timestamptz NULL
min_app_version     text NULL
source_path         text NOT NULL          -- 'tracks/cloud/aws-temelleri.yml'
source_commit_sha   text NOT NULL          -- ingestion versioning
content_hash        text NOT NULL
schema_version      smallint DEFAULT 1
published_at        timestamptz NULL
created_at, updated_at timestamptz
```

### 3.2 Content (authoring)

#### `domains`
- code (UNIQUE), parent_domain_id (self-FK), display_order, icon_url, color_hex
- short_i18n JSONB (name, tagline)
- *Hierarchy:* `cloud > aws | azure | gcp`

#### `tracks`
- domain_id, slug (UNIQUE), difficulty (1-5), est_minutes
- hearts_mode (`'strict' | 'lenient'`) — Q5
- default_locale, is_premium
- prerequisites uuid[]
- target_exam_id uuid NULL FK→exams (Q2)
- short_i18n JSONB

#### `track_translations`
- track_id, locale, description_md, learning_outcomes_md (PK composite)

#### `units`
- track_id, display_order, short_i18n
- unlock_rule JSONB (`{type: 'sequential'|'free'|'xp_threshold', value: N}`)

#### `lessons` (atomic content, paylaşılır — Q2)
- slug (UNIQUE), est_minutes, default_locale, difficulty
- hearts_mode override
- short_i18n

#### `lesson_translations`
- lesson_id, locale, body_md, hint_md, summary_md (PK composite)

#### `unit_lessons` (junction, ordered)
- unit_id, lesson_id, display_order (PK composite)

#### `activities` (Q3 hybrid)
- lesson_id, display_order, type
- content JSONB (type-specific schema validated by content pipeline)
- scoring JSONB (`base_xp, partial_credit, max_attempts`)
- metadata JSONB (`difficulty, tags[], min_time_ms`)

**Activity tipleri:** `multiple_choice, multi_select, fill_blank, match_pairs, order_steps, code_snippet, lab_launch, flashcard`

#### `activity_translations`
- activity_id, locale, prompt_md, explanation_md, hint_md (PK composite)

#### `roadmaps`
- domain_id, slug, default_locale, difficulty
- target_exam_id uuid NULL FK→exams
- short_i18n

#### `roadmap_translations`
- roadmap_id, locale, description_md, prerequisites_md (PK composite)

#### `roadmap_nodes`
- roadmap_id, parent_node_id (self-FK), display_order
- node_type (`'topic'|'subtopic'|'milestone'|'cert_target'`)
- lesson_id NULL FK→lessons (Q2 shared)
- short_i18n

#### `node_translations`
- node_id, locale, title, description_md (PK composite)

#### `node_resources` (external links)
- node_id, resource_type (`article|video|official_doc|youtube`), url, est_minutes
- short_i18n (title), display_order

#### `glossary_terms`
- domain_id, slug (UNIQUE), difficulty
- related_lesson_ids uuid[]
- short_i18n (term name)

#### `glossary_translations`
- term_id, locale, definition_md, examples_md
- search_tsvector tsvector — GIN index for full-text
- (PK composite)

#### `exams` (Q2 paid + cloud cert)
- domain_id, slug (UNIQUE)
- cert_provider (`'aws'|'azure'|'gcp'|'skillvio'`)
- cert_code (`'DVA-C02'`)
- exam_format (`'aws_associate'|'azure_900'|'gcp_professional'|...`)
- scoring_scheme (`'percentage_72'|'scaled_700_1000'|'percentage_70'`)
- question_count_per_attempt, time_limit_minutes, passing_score_pct
- is_premium, one_time_price_cents NULL
- allow_review bool DEFAULT true
- show_score_after bool DEFAULT true
- domain_breakdown JSONB (exam blueprint)
- expires_at timestamptz NULL (sunset DVA-C01 → DVA-C02)
- short_i18n

#### `exam_translations`
- exam_id, locale, description_md, exam_tips_md (PK composite)

#### `exam_questions`
- exam_id, question_type (`single|multi|scenario`), difficulty
- exam_domain_code (`'deploy'|'security'|...`)
- multi_select_count NULL
- topic_tags text[]
- content JSONB (`stem, options[], correct[], explanation_ref`)
- scenario_context text NULL
- reference_links text[]
- explanation_video_url text NULL
- flagged_for_review bool, last_verified_at timestamptz, expires_at timestamptz NULL
- source_ref text (internal traceability)

#### `exam_question_translations`
- question_id, locale, stem_md, options_md text[], explanation_md (PK composite)

#### `certificate_definitions` (Skillvio Certificate — Q2)
- code (UNIQUE), domain_id
- trigger_rule JSONB (`{track_completed?, roadmap_completed?, exam_passed?, min_score?}`)
- artwork_url, is_verifiable bool
- short_i18n

#### `certificate_translations`
- cert_id, locale, title, description_md (PK composite)

#### `badges` (Q6)
- code (UNIQUE), domain_id NULL (null = cross-domain)
- category (`volume|streak|perfect|exam|social|league`)
- rarity (`common|rare|epic|legendary`)
- tier (`'sync'|'async'`)
- trigger_event (`'LessonCompletedEvent'`)
- predicate_kind (`first_occurrence|count_ge|streak_ge|score_ge|composite`)
- predicate_args JSONB
- xp_reward int, repeatable bool
- artwork_url, short_i18n
- definition_hash text

#### `badge_translations`
- badge_id, locale, description_md (PK composite)

### 3.3 User Progress

#### `user_active_tracks` / `user_active_roadmaps` (Q9 max 5)
- user_id, target_id, started_at, display_position, is_pinned (PK composite)

#### `user_track_progress` / `user_roadmap_progress`
- user_id, target_id
- lessons_completed, lessons_total
- xp_earned
- progress_pct (GENERATED ALWAYS AS STORED)
- last_activity_at, completed_at NULL
- (PK composite)

#### `user_node_progress`
- user_id, node_id, status (`pending|in_progress|done`), completed_at NULL (PK composite)

#### `user_lesson_progress`
- user_id, lesson_id, best_attempt_id NULL, perfect bool, completed_at NULL (PK composite)

#### `user_lab_completions` (Lab event projection — Bölüm 3)
- user_id, lab_id, completed_at, xp_earned, source_event_id (PK composite)

#### `user_preferences` (Bölüm 3 onboarding)
- user_id PK, daily_goal_minutes, preferred_domain_ids uuid[]
- reminder_time_local time
- social_visibility (`friends_only|public|private`)
- leaderboard_visible bool, friend_activity_visible bool
- auto_freeze_streak bool DEFAULT false
- notification_settings JSONB
- updated_at

#### `attempts` (Q8 hybrid)
- id PK, user_id, attempt_type (`lesson|exam`), target_id
- status (`in_progress|completed|abandoned|timed_out`)
- started_at, last_activity_at, completed_at NULL
- total_xp_earned, score_pct, hearts_lost
- config_snapshot JSONB (scoring rules version snapshot)
- client_metadata JSONB (timezone, app_version, device)
- device_session_id text (multi-tab conflict)
- idempotency_key uuid, idempotency_response JSONB NULL, idempotency_expires_at timestamptz NULL

#### `attempt_answers` (PARTITION BY RANGE created_at)
- attempt_id, answer_index (PK composite)
- activity_id NULL, exam_question_id NULL
- answer_payload JSONB
- is_correct, partial_credit
- time_spent_ms, submitted_at

#### `exam_attempt_meta`
- attempt_id PK FK, questions_total, questions_correct
- passing_score_pct, passed bool, certificate_issued bool
- shuffled_question_order uuid[]

#### `exam_access` (Q2 paid)
- user_id, exam_id (PK composite)
- granted_at, granted_via (`plan_pro|plan_team|one_time|gift`)
- expires_at timestamptz NULL, attempts_remaining int NULL

#### `user_badges` (Q6)
- user_id, badge_id (PK composite)
- earned_at, earned_count, context JSONB

#### `badge_evaluation_log` (PARTITION BY RANGE evaluated_at, 30-day retention)
- id, user_id, badge_id, evaluated_at, result (`earned|not_yet|error`), reason

#### `user_certificates`
- id PK, user_id, certificate_id, issued_at
- verification_code text UNIQUE
- context JSONB, revoked_at timestamptz NULL

### 3.4 Gamification

#### `user_xp`
- user_id PK, total_xp bigint, current_level, xp_to_next_level, updated_at

#### `xp_transactions` (PARTITION BY RANGE created_at, monthly)
- id, user_id, delta, source (`lesson|exam|lab|badge|streak_milestone|league_reward`)
- source_ref uuid NULL, multipliers JSONB
- created_at, trace_id

#### `hearts` (Q5)
- user_id PK, current, max, last_refill_at, last_loss_at
- unlimited_until timestamptz NULL (Pro plan grant)

#### `day_streaks` (Q5)
- user_id PK, current_streak, longest_streak
- last_active_date_local date, user_timezone text
- freezes_remaining_this_month, freeze_used_dates date[]
- monthly_quota_reset_at date

#### `streak_repairs`
- id PK, user_id, repaired_at, original_break_date
- repaired_streak_value
- cost_kind (`pro_monthly_free|paid_xp|paid_money`), cost_amount

#### `level_definitions` (Git-managed, ContentSync ile DB seed)
- level int PK, total_xp_required bigint
- perks JSONB, short_i18n

### 3.5 Social

#### `friendships`
- id PK, requester_id, addressee_id
- status (`pending|accepted|blocked`)
- created_at, responded_at NULL
- UNIQUE(requester_id, addressee_id)

#### `friend_activity_feed`
- id PK, user_id, activity_type, activity_ref
- occurred_at, visibility (`friends|public|private`)

#### `league_tiers` (Git-managed config)
- tier int PK (1-8: Bronz → Şampiyon)
- name_i18n JSONB
- promote_count, demote_count
- promote_reward_xp, artwork_url

#### `league_seasons` (weekly cohorts)
- id PK, tier, cohort_number, week_starts_at, week_ends_at
- status (`active|finalized`), participant_count

#### `league_memberships`
- user_id, season_id (PK composite)
- joined_at, weekly_xp
- final_rank NULL, promoted bool NULL, demoted bool NULL

### 3.6 Cross-cutting

#### `outbox_messages` (Q10, MassTransit Outbox)
- id PK, occurred_at
- event_type, payload JSONB
- aggregate_type, aggregate_id, causation_id NULL, correlation_id NULL
- trace_id
- dispatched_at NULL, retry_count, last_error NULL

#### `processed_events` (consumer idempotency)
- event_id, consumer_name (PK composite)
- processed_at

#### `content_imports`
- id PK, content_repo_commit_sha
- started_at, completed_at NULL
- status (`running|success_dry_run|success|failed`)
- entities_added/updated/unchanged
- errors JSONB

#### `idempotency_keys` (POST endpoint cache)
- key, scope (PK composite)
- response_payload JSONB, status_code, expires_at

### 3.7 GDPR/KVKK View

```sql
CREATE VIEW v_user_data_tables AS SELECT unnest(ARRAY[
  'attempts','attempt_answers','user_badges','user_certificates',
  'user_track_progress','user_roadmap_progress','user_node_progress',
  'user_lesson_progress','user_active_tracks','user_active_roadmaps',
  'hearts','day_streaks','streak_repairs','user_xp','xp_transactions',
  'exam_access','badge_evaluation_log','user_lab_completions',
  'user_preferences','friendships','friend_activity_feed',
  'league_memberships'
]) AS table_name;
```

`UserHardDeletedHandler` bu view'i okuyup tek noktadan cascade delete yapar.

### 3.8 Indexler (kritik)

```sql
CREATE INDEX ON tracks(domain_id, status, published_at);
CREATE INDEX ON lessons(slug);
CREATE INDEX ON activities(lesson_id, display_order);
CREATE INDEX ON unit_lessons(unit_id, display_order);
CREATE INDEX ON roadmap_nodes(roadmap_id, parent_node_id);
CREATE INDEX ON user_track_progress(user_id, last_activity_at DESC);
CREATE INDEX ON user_active_tracks(user_id) WHERE is_pinned = true;
CREATE INDEX ON attempts(user_id, attempt_type, status);
CREATE INDEX ON xp_transactions(user_id, created_at DESC);
CREATE INDEX ON user_badges(user_id, earned_at DESC);
CREATE INDEX ON exam_access(user_id);
CREATE INDEX ON friendships(addressee_id, status);
CREATE INDEX ON friendships(requester_id, status);
CREATE INDEX ON friend_activity_feed(user_id, occurred_at DESC);
CREATE INDEX ON league_memberships(season_id, weekly_xp DESC);
CREATE INDEX ON outbox_messages(occurred_at) WHERE dispatched_at IS NULL;

CREATE INDEX ON glossary_translations USING GIN(search_tsvector);
CREATE INDEX ON tracks USING GIN(short_i18n);
CREATE INDEX ON lessons USING GIN(short_i18n);
```

### 3.9 Partitioning

`xp_transactions`, `attempt_answers`, `badge_evaluation_log` aylık RANGE partition.

```sql
CREATE TABLE xp_transactions (...) PARTITION BY RANGE (created_at);
CREATE TABLE xp_transactions_2026_05 PARTITION OF xp_transactions
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

Hangfire `partition.create.monthly` her ayın 1'inde gelecek ay'ın partition'ını oluşturur. `partition.drop.old` 12 aydan eski partition'ları drop eder.

### 3.10 Postgres triggers

```sql
-- Outbox notify (Bölüm 6 #2)
CREATE FUNCTION outbox_notify() RETURNS trigger AS $$
BEGIN PERFORM pg_notify('outbox_new', NEW.id::text); RETURN NEW; END;
$$ LANGUAGE plpgsql;
CREATE TRIGGER outbox_notify_trg AFTER INSERT ON outbox_messages
  FOR EACH ROW EXECUTE FUNCTION outbox_notify();
```

---

## 4. Cloud Sertifika Sistemi

### 4.1 Desteklenen Sertifikalar (Sprint 3 launch)

#### AWS (10)
| Code | Adı | Tier | Soru | Süre | Pass |
|------|-----|------|------|------|------|
| CLF-C02 | Cloud Practitioner | Foundational | 65 | 90m | 70% |
| AIF-C01 | AI Practitioner | Foundational | 65 | 90m | 70% |
| SAA-C03 | Solutions Architect Associate | Associate | 65 | 130m | 72% |
| DVA-C02 | Developer Associate | Associate | 65 | 130m | 72% |
| SOA-C02 | SysOps Administrator Associate | Associate | 65 | 130m | 72% |
| MLA-C01 | ML Engineer Associate | Associate | 65 | 130m | 72% |
| DEA-C01 | Data Engineer Associate | Associate | 65 | 130m | 72% |
| SAP-C02 | SA Professional | Professional | 75 | 180m | 75% |
| DOP-C02 | DevOps Engineer Professional | Professional | 75 | 180m | 75% |
| SCS-C02 | Security Specialty | Specialty | 65 | 170m | 75% |

#### Azure (12)
AZ-900, AI-900, DP-900, SC-900, AZ-104, AZ-204, AZ-500, AZ-700, AZ-305, AZ-400, DP-203, AI-102.
Pass: 700/1000 scaled.

#### GCP (8)
CDL, ACE, PCA, PCDE, PDE, PMLE, PCSE, PCNE.
Pass: ~70%.

**Toplam: 30 sertifika sınavı** + 30 prep track + 30 prep roadmap + 30 Skillvio prep certificate.

### 4.2 Question Format

Soru tipleri (Sprint 3 scope): `single`, `multi`, `scenario`. `case_study` Sprint 5'te, `drag_drop` Sprint 6'da, `lab_simulation` Sprint v3+'da.

JSON Schema dosyaları `skillvio-content/schemas/exam-question-*.schema.json`.

### 4.3 Soru Iletim Pipeline'ı

```
[User] → Claude (sohbet, ExamTopics paste / CSV / JSON / raw)
       → q-NNNN.yml format'a çevrilir (EN+TR çift dil zorunlu)
       → skillvio-content PR
       → CI: JSON Schema validation + duplicate check
       → Merge to main
       → GitHub webhook → ContentSync trigger
       → HMAC verify + commit_sha idempotency
       → Postgres insert/update
       → HybridCache invalidate
       → ContentImportCompletedEvent → BadgeWorker recompute, Notification
```

**Telif notu (R3, OI4):** ExamTopics içerikleri **kopyalanmaz**. Yeniden ifade + farklı senaryo + Skillvio authoring. Mod ekibi review zorunlu. ADR `0015-exam-content-licensing.md` yazılacak.

### 4.4 Skillvio Certificate Trigger

```yaml
# certificates/skillvio-aws-dev-prep.yml
code: skillvio-aws-dev-prep
domain: aws
trigger_rule:
  all:
    - track_completed: aws-developer-prep
    - exam_passed: { exam: dva-c02, min_score_pct: 72 }
artwork_url: https://cdn.skillvio.io/certs/aws-dev-prep.svg
is_verifiable: true
```

Public verification URL: `app.skillvio.io/verify/{verification_code}` (auth-free, rate-limited 60/min/IP).

### 4.5 İçerik Repo Yapısı

```
skillvio-content/
├── schemas/                    # JSON Schemas
├── domains/                    # cloud, code, ai, db, devops, terms
├── exams/{provider}/{code}/
│   ├── exam.yml
│   ├── translations/{tr|en|de|ar}.yml
│   └── questions/q-NNNN.yml    # bir soru bir dosya
├── tracks/                     # exam'e hazırlık
├── roadmaps/
├── certificates/               # Skillvio prep cert
├── badges/
└── glossary/
```

---

## 5. API Endpoints (~70)

### 5.1 Convention

- Base: `/api/v1/...`
- Auth: gateway → `X-User-Id`, `X-User-Plan`, `X-User-Role` headers
- Locale: `Accept-Language` → `X-User-Locale` header
- Idempotency: state-changing endpoint'lerde `Idempotency-Key`
- Pagination: `?page=1&size=20`, `X-Total-Count` header
- Errors: RFC 7807 Problem Details
- Cache: ETag (`If-None-Match` 304)
- Response: `Content-Language`, `X-Content-Version` headers

### 5.2 Public Content (read, locale-aware)

```
GET  /api/v1/learning/domains
GET  /api/v1/learning/domains/{slug}
GET  /api/v1/learning/tracks?domain={slug}&difficulty={n}&page=1
GET  /api/v1/learning/tracks/{id}
GET  /api/v1/learning/tracks/{id}/lessons
GET  /api/v1/learning/lessons/{id}
GET  /api/v1/learning/roadmaps?domain={slug}
GET  /api/v1/learning/roadmaps/{id}
GET  /api/v1/learning/roadmaps/nodes/{id}
GET  /api/v1/learning/exams?domain={slug}
GET  /api/v1/learning/exams/{id}
GET  /api/v1/learning/exams/{id}/questions?attempt={id}    # auth + access
GET  /api/v1/learning/glossary?q={query}&domain={slug}&page=1
GET  /api/v1/learning/glossary/{slug}
GET  /api/v1/learning/badges
GET  /api/v1/learning/certificates
GET  /api/v1/learning/leagues/tiers
GET  /api/v1/learning/search?q={query}&type=all&domain={slug}    # federated
```

### 5.3 User Personalized (`/me/*`)

```
GET    /api/v1/me/dashboard
POST   /api/v1/me/onboarding
GET    /api/v1/me/tracks
POST   /api/v1/me/tracks/{trackId}/start
PATCH  /api/v1/me/tracks/{trackId}                # is_pinned, display_position
DELETE /api/v1/me/tracks/{trackId}
GET    /api/v1/me/tracks/{trackId}/progress
GET    /api/v1/me/roadmaps                        # aynı pattern
POST   /api/v1/me/roadmaps/{id}/start
PATCH  /api/v1/me/roadmaps/{id}
DELETE /api/v1/me/roadmaps/{id}
GET    /api/v1/me/lessons/{lessonId}/progress
GET    /api/v1/me/streak
POST   /api/v1/me/streak/freeze
POST   /api/v1/me/streak/repair
GET    /api/v1/me/hearts
GET    /api/v1/me/xp
GET    /api/v1/me/xp/history?page=1
GET    /api/v1/me/badges
GET    /api/v1/me/certificates
GET    /api/v1/me/certificates/verify/{code}      # public, no auth
GET    /api/v1/me/exam-access
GET    /api/v1/me/lab-attempts?page=1             # Lab event projection
GET    /api/v1/me/next-lesson                     # heuristic
GET    /api/v1/me/recommendations?source=heuristic|ai
GET    /api/v1/me/leaderboard?scope=global|domain|track&period=daily|weekly|all
GET    /api/v1/me/league
GET    /api/v1/me/league/cohort
GET    /api/v1/me/league/history
```

### 5.4 Social

```
GET    /api/v1/me/friends
GET    /api/v1/me/friends/requests
POST   /api/v1/me/friends/requests
POST   /api/v1/me/friends/requests/{id}/accept
POST   /api/v1/me/friends/requests/{id}/decline
DELETE /api/v1/me/friends/{userId}
POST   /api/v1/me/friends/{userId}/block
GET    /api/v1/me/friends/{userId}/profile
GET    /api/v1/me/friends/feed?page=1
GET    /api/v1/me/friends/leaderboard?period=weekly
GET    /api/v1/users/search?q={username}          # federated to Identity
```

### 5.5 Attempts (state-changing, idempotent)

```
POST   /api/v1/attempts                            # start lesson
GET    /api/v1/attempts/{id}                       # resume
POST   /api/v1/attempts/{id}/answers               # live save
POST   /api/v1/attempts/{id}/complete              # final saga
DELETE /api/v1/attempts/{id}                       # abandon
POST   /api/v1/attempts/exam/{examId}/start
POST   /api/v1/attempts/{id}/heartbeat             # exam anti-idle
```

### 5.6 Exam Access

```
POST   /api/v1/exam-access/purchase                # one-time (Stripe/Iyzico — Sprint 7)
POST   /api/v1/exam-access/redeem                  # gift code
GET    /api/v1/exam-access/check/{examId}
```

### 5.7 Admin (role-gated)

```
GET    /api/v1/admin/learning/content-imports
POST   /api/v1/admin/learning/content-imports
GET    /api/v1/admin/learning/content-imports/{id}
POST   /api/v1/admin/learning/tracks/{id}/publish      # status: review→published
POST   /api/v1/admin/learning/tracks/{id}/archive
POST   /api/v1/admin/learning/badges/{id}/recompute
GET    /api/v1/admin/learning/attempts/{id}/audit
POST   /api/v1/admin/learning/users/{userId}/grant-exam-access
POST   /api/v1/admin/learning/users/{userId}/revoke-exam-access
POST   /api/v1/admin/learning/dlq/replay/{messageId}
```

### 5.8 WebSocket

```
WS  /api/v1/ws/me     # cookie auth, gateway pass-through
```

Push event'ler: `xp_awarded`, `level_up`, `badge_earned`, `streak_warning`, `hearts_refilled`, `friend_request`, `friend_accepted`, `friend_milestone`, `league_promoted`, `league_demoted`, `league_ended`, `xp_overtaken`.

### 5.9 Health & Ops

```
GET /health/live
GET /health/ready
GET /metrics                     # Prometheus
GET /openapi.json                # built-in
```

### 5.10 Response shape örneği

```json
GET /api/v1/learning/tracks/{id}?Accept-Language=tr-TR
{
  "id": "8f3...",
  "slug": "aws-temelleri",
  "domain": { "code": "aws", "name": "AWS" },
  "title": "AWS Temelleri",
  "summary": "AWS bulut servislerine giriş",
  "difficulty": 2,
  "est_minutes": 240,
  "is_premium": false,
  "hearts_mode": "lenient",
  "target_exam": { "id": "...", "slug": "aws-dva-c02", "code": "DVA-C02", "is_premium": true },
  "units": [...],
  "_meta": {
    "locale": "tr-TR",
    "fallback_used": false,
    "content_version": "v2026.05.04-abc123"
  }
}
```

Headers: `Content-Language: tr-TR`, `ETag: "v2026.05.04-abc123"`, `X-Content-Version: v2026.05.04-abc123`.

---

## 6. Domain Logic (Application Services)

### 6.1 ScoringService (Q4)

**Pure function, no IO. Mutation-test ready.**

```csharp
public record ScoringContext(
    Guid UserId, Guid AttemptId,
    Activity Activity, AttemptAnswer Answer,
    int AttemptCount,           // 1=first try
    DayStreakSnapshot Streak,
    ScoringRules Rules);        // Git-versioned snapshot

public record XpCalculation(
    int BaseXp,
    decimal AccuracyMultiplier,
    decimal StreakBonus,
    int PerfectLessonBonus,
    int FinalXp,
    Dictionary<string,object> Breakdown);
```

**Formül (v1):**
```
final = base × accuracy × streak_bonus + perfect_bonus
```
- `accuracy`: 1.0 first-try, 0.5 second, 0.25 third+
- `streak_bonus`: 1.0 base, 1.1 if 7-day, 1.25 if 30-day
- `perfect_bonus`: +25 XP if 100% activities first-try

v1 disabled: hearts_intact_bonus, plan_multiplier (config flag, v2'de açılır).

### 6.2 AttemptOrchestrator (Q8)

State machine: `None → InProgress → (Completed | Abandoned | TimedOut)`.

**Methods:**
- `StartAsync` — multi-tab `device_session_id` check, max active limit
- `ResumeAsync` — state restore for `in_progress`
- `SubmitAnswerAsync` — server-side timing, anti-cheat threshold (`min_time_ms`)
- `CompleteAsync` — 14-step saga (§7)
- `AbandonAsync` + Hangfire `attempts.cleanup.abandoned`

**Idempotency:** Her command'da `Idempotency-Key`, `idempotency_response` JSONB cache.

### 6.3 BadgeRuleEngine (Q6 sync) + BadgeWorker (Q6 async)

**Sync tier (BadgeRuleEngine, in-process):**
- < 50ms target
- In-memory rule cache (startup load)
- Predicates: `first_occurrence`, `count_ge`, `streak_ge`, `score_ge`, `composite (all/any)`

**Async tier (BadgeWorker, ayrı project):**
- MassTransit consumer
- Aggregation queries (cross-domain, lab milestones, multi_cloud_master)
- Backfill mode (admin endpoint)

`user_badges` `INSERT ... ON CONFLICT DO NOTHING` (idempotent).

### 6.4 CertificateEligibilityService (Q2)

```csharp
Task<IReadOnlyList<CertificateAward>> EvaluateAsync(
    Guid userId, CertificateTrigger trigger, CancellationToken ct);
```

Rule trigger: `lesson_completed`, `exam_passed`. Trigger rule kompozit (track + exam combo).

### 6.5 StreakService (Q5)

- `RecordActivity(userId, localDate)` — increment, milestone detect
- `UseFreeze(userId, date)` — Pro plan check, quota decrement, idempotent per date
- `Repair(userId, originalBreakDate, RepairCost)` — 48h window, cost validation
- `EvaluateNightly(timezone, date)` — at-risk users, freeze auto-apply, break propagate

**Dynamic timezone evaluator** (Bölüm 6 #1): bootstrap'ta `DISTINCT user_timezone` alınır, her timezone için ayrı Hangfire recurring job register edilir.

### 6.6 HeartsService (Q5)

- `LoseHeart(userId, reason)` — strict mode'da yanlış cevap, lenient'ta sadece exam/lab
- `RefillIfDue(userId)` — Hangfire `hearts.refill` her 15dk
- `ApplyUnlimited(userId, plan)` — Pro plan grant `unlimited_until`

### 6.7 LeagueService

- `AssignWeeklyCohorts` — Fisher-Yates shuffle, 30-user cohorts
- `FinalizeWeek` — top promote, bottom demote, rewards, events
- `UpdateWeeklyXp(userId, delta)` — atomic `weekly_xp += delta`

### 6.8 FriendshipService

- `SendRequest`, `Accept`, `Decline`, `Unfriend`, `Block`
- Spam guard: 5/day, 50/week
- Privacy guard: `social_visibility` + blocked exclusion

### 6.9 LocalizedQueryService<T,TT> (Q7)

JSONB short_i18n + translations table JOIN with COALESCE fallback.

```csharp
Task<TEntity> GetWithLocaleAsync(Guid id, string locale, CancellationToken ct);
Task<IReadOnlyList<TEntity>> ListWithLocaleAsync(IQueryable<TEntity> query, string locale, CancellationToken ct);
```

HybridCache key: `entity:{type}:{id}:locale:{locale}`.

**Fallback chain:**
```
user.preferred_locale → content.default_locale → "en" → first available
```

### 6.10 ContentSyncOrchestrator (Q1=B)

```
1. Acquire pg_advisory_lock('content_sync_global')
2. Git clone/fetch skillvio-content @ commit_sha
3. Walk file tree → parse YAML/JSON
4. JSON Schema validate (per type)
5. Diff: { added, updated, renamed, archived } via content_hash + source_path
6. If dry_run → return plan
7. Else: TX BEGIN → apply diff → TX COMMIT
8. content_imports.completed_at, status='success'
9. Cache invalidate (HybridCache prefix delete)
10. ContentImportCompletedEvent → outbox
11. Release advisory lock
```

**Webhook**: HMAC `X-Hub-Signature-256` verify; `commit_sha` idempotency check before Hangfire enqueue.

---

## 7. Lesson Complete Saga (14 Step)

```
POST /api/v1/attempts/{id}/complete
  Idempotency-Key: <uuid>

[AttemptOrchestrator.CompleteAsync]
TX BEGIN
  1.  Idempotency check (lookup attempts.idempotency_key)
  2.  ScoringService → XP breakdown per answer + total
  3.  UserXp.total_xp += xp (atomic UPDATE)
  4.  xp_transactions INSERT (with multipliers, trace_id)
  5.  LevelService.Recompute → maybe LevelUpEvent
  6.  user_lesson_progress UPSERT (ON CONFLICT DO NOTHING for first complete)
  7.  user_track_progress.lessons_completed += 1 (atomic)
  8.  HeartsService apply losses (heart_lost from answers)
  9.  StreakService.RecordActivity (streak ilerlet, milestone detect)
  10. LeagueService.UpdateWeeklyXp (atomic)
  11. ActivityFeedService.Record (visibility-aware)
  12. BadgeRuleEngine.EvaluateSyncTier → user_badges INSERT
  13. CertificateEligibility.Evaluate → user_certificates INSERT (if eligible)
  14. Outbox INSERT:
        - LessonCompletedEvent
        - AttemptCompletedEvent
        - XpAwardedEvent
        - LevelUpEvent (conditional)
        - BadgeEarnedEvent[] (per new badge)
        - SkillvioCertificateIssuedEvent (conditional)
      attempts.status = 'completed'
      idempotency_response = JSON cache
TX COMMIT

[Async after commit]
OutboxDispatcher (LISTEN/NOTIFY + 5s polling fallback)
  → MassTransit publish to RabbitMQ
  → Consumers: Notification, BadgeWorker, SignalRBridge, AI (Sprint 5)
  → DLQ after 5 retries
```

**Concurrency strategy:**

| Resource | Strategy |
|----------|----------|
| `attempts.status` transition | Optimistic (`xmin`) — `WHERE status='in_progress'` predicate |
| `user_xp.total_xp` increment | `UPDATE user_xp SET total_xp = total_xp + @delta` (atomic) |
| `user_lesson_progress` first completion | `INSERT ... ON CONFLICT DO NOTHING RETURNING completed_at` |
| `user_track_progress` lessons_completed | Atomic UPDATE |
| `friendships` request | UNIQUE(requester_id, addressee_id) + symmetric block check |
| `league_memberships.weekly_xp` | Atomic UPDATE |
| Multi-tab attempt | `device_session_id`, ilk gelen kazanır, ikinci HTTP 409 |

---

## 8. Event Choreography

### 8.1 Publish (Learning → bus)

| Event | Aggregate | Trigger | Consume Eden |
|-------|-----------|---------|--------------|
| `LessonCompletedEvent` | Attempt | CompleteAsync | Notification, AI (Sprint 5), BadgeWorker |
| `AttemptCompletedEvent` | Attempt | CompleteAsync | Notification, Identity (cert hint) |
| `BadgeEarnedEvent` | UserBadge | RuleEngine + Worker | Notification, ActivityFeed, SignalR |
| `LevelUpEvent` | UserXp | LevelService | Notification, SignalR |
| `XpAwardedEvent` | UserXp | XpTransaction insert | LeagueService, SignalR |
| `StreakBrokenEvent` | DayStreak | NightlyEvaluator | Notification, BadgeWorker |
| `StreakMilestoneEvent` | DayStreak | 7/14/30/100/365 | Notification, BadgeWorker, ActivityFeed |
| `SkillvioCertificateIssuedEvent` | UserCertificate | EligibilityService | Identity, Notification |
| `LeagueAssignedEvent` | LeagueMembership | Hangfire weekly | Notification, SignalR |
| `LeagueEndedEvent` | LeagueSeason | Hangfire weekly | Notification, BadgeWorker, ActivityFeed |
| `LeaguePromotedEvent` | LeagueMembership | Finalization | Notification, BadgeWorker |
| `LeagueDemotedEvent` | LeagueMembership | Finalization | Notification |
| `FriendRequestSentEvent` | Friendship | Send | Notification, SignalR |
| `FriendRequestAcceptedEvent` | Friendship | Accept | Notification, ActivityFeed |
| `FriendBlockedEvent` | Friendship | Block | (internal) |
| `ExamAccessGrantedEvent` | ExamAccess | grant | Notification |
| `ExamAttemptStartedEvent` | Attempt | StartExam | (audit Loki) |
| `ExamAttemptCompletedEvent` | Attempt | CompleteExam | Notification, Identity |
| `ContentImportCompletedEvent` | ContentImport | Apply | BadgeWorker, Notification |
| `ComplianceAuditEvent` | (cross-cutting) | sensitive actions | Identity (compliance_audit) |

### 8.2 Consume (bus → Learning)

| Event | Publish Eden | Handler | Aksiyon |
|-------|--------------|---------|---------|
| `UserRegisteredEvent` | Identity | UserRegisteredHandler | hearts/streak/xp/prefs default rows |
| `UserSoftDeletedEvent` | Identity | UserSoftDeletedHandler | friendships clear, league withdraw |
| `UserHardDeletedEvent` | Identity | UserHardDeletedHandler | 22 tablo cascade + ComplianceAuditEvent |
| `UserPlanChangedEvent` | Identity | UserPlanChangedHandler | hearts unlimited toggle, exam_access grant |
| `UserProfileUpdatedEvent` | Identity | UserProfileUpdatedHandler | timezone update |
| `LabSubmittedEvent` | Lab | LabSubmittedHandler | XP + user_lab_completions + badge eval |
| `PaymentSucceededEvent` | Payment | PaymentSucceededHandler | exam_access insert |
| `PaymentRefundedEvent` | Payment | PaymentRefundedHandler | exam_access revoke |
| `CertificationVerifiedEvent` | Identity | CertificationVerifiedHandler | special badge |

### 8.3 Idempotency Pattern

```csharp
public async Task<bool> ConsumeAsync(TEvent evt, CancellationToken ct)
{
    using var tx = await _db.BeginTransactionAsync(ct);
    var inserted = await _db.ExecuteSqlInterpolatedAsync($@"
        INSERT INTO processed_events (event_id, consumer_name, processed_at)
        VALUES ({evt.EventId}, {ConsumerName}, NOW())
        ON CONFLICT DO NOTHING", ct);
    if (inserted == 0) { await tx.CommitAsync(ct); return true; }   // already processed
    await DoWorkAsync(evt, ct);
    await tx.CommitAsync(ct);
    return true;
}
```

`processed_events` 30-day retention via `processed_events.cleanup` Hangfire job.

### 8.4 Schema Versioning

Event'ler `skillvio-shared-contracts` NuGet'ında. Additive change → same version. Breaking change → V2 type, dual publish 30 gün, V1 deprecation log. Routing key: `learning.lesson.completed.v1`.

### 8.5 DLQ + Alerting

- MassTransit retry: 3 immediate + exponential backoff (1s, 5s, 30s, 2m, 10m) + DLQ
- DLQ depth > 0 → Grafana alert
- Queue depth > 1000 → page oncall
- Manual replay: `POST /api/v1/admin/learning/dlq/replay/{messageId}`

### 8.6 Tracing

OpenTelemetry W3C trace context propagation. `traceparent` MassTransit header'da. Tempo'da end-to-end span: `lesson_complete → 5 event publish → 5 consumer process` aynı trace_id.

### 8.7 Volume Estimate (100K MAU, 1 yıl)

| Event | Volume |
|-------|--------|
| `LessonCompletedEvent` | ~180M |
| `XpAwardedEvent` | ~360M |
| `BadgeEarnedEvent` | ~11M |
| `StreakMilestoneEvent` | ~1.8M |
| `LeagueEndedEvent` | ~5.2M |

`xp_transactions` ve `processed_events` partitioning zorunlu. RabbitMQ throughput hedef: 1000-2000 msg/s sustained. Sprint 7'de 3-5 node Quorum Queue cluster.

---

## 9. Hangfire Jobs

### 9.1 Recurring

| Job ID | Cron | Açıklama |
|--------|------|----------|
| `hearts.refill` | `*/15 * * * *` | < max users → +1 heart if 4h elapsed |
| `streak.evaluate.{tz}` | per timezone, 00:05 local | DISTINCT user_timezone'dan dynamic register |
| `streak.repair.warning` | `0 */4 * * *` | Streak break detected → 4h warning |
| `attempts.cleanup.abandoned` | `0 3 * * *` | > 24h idle in_progress → abandoned |
| `attempts.cleanup.expired_exam` | `*/30 * * * *` | Exam time_limit aşıldı → timed_out + scoring |
| `league.weekly.assignment` | `0 0 * * 1` | Monday 00:00 UTC: cohort assign |
| `league.weekly.finalization` | `59 23 * * 0` | Sunday 23:59 UTC: rank, promote/demote, events |
| `outbox.cleanup` | `0 4 * * *` | dispatched_at < 7 days → DELETE |
| `processed_events.cleanup` | `0 4 * * *` | < 30 days → DELETE |
| `idempotency.cleanup` | `0 4 * * *` | expires_at past → CLEAR |
| `partition.create.monthly` | `0 1 1 * *` | next month partition for xp_tx, attempt_answers, badge_eval |
| `partition.drop.old` | `0 2 1 * *` | > 12 month → drop |
| `badge.evaluation.async` | `*/5 * * * *` | Backfill async tier badges queue |
| `friend.request.expire` | `0 6 * * *` | > 30 days pending → auto-decline |

### 9.2 Outbox Dispatcher

**Not Hangfire** — `IHostedService` background service:
- Postgres `LISTEN outbox_new` for sub-100ms latency
- 5s polling fallback (NOTIFY missed)
- Exponential backoff retry, DLQ after 5 attempts

### 9.3 Ad-hoc (admin)

- `content.sync.run` — webhook + admin POST
- `badge.recompute.user` — single user re-eval (support tool)
- `badge.recompute.global` — single badge backfill (yeni rozet)
- `xp.recompute.user` — xp_transactions'tan total_xp recompute
- `cert.eligibility.recompute` — retroactive

### 9.4 Hangfire Dashboard

- Dev: `/_internal/hangfire` enabled
- Prod: `Hangfire__DashboardEnabled: false` default; açıksa **admin-role authorization filter** zorunlu
- Public-facing **asla** olmaz (RCE riski)

---

## 10. Deployment

### 10.1 Container Topology

| Container | Project | Replicas (prod) |
|-----------|---------|-----------------|
| `learning-api` | Skillvio.Learning.Api | 3 (HPA max 10) |
| `learning-worker` | Skillvio.Learning.Api (Hangfire host — Sprint 7'de ayrı project) | 2 (leader election) |
| `learning-content-sync` | Skillvio.Learning.ContentSync | 1 (single, leader-only) |
| `learning-badge-worker` | Skillvio.Learning.BadgeWorker | 2 (HPA queue depth) |
| `learning-postgres` | Postgres 16 | RDS multi-AZ |
| (shared) | Redis, RabbitMQ | infra |

### 10.2 docker-compose.dev.yml (excerpt)

```yaml
learning-postgres:
  image: postgres:16
  environment: { POSTGRES_DB: learning_db, POSTGRES_USER: learning, POSTGRES_PASSWORD: dev_password }
  ports: ["5435:5432"]

learning-api:
  build: { context: ../skillvio-learning-service, dockerfile: src/Skillvio.Learning.Api/Dockerfile }
  environment:
    ConnectionStrings__Default: ...
    ConnectionStrings__Redis: redis:6379
    RabbitMQ__Host: rabbitmq
    Identity__BaseUrl: http://identity-api:4001
    Hangfire__DashboardEnabled: "true"
  ports: ["4002:8080"]

learning-content-sync:
  build: { ..., dockerfile: src/Skillvio.Learning.ContentSync/Dockerfile }
  volumes: [ "../skillvio-content:/app/content:ro" ]
  environment: { ContentRepo__Mode: local }

learning-badge-worker:
  build: { ..., dockerfile: src/Skillvio.Learning.BadgeWorker/Dockerfile }
```

### 10.3 CI/CD (GitHub Actions)

`skillvio-learning-service/.github/workflows/ci.yml`:
- `build-test` — Postgres/Redis/RabbitMQ services, dotnet test, codecov
- `schema-validate` — `dotnet ef migrations bundle` + idempotent apply check
- `docker-build` — 3 image (api, contentsync, badgeworker), push to GHCR on main

Branch protection: main = PR + 1 review + CI green.

### 10.4 Configuration

```json
{
  "ConnectionStrings": { "Default": "...", "Redis": "..." },
  "RabbitMQ": { "Host": "...", "VHost": "/skillvio", "User": "...", "Password": "..." },
  "Identity": { "BaseUrl": "..." },
  "ContentRepo": {
    "Mode": "git",
    "GitUrl": "https://github.com/skillvio/skillvio-content.git",
    "GitRefDefault": "main",
    "LocalPath": "/var/skillvio/content",
    "WebhookSecret": "..."
  },
  "Cache": {
    "DefaultTtlSeconds": 300,
    "ContentTtlSeconds": 600,
    "GlossarySearchTtlSeconds": 60
  },
  "Scoring": {
    "BaseRulesVersion": "v1",
    "EnableAccuracyMultiplier": true,
    "EnableStreakBonus": true,
    "EnablePerfectBonus": true,
    "EnableHeartsBonus": false,
    "EnablePlanMultiplier": false
  },
  "Streak": {
    "AutoFreezeDefault": false,
    "RepairWindowHours": 48,
    "MilestoneDays": [7, 14, 30, 100, 365]
  },
  "Hearts": {
    "MaxFreeUser": 5,
    "RefillIntervalHours": 4,
    "ProUnlimited": true
  },
  "League": {
    "CohortSize": 30,
    "TimezoneEvaluation": "UTC"
  },
  "SignalR": { "RedisBackplane": true },
  "Hangfire": { "DashboardEnabled": false }
}
```

Secrets (Azure Key Vault / AWS Secrets Manager prod):
- DB password, Redis password, RabbitMQ password
- Cookies session signing key (gateway shared)
- JWT signing key (gateway shared)
- ContentRepo webhook secret (HMAC)

---

## 11. Observability

### 11.1 Metrics (Prometheus, `/metrics`)

- `learning_attempts_started_total{type}` (counter)
- `learning_attempts_completed_total{type, passed}` (counter)
- `learning_xp_awarded_total{source}` (counter, sum)
- `learning_badge_evaluations_total{tier, result}` (counter)
- `learning_outbox_dispatched_total{event_type}` (counter)
- `learning_outbox_undispatched_count` (gauge — scrape lag)
- `learning_streak_breaks_total` (counter)
- `learning_league_finalizations_total{tier}` (counter)
- `learning_active_attempts` (gauge)
- `learning_content_import_duration_seconds` (histogram)
- ASP.NET Core standard: `aspnetcore_request_duration_seconds`

### 11.2 Logs (Serilog → Loki)

JSON structured: `{ timestamp, level, message, user_id, attempt_id, trace_id, ... }`. Sensitive scrub: password, payment_method, jwt_token. Log levels: Information default, Debug for `Skillvio.Learning.Application.*`.

### 11.3 Traces (OpenTelemetry → Tempo)

Auto-instrumented: ASP.NET Core, EF Core, MassTransit, HttpClient, Redis. Custom spans: `ScoringService.Calculate`, `BadgeRuleEngine.Evaluate`, `ContentSyncOrchestrator.Apply`.

### 11.4 Performance Targets

| Endpoint | p50 | p95 | p99 |
|----------|-----|-----|-----|
| `GET /tracks` (cached) | 30ms | 100ms | 200ms |
| `GET /tracks/{id}` | 50ms | 150ms | 300ms |
| `GET /lessons/{id}` | 50ms | 150ms | 300ms |
| `POST /attempts` (start) | 80ms | 200ms | 400ms |
| `POST /attempts/{id}/answers` | 60ms | 150ms | 300ms |
| `POST /attempts/{id}/complete` (saga) | **150ms** | **400ms** | **800ms** |
| `GET /me/dashboard` | 100ms | 250ms | 500ms |
| `GET /learning/search` | 100ms | 300ms | 600ms |

SLA: 99.5% availability. RPO 5dk, RTO 1h. Sprint 7'de SLO doc.

**Eğer p99 > 800ms**: Sprint 7'de saga 2-faz split (commit core → async outbox dispatch).

---

## 12. Sprint Görev Listesi

### 12.1 Sprint 3a — Core (4 hafta, ~130h)

| # | Task | Saat |
|---|------|------|
| T1 | Solution skeleton (.NET 10, 6 proje, 3 test) | 4 |
| T2 | DbContext + Init migration (30 tablo, partition, view, tsvector) | 10 |
| T3 | Domain models + value objects (mutation-test ready) | 12 |
| T4 | Localization + FallbackResolver + LocaleFilter + HybridCache | 6 |
| T5 | ScoringService + LevelService + HeartsService + table-driven tests | 8 |
| T6 | StreakService + freeze + repair + dynamic timezone evaluator | 6 |
| T7a | Attempt state machine (Start/Resume/SubmitAnswer/Abandon) | 8 |
| T7b | Complete saga + IdempotencyFilter + concurrent tests | 6 |
| T8 | BadgeRuleEngine (sync) + BadgeWorker (async, ayrı project) | 8 |
| T9 | CertificateEligibilityService + verify endpoint | 4 |
| T10 | Outbox + LISTEN/NOTIFY dispatcher + 8 inbound consumer + DLQ | 10 |
| T11a | Public content endpoints (~20) + ETag | 5 |
| T11b | /me/* endpoints (~25) + onboarding + next-lesson | 5 |
| T12 | Attempts + exam flow endpoints + heartbeat + cleanup | 8 |
| T15 | ContentSync (Git→DB) + webhook + advisory lock + dry-run | 10 |
| T16 | Cloud cert catalog (30 cert seed: exam.yml + roadmap + cert def + schemas) | 15 |
| T18a | Admin endpoints + observability core (Prometheus + Loki + role auth) | 5 |
| **Total** |  | **130h ÷ 6h/day = 22 days = ~4 hafta** |

**Sprint 3a sonu demo-able:** kullanıcı track açar, lesson çözer, XP kazanır, rozet alır, exam attempt'i tamamlar, Skillvio cert kazanır. Engagement loop %80 çalışır.

### 12.2 Sprint 3b — Social/League/Real-time (1.5 hafta, ~27h)

| # | Task | Saat |
|---|------|------|
| T13 | Friends + activity feed + spam guard + privacy | 8 |
| T14 | Leagues (cohort assignment + finalization + endpoints + Hangfire) | 8 |
| T17 | SignalR hub + Redis backplane + event bridge | 6 |
| T18b | Performance smoke (k6) + Hangfire dashboard + OWASP review | 5 |
| **Total** |  | **27h ÷ 6h/day = 5 days = ~1.5 hafta** |

**Toplam: 5.5 hafta solo, 157 saat efor.**

---

## 13. Riskler

| # | Risk | Olasılık | Etki | Mitigation |
|---|------|----------|------|-----------|
| R1 | Lesson complete saga p99 > 800ms | Orta | Yüksek | Sprint 7'de 2-faz split; k6 smoke test T18b zorunlu |
| R2 | ContentSync race condition (multi-pod) | Düşük | Orta | `pg_advisory_lock` + webhook commit_sha idempotency |
| R3 | **Cloud exam soruları telif/lisans** | Yüksek | Çok yüksek | ❗ ExamTopics yeniden ifade, mod review, ADR 0015 |
| R4 | AWS/Azure/GCP exam version değişimi | Yüksek | Orta | `exams.expires_at` + `cert_code` versioning, eski attempt korunur |
| R5 | Hangfire Postgres dashboard scale | Düşük | Düşük | Sprint 7'de Hangfire Pro alternatif revisit |
| R6 | Outbox LISTEN/NOTIFY connection idle timeout | Orta | Orta | 5s polling fallback + keep-alive |
| R7 | **Sprint 2 WebSocket pass-through eksik** | Orta | Yüksek | ⚠️ Sprint 2 amendment Sprint 3a kickoff'tan önce |
| R8 | `skillvio-shared-contracts` v0.1 freeze | Orta | Yüksek | Sprint 3 kickoff'tan önce contract finalize |
| R9 | T16 cloud cert bootstrap 15h ucu açık | Orta | Düşük | Kritik path'te değil; rolling Sprint 4-7 |
| R10 | exam_questions BLOB performance | Düşük | Orta | Sprint 7'de partition revisit if > 50K row/exam |
| R11 | Friends federated query Identity'e bağlı | Orta | Düşük | 60s cache, rate limit 30/min |
| R12 | League cohort fairness | Düşük | Düşük | Fisher-Yates + Sprint 5 newcomer cohort |
| R13 | Streak DST geçişi | Düşük | Orta | TimeZoneInfo DST-aware + boundary tests |
| R14 | UserHardDeleted consume fail | Düşük | Yüksek | DLQ + manual replay + compliance_audit korunur |
| R15 | Cert verification public URL abuse | Düşük | Düşük | Rate limit 60/min/IP + 128-bit code entropy |

---

## 14. Cross-Sprint Amendment'lar

### 14.1 Sprint 1 (Identity) — geriye dönük

- [ ] `GET /users/search?q={username}&limit=20` endpoint
- [ ] `UserPlanChangedEvent` schema finalize
- [ ] `UserProfileUpdatedEvent` schema (timezone)
- [ ] `UserSoftDeletedEvent` + `UserHardDeletedEvent` 30-day finalizer Hangfire
- [ ] `compliance_audit` action types: `data_deleted_hard`, `certificate_revoked`, `exam_access_revoked`

### 14.2 Sprint 2 (Gateway) — amendment

- [ ] WebSocket pass-through: `/api/v1/ws/me` route + `WebSocketsEnabled: true` + cookie auth pre-flight
- [ ] Rate limit policies: `LessonSubmit`, `ExamStart`, `FriendRequest`, `ContentSearch`
- [ ] YARP `learning` cluster confirmed

### 14.3 Sprint 4 (Lab)

- [ ] Lab service `LabSubmittedEvent` publish (Sprint 4 zorunlu)
- [ ] Sprint 3'te Lab consumer **stub'lanır**, Sprint 4'te real event devreye girer

### 14.4 Sprint 5 (AI)

- [ ] AI service `/recommendations` endpoint (Sprint 3 stub'tan beslenir)
- [ ] `LessonCompletedEvent` AI consumer

### 14.5 Sprint 6 (Frontend)

- [ ] WebSocket connect (cookie auth)
- [ ] HybridCache + ETag client-side cache
- [ ] OpenAPI → TS contract auto-gen

### 14.6 Sprint 7 (Production)

- [ ] Hangfire/MassTransit consumer process'leri **API'den ayır**
- [ ] Saga 2-phase split revisit (eğer p99 > 800ms)
- [ ] Postgres partition retention policy (12 ay)
- [ ] RabbitMQ Quorum Queue migration
- [ ] Multi-region planning (`region_hint`)

---

## 15. Open Issues

Sprint 3 implementation'ından önce karara bağlanacak:

| OI# | Konu | Owner | Deadline |
|-----|------|-------|----------|
| OI1 | Stripe vs Iyzico exam payment provider (coğrafi split mi tek provider mı) | User | Sprint 3a kickoff |
| OI2 | Skillvio Cert verifiable URL (`app.skillvio.io/verify/{code}` vs subdomain) | User | T9 öncesi |
| OI3 | Friend leaderboard opt-in default (true vs false) | User | Sprint 3b kickoff |
| OI4 | **Soru telif modeli** ("inspired by ExamTopics" disclaimer vs 100% original) — legal review | User + legal | T16 öncesi (KRİTİK) |
| OI5 | League prize XP (Bronz +50, Şampiyon +500, exponential vs linear) | User | T14 |
| OI6 | Hearts max Free 5 / Pro unlimited mı yoksa Pro 10 mı | User | T5 |
| OI7 | Glossary search engine v1 Postgres tsvector → Sprint 5'te Meilisearch revisit | User | Sprint 5 başlangıç |
| OI8 | Skillvio cert artwork (designer / Figma / AI generated) | User | T9 öncesi |
| OI9 | Domain icon set (Lucide / Heroicons / custom) | User | Sprint 6 |
| OI10 | Mobile strategy (React Native vs PWA) | User | Sprint 7 |

---

## 16. ADR'ler (Sprint 3 sırasında yazılacak)

- `0011-content-as-code-git-source-of-truth.md` (Q1=B)
- `0012-i18n-hybrid-jsonb-translations-tables.md` (Q7=C)
- `0013-outbox-pattern-eventbus.md` (Q10=B)
- `0014-skillvio-certificates.md` (Q2 yeni)
- `0015-exam-content-licensing.md` (R3 + OI4)

---

## 17. Tamamlanma Kriterleri (DoD)

### Sprint 3a sonu

- [ ] 30 tablo migration applies cleanly
- [ ] Tüm content endpoint'leri çalışır, locale-aware
- [ ] BadgeService 30+ rozeti doğru evaluate eder (test)
- [ ] Streak gece doğru ilerler/sıfırlanır (TR + UTC + 4 timezone test)
- [ ] Hearts 4 saatte 1 yenilenir, Pro unlimited
- [ ] Lesson complete event publish ediliyor (outbox commit'te)
- [ ] Exam attempt flow timed, scored, cert eligible
- [ ] Skillvio cert verifiable
- [ ] ContentSync Git→DB tam çalışır (webhook + dry-run + advisory lock)
- [ ] 30 cert catalog seed Git'te
- [ ] %75+ test coverage (Domain %90+, Application %75+, Integration full happy paths)

### Sprint 3b sonu

- [ ] Friends lifecycle (send/accept/decline/block) çalışır
- [ ] Friend feed privacy-aware
- [ ] League weekly cohort + finalization Hangfire çalışır
- [ ] SignalR push (XP, badge, level, friend, league) frontend'e ulaşır
- [ ] k6 smoke: 100 concurrent attempt complete p99 < 800ms

---

## 18. Spec Self-Review Sonucu

**Placeholder scan:** Yok (TBD/TODO yok).

**Internal consistency:**
- Bölüm 2 schema vs Bölüm 8 event publishers → uyumlu
- Bölüm 5 endpoints vs Bölüm 6 services → uyumlu
- Q5 hearts vs Bölüm 7 saga step 8 → uyumlu
- Q10 outbox vs Bölüm 6 LISTEN/NOTIFY → uyumlu

**Scope check:** Sprint 3a + Sprint 3b bölünmesi tek implementation plan'a sığar (writing-plans iki ayrı plan üretebilir).

**Ambiguity check:** OI1-OI10 explicit Open Issue olarak işaretlendi (ambiguity yerine "decision needed" netliği).

---

## 19. Sonraki Adım

Spec onay sonrası **`writing-plans`** skill'i ile implementation plan üretilecek. Plan iki parça olarak üretilir:
- `2026-05-04-sprint-3a-learning-core-plan.md`
- `2026-05-04-sprint-3b-learning-social-plan.md`

Her plan task-by-task acceptance test, file paths, code snippet placeholders ile birlikte gelecek.
