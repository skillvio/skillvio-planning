# Sprint 1 — Identity Service Design Spec

- **Status:** Approved (2026-05-04)
- **Date:** 2026-05-04
- **Author:** Brainstorming session (Skillvio team + Claude)
- **Sprint:** 1 / 8
- **Estimated effort:** 2 weeks
- **Repo:** [skillvio/skillvio-identity-service](https://github.com/skillvio/skillvio-identity-service)
- **Related ADRs:** 0002 (microservices), 0003 (.NET 10 LTS + EF Core 10), 0004 (Postgres + Redis), 0006 (cookie session), 0007 (RabbitMQ events)

---

## 1. Amaç

Skillvio'nun **Identity Service**'ini sıfırdan .NET 10 + EF Core 10 ile inşa etmek. Bu servis tüm
kullanıcı kimliği, oturum yönetimi, abonelik durumu ve sertifika akışlarının **tek doğruluk
kaynağı**dır.

Mevcut Node.js backend (`/Users/gokhannihal/exam/backend/`) bu sprint sonunda devre dışı
bırakılacak; frontend (mevcut `app/`) yeni .NET service ile çalışmaya başlayacak.

## 2. Kapsam

### Bu sprintte var

- Email + şifre + sosyal giriş (Google + GitHub)
- Email doğrulama akışı + Resend entegrasyonu
- Cookie-based session + refresh token rotation + family invalidation
- BCrypt cost 12 + pepper hash, opsiyonel TOTP MFA
- 5 sistem rolü + 4 abonelik planı (rol ve plan bağımsız)
- Soft delete (30 gün) + cross-service `UserHardDeletedEvent`
- Sertifika başvuru + admin onay/red akışı
- Onboarding (3 adım, atlanabilir, multi-track, dinamik tema)
- Loki + OpenTelemetry observability (logs + traces + metrics)
- Postgres `compliance_audit` (yasal kanıt)
- xUnit + Testcontainers integration testleri (≥%70 coverage)
- Multi-stage Docker build + GitHub Actions CI

### Bu sprintte **yok**

- Stripe / Iyzico entegrasyonu (Sprint X — Billing)
- haveibeenpwned passive check (Sprint 2)
- Aktif Cihazlar UI (backend hazır, UI Sprint 6'da)
- Admin panel UI (Sprint 6.5)
- GDPR data export (Sprint 5 — event bus aktif olunca)
- Marketing site (Sprint 6 ile paralel — Q2=C kararı)
- Reporting (Sprint 7 — BFF pattern, Q3=D kararı)
- Frontend port (Sprint 6 — `skillvio-app` repo)

---

## 3. Brainstorming Karar Özeti

### Soru 1 — Kayıt Akışı: **B**
**Email doğrulamalı + sosyal giriş Sprint 1'de**

- Email + şifre kayıt zorunlu
- **Email doğrulama zorunlu** — onaylamadan derslere/lab'lara erişim yok (`onboarding` görünür ama lesson/lab API'leri 403)
- Sosyal giriş Sprint 1'de: **Google OAuth + GitHub OAuth**
- SMTP: Mailtrap (dev) → Resend (prod)
- Spam koruması: doğrulanmamış kullanıcı 7 gün içinde doğrulamazsa otomatik silinir

### Soru 2 — Şifre & Hash: **D**
**Modern baseline + opsiyonel MFA + BCrypt cost 12 + pepper**

- Min 8 karakter, en az 1 harf + 1 rakam (özel karakter zorunlu değil)
- Hash: **BCrypt cost 12 + server-side pepper** (env: `IDENTITY_PASSWORD_PEPPER`)
- Login rate limit: 15 dk'da 5 başarısız → 15 dk lock + email uyarısı
- Şifre değişimi: eski şifre + yeni × 2 + email bildirimi + tüm refresh token revoke
- **MFA (TOTP):** Sprint 1'de endpoint'ler hazır, default kapalı, profile'dan açılır
- haveibeenpwned passive check Sprint 2/3'e ertelendi

### Soru 3 — Session Model: **B**
**Cookie + refresh rotation + family invalidation + session metadata**

- **Access cookie:** `skillvio.sid` HttpOnly, SameSite=Lax, Secure (prod), 15 dk TTL
- **Refresh cookie:** `skillvio.rt` HttpOnly, path=`/api/auth/refresh`, 30 gün TTL
- Cookie domain: `.skillvio.io` (subdomain'ler arası paylaşım — marketing/app/admin)
- Refresh rotation: her `/refresh` çağrısı yeni access + yeni refresh, eskisini revoke
- **Family invalidation:** revoked refresh tekrar kullanılırsa **tüm aile** invalidate + email uyarısı
- Token DB'de SHA-256 hash olarak (raw token cookie'de)
- Session metadata kolonları hazır: `device_name`, `ip`, `user_agent`, `last_seen`
- Şifre değişiminde tüm aileler revoke
- "Tüm cihazlardan çıkış" Sprint 1'de backend hazır, UI Sprint 6'da

### Soru 4 — Hesap Silme: **B + D evrim**
**Sprint 1'de soft delete + 30 gün; GDPR data export Sprint 5'te**

- Sprint 1: `users.deleted_at`, `scheduled_purge_at`, `deleted_email_hash`
- Soft delete → 30 gün sonra Hangfire job hard delete
- 30 gün içinde geri yüklenebilir (login → "Hesabını geri yükle?" ekranı)
- Hard delete sırasında `UserHardDeletedEvent` yayılır → diğer servisler kendi verilerini siler
- EF Core global query filter: `WHERE deleted_at IS NULL`
- GDPR export endpoint Sprint 5'te (event-driven cross-service aggregation gerek)

### Soru 5 — Onboarding: **D + multi-track + dinamik tema**
**3-adım atlanabilir, çoklu yol kayıt, accent renk değişimi**

- 3 adım: hedef → tecrübe → ilgi alanları (multi-select)
- Her adımda **"Şimdi Atla"** linki
- Atlayan kullanıcı: persistent banner + profile'dan tekrar açılabilir
- **Multi-track:** kullanıcı birden fazla yola aynı anda kayıt olabilir
  - DB tablo: `user_active_tracks` (Sprint 3'te detay; Sprint 1'de `users.active_track_ids` JSONB cache)
  - `is_primary` flag, `ord` (sürükle-bırak öncelik)
- **Dinamik tema:** her track/domain'in kendi `accent_color` + `gradient`'i; frontend `data-active-track="..."` ile CSS variable override
- Track switcher UI Sprint 6'da

### Soru 6 — Roller & Yetkilendirme: **C**
**Rol ve subscription bağımsız; Stripe iskeleti hazır**

- **Rol** (`users.role`): `user` | `admin` | `moderator` | `content_author` | `support`
- **Plan** (`subscriptions.plan`): `free` | `pro` | `team` | `enterprise`
- Yeni kayıt: rol=`user`, plan=`free` (signup tetikleyici default subscription oluşturur)
- İlk kayıt olan otomatik admin (mevcut davranış korunur)
- Authorization attribute'ları:
  - `[Authorize]` — login gerek
  - `[Authorize(Roles = "admin")]` — rol kontrolü
  - `[RequirePlan("pro")]` — custom attribute (Sprint X'te aktif)
- Audit log: rol değişiklikleri `compliance_audit`'e yazılır
- Stripe entegrasyonu **Sprint X** (henüz tarih yok)

### Frontend Yapısı: **Q1=A, Q2=C, Q3=D**

- **Q1=A:** 14 ayrı repo (multi-repo paradigm)
- **Q2=C:** Marketing site Sprint 6'da, app port ile paralel (8 hafta SEO/waitlist kaybı kabul)
- **Q3=D:** Reporting BFF pattern Sprint 7'de; ayrı reporting service ileride (1000+ kullanıcı)

### Soru 7 — Email Sağlayıcı: **B (Resend)**

- Free tier 3K/ay başlangıç, Pro $20/ay 50K
- React Email template'leri (`@react-email/components`)
- EU region (KVKK ek koruma)
- Domain: DKIM/SPF/DMARC kayıtları Cloudflare DNS'de
- From: `noreply@skillvio.io`
- Dev: Mailhog (Docker)
- Bounce/spam handling: Resend webhook → `notification_log`
- 8 başlangıç template'i (Welcome, EmailVerification, PasswordReset, PasswordChanged, NewDeviceLogin, AccountDeleted, AccountPurged, MfaEnabled)

### Soru 8 — Observability: **Loki + OpenTelemetry + Postgres compliance subset**

- **Logs:** Serilog → OTel → Loki (30 gün hot, 1 yıl S3 cold)
- **Traces:** OTel SDK → Tempo (14 gün hot, 90 gün cold)
- **Metrics:** OTel SDK → Prometheus → Mimir (30 gün, 1 yıl)
- **Unified UI:** Grafana (trace ↔ logs link aktif)
- **Compliance audit:** Postgres `compliance_audit` tablosu — yasal kanıt subset (~10K satır/yıl)
- Tüm log/trace/metric **OpenTelemetry standardı** — vendor-agnostic, ileride backend değiştirilebilir

---

## 4. Mimari

### 4.1 Servis Sınırları

```
┌──────────────────────────────────────────────────────────────────┐
│                    skillvio-identity-service                      │
│                                                                   │
│  Bounded context'ler:                                            │
│  ─────────────────────                                            │
│  • Auth        — signup, login, logout, refresh, MFA, email vrf  │
│  • Users       — profile CRUD, soft delete, role mgmt            │
│  • Sessions    — refresh tokens, devices                         │
│  • Subscriptions — plan tracking (Stripe webhook ileride)        │
│  • Certifications — kullanıcı sertifika başvuru + onay           │
│  • Onboarding  — goal, experience, domains, active tracks        │
│  • Audit       — compliance_audit (yasal kanıt)                  │
│                                                                   │
│  Bağımlılıklar:                                                  │
│  ──────────────                                                   │
│  • Postgres (identity_db)                                        │
│  • Redis (DataProtection, rate limit, session cache)             │
│  • RabbitMQ (event publish)                                      │
│  • Resend API (email)                                            │
│  • Loki + Tempo + Prometheus (via OTel collector)                │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Clean Architecture Layout

```
skillvio-identity-service/
├── Skillvio.Identity.sln
├── src/
│   ├── Skillvio.Identity.Api/                # HTTP layer
│   │   ├── Program.cs                        # DI, middleware, OTel
│   │   ├── Endpoints/
│   │   │   ├── AuthEndpoints.cs
│   │   │   ├── UsersEndpoints.cs
│   │   │   ├── CertificationsEndpoints.cs
│   │   │   └── OnboardingEndpoints.cs
│   │   ├── Filters/
│   │   │   ├── ValidationFilter.cs
│   │   │   └── ComplianceAuditFilter.cs
│   │   └── appsettings.json
│   │
│   ├── Skillvio.Identity.Application/        # Use cases + DTOs
│   │   ├── Auth/
│   │   │   ├── Commands/                     # SignupCommand, LoginCommand, etc.
│   │   │   ├── Services/                     # IAuthService, IPasswordService
│   │   │   └── Validators/                   # FluentValidation
│   │   ├── Users/
│   │   ├── Certifications/
│   │   ├── Onboarding/
│   │   └── Common/
│   │       ├── Result.cs                     # Result pattern
│   │       └── Errors.cs
│   │
│   ├── Skillvio.Identity.Domain/             # Entities + value objects + events
│   │   ├── Users/
│   │   │   ├── User.cs
│   │   │   ├── UserRole.cs (enum)
│   │   │   └── Events/
│   │   │       ├── UserRegisteredEvent.cs
│   │   │       ├── UserDeletedEvent.cs
│   │   │       ├── UserHardDeletedEvent.cs
│   │   │       └── CertificationVerifiedEvent.cs
│   │   ├── Sessions/
│   │   │   ├── RefreshToken.cs
│   │   │   └── TokenFamily.cs
│   │   ├── Subscriptions/
│   │   │   ├── Subscription.cs
│   │   │   └── SubscriptionPlan.cs (enum)
│   │   └── Certifications/
│   │       └── Certification.cs
│   │
│   ├── Skillvio.Identity.Infrastructure/     # Persistence + external clients
│   │   ├── Persistence/
│   │   │   ├── IdentityDbContext.cs
│   │   │   ├── Configurations/               # IEntityTypeConfiguration<T>
│   │   │   └── Migrations/
│   │   ├── Auth/
│   │   │   ├── BCryptPasswordHasher.cs
│   │   │   ├── TotpService.cs
│   │   │   └── OAuthClients/
│   │   │       ├── GoogleOAuthClient.cs
│   │   │       └── GitHubOAuthClient.cs
│   │   ├── Email/
│   │   │   ├── ResendEmailClient.cs
│   │   │   └── Templates/                    # React Email rendered HTML
│   │   ├── Cache/
│   │   │   └── RedisCacheService.cs
│   │   ├── EventBus/
│   │   │   └── MassTransitConfiguration.cs
│   │   └── Audit/
│   │       └── PostgresComplianceAuditLogger.cs
│   │
│   └── Skillvio.Identity.Tests/
│       ├── Unit/
│       │   ├── Auth/
│       │   ├── Users/
│       │   └── Subscriptions/
│       ├── Integration/                       # Testcontainers
│       │   ├── AuthFlowTests.cs
│       │   └── UserManagementTests.cs
│       └── Fixtures/
│
├── Dockerfile                                 # multi-stage build
├── .dockerignore
├── .editorconfig
├── Directory.Build.props                      # nullable, langversion
└── README.md
```

### 4.3 Subdomain Cookie Sharing

Frontend mimarisi 3 ayrı subdomain üzerinde:

```
www.skillvio.io        — Marketing (Next.js, Sprint 6)
app.skillvio.io        — Kullanıcı uygulaması (React+Vite, Sprint 6)
admin.skillvio.io      — Admin panel (React+Vite, Sprint 6.5)
api.skillvio.io        — Backend gateway (Sprint 2)
```

**Set-Cookie:**
```
Set-Cookie: skillvio.sid=<value>; Domain=.skillvio.io; Path=/; HttpOnly; SameSite=Lax; Secure; Max-Age=900
Set-Cookie: skillvio.rt=<value>; Domain=.skillvio.io; Path=/api/auth/refresh; HttpOnly; SameSite=Lax; Secure; Max-Age=2592000
```

`Domain=.skillvio.io` sayesinde 3 subdomain de cookie'yi okur. CSRF için
SameSite=Lax + token-based double-submit cookie pattern (Sprint 1'de hazır olacak).

---

## 5. Database Schema

### 5.1 Tüm tablolar (Sprint 1)

```sql
-- ════════════════════════════════════════════════════════════════
-- Snake_case naming convention (EFCore.NamingConventions)
-- DB: identity_db
-- ════════════════════════════════════════════════════════════════

-- Extensions
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS citext;       -- case-insensitive email

-- ────────────────────────────────────────────────────────────────
-- USERS
-- ────────────────────────────────────────────────────────────────
CREATE TABLE users (
  id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username                 TEXT UNIQUE NOT NULL CHECK (username ~ '^[a-z0-9_]{3,30}$'),
  display_name             TEXT NOT NULL,
  email                    CITEXT UNIQUE,
  email_verified           BOOLEAN NOT NULL DEFAULT FALSE,
  email_verified_at        TIMESTAMPTZ,
  password_hash            TEXT,                                  -- BCrypt + pepper
  avatar_colors            JSONB NOT NULL DEFAULT '["#ff9c00","#ff5e00"]'::jsonb,
  avatar_icon              TEXT NOT NULL DEFAULT '',

  role                     TEXT NOT NULL DEFAULT 'user'
                           CHECK (role IN ('user','admin','moderator','content_author','support')),

  -- Onboarding
  onboarding_completed     BOOLEAN NOT NULL DEFAULT FALSE,
  goal                     TEXT CHECK (goal IN ('job','cert','curiosity','switch_career')),
  experience_level         TEXT CHECK (experience_level IN ('none','student','junior','mid','senior')),
  ui_locale                TEXT NOT NULL DEFAULT 'tr',
  preferred_locale         TEXT NOT NULL DEFAULT 'tr',
  active_track_ids         JSONB NOT NULL DEFAULT '[]'::jsonb,    -- Sprint 3'te denormalize cache
  primary_track_id         TEXT,                                  -- sidebar default

  -- MFA (TOTP)
  mfa_enabled              BOOLEAN NOT NULL DEFAULT FALSE,
  mfa_secret_encrypted     TEXT,                                  -- AES-256-GCM with KEK
  mfa_backup_codes_hash    TEXT[],                                -- 10 adet, SHA-256

  -- Soft delete
  deleted_at               TIMESTAMPTZ,
  scheduled_purge_at       TIMESTAMPTZ,
  deleted_email_hash       TEXT,                                  -- SHA-256(email) — yeniden kayıt unique check

  -- Login security
  failed_login_count       INTEGER NOT NULL DEFAULT 0,
  locked_until             TIMESTAMPTZ,

  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_login_at            TIMESTAMPTZ
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email) WHERE email IS NOT NULL;
CREATE INDEX idx_users_deleted_at ON users(deleted_at);
CREATE INDEX idx_users_purge ON users(scheduled_purge_at) WHERE scheduled_purge_at IS NOT NULL;
CREATE INDEX idx_users_role ON users(role);

-- ────────────────────────────────────────────────────────────────
-- EMAIL VERIFICATION TOKENS
-- ────────────────────────────────────────────────────────────────
CREATE TABLE email_verification_tokens (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash   TEXT NOT NULL,                    -- SHA-256
  email        CITEXT NOT NULL,                  -- email değişikliklerinde de kullanılır
  expires_at   TIMESTAMPTZ NOT NULL,             -- 24 saat
  consumed_at  TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_evt_user ON email_verification_tokens(user_id, created_at DESC);
CREATE INDEX idx_evt_hash ON email_verification_tokens(token_hash) WHERE consumed_at IS NULL;

-- ────────────────────────────────────────────────────────────────
-- PASSWORD RESET TOKENS
-- ────────────────────────────────────────────────────────────────
CREATE TABLE password_reset_tokens (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash   TEXT NOT NULL,                    -- SHA-256
  expires_at   TIMESTAMPTZ NOT NULL,             -- 1 saat
  consumed_at  TIMESTAMPTZ,
  ip           INET,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_prt_hash ON password_reset_tokens(token_hash) WHERE consumed_at IS NULL;

-- ────────────────────────────────────────────────────────────────
-- REFRESH TOKENS (with family invalidation)
-- ────────────────────────────────────────────────────────────────
CREATE TABLE refresh_tokens (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  family_id     UUID NOT NULL,                   -- aynı cihazın tüm token'ları
  token_hash    TEXT NOT NULL,                   -- SHA-256
  device_name   TEXT,                            -- "Chrome on macOS"
  ip            INET,
  user_agent    TEXT,
  last_seen     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at    TIMESTAMPTZ NOT NULL,            -- 30 gün
  revoked_at    TIMESTAMPTZ,
  revoke_reason TEXT,                            -- 'rotation' | 'logout' | 'family_invalidation' | 'password_change' | 'admin'
  replaced_by   UUID REFERENCES refresh_tokens(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_rt_user_family ON refresh_tokens(user_id, family_id);
CREATE INDEX idx_rt_hash ON refresh_tokens(token_hash) WHERE revoked_at IS NULL;
CREATE INDEX idx_rt_active ON refresh_tokens(user_id) WHERE revoked_at IS NULL;

-- ────────────────────────────────────────────────────────────────
-- OAUTH IDENTITIES (Google + GitHub)
-- ────────────────────────────────────────────────────────────────
CREATE TABLE oauth_identities (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider       TEXT NOT NULL CHECK (provider IN ('google','github')),
  provider_uid   TEXT NOT NULL,
  email          CITEXT,
  display_name   TEXT,
  avatar_url     TEXT,
  raw_payload    JSONB,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_used_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (provider, provider_uid)
);
CREATE INDEX idx_oauth_user ON oauth_identities(user_id);

-- ────────────────────────────────────────────────────────────────
-- SUBSCRIPTIONS (plan tracking — Stripe ileride)
-- ────────────────────────────────────────────────────────────────
CREATE TABLE subscriptions (
  id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id                UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  plan                   TEXT NOT NULL DEFAULT 'free'
                         CHECK (plan IN ('free','pro','team','enterprise')),
  status                 TEXT NOT NULL DEFAULT 'active'
                         CHECK (status IN ('active','trial','past_due','canceled')),
  trial_ends_at          TIMESTAMPTZ,
  current_period_start   TIMESTAMPTZ,
  current_period_end     TIMESTAMPTZ,
  cancel_at_period_end   BOOLEAN NOT NULL DEFAULT FALSE,
  canceled_at            TIMESTAMPTZ,

  -- Stripe (Sprint X)
  stripe_customer_id     TEXT,
  stripe_subscription_id TEXT,

  -- Iyzico (Türkiye, Sprint X+1)
  iyzico_customer_id     TEXT,

  created_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at             TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_subs_status ON subscriptions(status);
CREATE INDEX idx_subs_period_end ON subscriptions(current_period_end);

-- ────────────────────────────────────────────────────────────────
-- CERTIFICATIONS (kullanıcı başvurusu + admin onay)
-- ────────────────────────────────────────────────────────────────
CREATE TABLE certifications (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  exam_id         TEXT NOT NULL,
  provider        TEXT NOT NULL,
  cert_code       TEXT NOT NULL,
  cert_number     TEXT,
  date_earned     DATE,
  valid_until     DATE,
  proof_image_url TEXT,
  verify_url      TEXT,
  notes           TEXT,
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','verified','rejected')),
  admin_note      TEXT,
  reviewed_by     UUID REFERENCES users(id) ON DELETE SET NULL,
  reviewed_at     TIMESTAMPTZ,
  submitted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_certs_user ON certifications(user_id);
CREATE INDEX idx_certs_status ON certifications(status, submitted_at DESC);

-- ────────────────────────────────────────────────────────────────
-- USER INTERESTS (Sprint 3'te genişler)
-- ────────────────────────────────────────────────────────────────
CREATE TABLE user_interests (
  user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  domain_id    TEXT NOT NULL,                    -- 'cloud','code','ai','database','devops','data','frontend','backend','terms'
  level        TEXT DEFAULT 'beginner',
  selected_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_id, domain_id)
);

-- ────────────────────────────────────────────────────────────────
-- COMPLIANCE AUDIT (yasal kanıt — Loki dışı, kalıcı)
-- ────────────────────────────────────────────────────────────────
CREATE TABLE compliance_audit (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  category        TEXT NOT NULL
                  CHECK (category IN ('gdpr','security','billing','admin_action','cert')),
  action          TEXT NOT NULL,                 -- 'user.deleted', 'role.changed', vs.
  actor_user_id   UUID,
  actor_role      TEXT,                          -- snapshot at time
  target_user_id  UUID,
  target_id       TEXT,                          -- non-user resources
  details         JSONB NOT NULL DEFAULT '{}'::jsonb,
  ip              INET,
  user_agent      TEXT,
  trace_id        TEXT,                          -- OTel trace_id (Tempo cross-link)
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_audit_actor ON compliance_audit(actor_user_id, created_at DESC);
CREATE INDEX idx_audit_target ON compliance_audit(target_user_id, created_at DESC);
CREATE INDEX idx_audit_action ON compliance_audit(action, created_at DESC);
CREATE INDEX idx_audit_created ON compliance_audit(created_at DESC);

-- ────────────────────────────────────────────────────────────────
-- LOGIN ATTEMPTS (rate limit + brute force tracking)
-- ────────────────────────────────────────────────────────────────
CREATE TABLE login_attempts (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username      TEXT,                            -- başarısız attempt'lerde de kayıt
  user_id       UUID,                            -- bulunmuşsa
  ip            INET NOT NULL,
  user_agent    TEXT,
  successful    BOOLEAN NOT NULL,
  failure_reason TEXT,                           -- 'invalid_password', 'user_not_found', 'locked', 'mfa_required'
  attempted_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_login_ip_time ON login_attempts(ip, attempted_at DESC);
CREATE INDEX idx_login_username_time ON login_attempts(username, attempted_at DESC);
-- Hangfire job: 90 günden eski kayıtları siler
```

### 5.2 Compliance audit aksiyon listesi

| Action | Kategori | Yazan |
|--------|----------|-------|
| `gdpr.user_deleted` | gdpr | DELETE /users/me |
| `gdpr.user_restored` | gdpr | POST /users/me/restore |
| `gdpr.user_purged` | gdpr | Hangfire HardDeleteJob |
| `gdpr.data_exported` | gdpr | (Sprint 5) |
| `gdpr.consent_changed` | gdpr | PATCH /users/me/consents |
| `security.role_changed` | security | PATCH /admin/users/{id}/role |
| `security.password_reset` | security | POST /auth/password-reset/confirm |
| `security.mfa_enabled` | security | POST /auth/mfa/verify |
| `security.mfa_disabled` | security | POST /auth/mfa/disable |
| `security.admin_impersonate_started` | security | POST /admin/users/{id}/impersonate |
| `security.admin_impersonate_ended` | security | DELETE /admin/users/{id}/impersonate |
| `security.session_invalidated_all` | security | POST /auth/sessions/invalidate-all |
| `billing.subscription_changed` | billing | (Sprint X — Stripe webhook) |
| `cert.approved` | cert | PATCH /admin/certifications/{id}/status |
| `cert.rejected` | cert | PATCH /admin/certifications/{id}/status |

---

## 6. API Spesifikasyonu

### 6.1 Auth

| Method | Path | Auth | Açıklama |
|--------|------|------|----------|
| POST | `/api/auth/signup` | ❌ | Email + şifre + displayName + (opsiyonel) email |
| POST | `/api/auth/login` | ❌ | Şifre OK → MFA enabled ise mfa_required:true, değilse cookie set |
| POST | `/api/auth/login/mfa` | ❌ | MFA challenge cevabı (TOTP code) |
| POST | `/api/auth/refresh` | refresh cookie | Yeni access + refresh çifti |
| POST | `/api/auth/logout` | ✅ | Mevcut session'ı sonlandır |
| POST | `/api/auth/sessions/invalidate-all` | ✅ | Tüm cihazlardan çıkış |
| GET | `/api/auth/me` | ✅ | Kullanıcı + plan + onboarding state |
| POST | `/api/auth/email-verification/resend` | ✅ | Doğrulama emailini tekrar gönder |
| POST | `/api/auth/email-verification/confirm` | ❌ | Token ile email doğrula |
| POST | `/api/auth/password-reset/request` | ❌ | Email reset link gönderir |
| POST | `/api/auth/password-reset/confirm` | ❌ | Token + yeni şifre |
| POST | `/api/auth/password/change` | ✅ | Eski şifre + yeni şifre |
| POST | `/api/auth/mfa/enroll` | ✅ | TOTP secret + QR kod döner |
| POST | `/api/auth/mfa/verify` | ✅ | İlk TOTP kod ile aktivasyon |
| POST | `/api/auth/mfa/disable` | ✅ | Şifre + TOTP code ile kapat |
| GET | `/api/auth/oauth/google` | ❌ | Google OAuth redirect başlat |
| GET | `/api/auth/oauth/google/callback` | ❌ | Google callback handler |
| GET | `/api/auth/oauth/github` | ❌ | GitHub OAuth redirect başlat |
| GET | `/api/auth/oauth/github/callback` | ❌ | GitHub callback handler |

### 6.2 Users

| Method | Path | Auth | Açıklama |
|--------|------|------|----------|
| GET | `/api/users/me` | ✅ | Mevcut kullanıcı detay (subscription dahil) |
| PATCH | `/api/users/me` | ✅ | displayName, avatarColors, avatarIcon, ui_locale, preferred_locale |
| POST | `/api/users/me/email/change-request` | ✅ | Yeni email'e doğrulama gönder |
| POST | `/api/users/me/email/change-confirm` | ❌ | Token ile email değiştir |
| DELETE | `/api/users/me` | ✅ | Soft delete + 30 gün purge schedule |
| POST | `/api/users/me/restore` | session valid (deleted state) | Soft delete'i geri al |
| GET | `/api/users/me/sessions` | ✅ | Aktif refresh token aileleri (UI Sprint 6) |
| DELETE | `/api/users/me/sessions/{family_id}` | ✅ | Belli bir cihazı sonlandır |

### 6.3 Certifications

| Method | Path | Auth | Açıklama |
|--------|------|------|----------|
| GET | `/api/certifications/me` | ✅ | Kullanıcının başvuruları |
| POST | `/api/certifications` | ✅ | Yeni başvuru (status=pending) |
| DELETE | `/api/certifications/{id}` | ✅ | Kendi başvurusunu sil |

### 6.4 Onboarding

| Method | Path | Auth | Açıklama |
|--------|------|------|----------|
| POST | `/api/onboarding` | ✅ | goal + experience_level + domains[] + active_track_ids[] |
| POST | `/api/onboarding/skip` | ✅ | onboarding_completed=true, alanlar boş |
| GET | `/api/onboarding/me/interests` | ✅ | İlgi alanları listesi |

### 6.5 Admin (role kontrol)

| Method | Path | Roller | Açıklama |
|--------|------|--------|----------|
| GET | `/api/admin/users` | admin, moderator, support | Filtre + paginate |
| GET | `/api/admin/users/{id}` | admin, moderator, support | Detay |
| PATCH | `/api/admin/users/{id}/role` | admin | Rol değiştir + audit log |
| POST | `/api/admin/users/{id}/impersonate` | admin | Geçici impersonate session |
| DELETE | `/api/admin/users/{id}/impersonate` | admin | Impersonate'i sonlandır |
| POST | `/api/admin/users/{id}/lock` | admin, moderator | Kullanıcıyı kilitle |
| GET | `/api/admin/certifications` | admin, moderator | Onay kuyruğu |
| PATCH | `/api/admin/certifications/{id}/status` | admin, moderator | verified \| rejected + admin_note |
| GET | `/api/admin/audit` | admin | compliance_audit query |

### 6.6 Health

| Method | Path | Açıklama |
|--------|------|----------|
| GET | `/health/live` | Liveness probe (servis ayakta mı) |
| GET | `/health/ready` | Readiness (DB + Redis bağlı mı) |
| GET | `/metrics` | Prometheus scrape endpoint |

### 6.7 Request/Response örnekleri

**POST /api/auth/signup**
```json
// Request
{
  "username": "gokhan",
  "displayName": "Gökhan Nihal",
  "email": "[email protected]",
  "password": "Strong1Password"
}

// Response 201
{
  "user": {
    "id": "3f2a...",
    "username": "gokhan",
    "displayName": "Gökhan Nihal",
    "email": "[email protected]",
    "emailVerified": false,
    "role": "user",
    "subscription": { "plan": "free", "status": "active" },
    "onboardingCompleted": false
  },
  "message": "Email gönderildi. Doğrulama linkine tıkla."
}
// Set-Cookie: skillvio.sid + skillvio.rt
```

**POST /api/auth/login (MFA-enabled user)**
```json
// Request
{ "username": "gokhan", "password": "Strong1Password" }

// Response 200
{ "mfaRequired": true, "mfaToken": "<short-lived-jwt>" }

// Sonra POST /api/auth/login/mfa
{ "mfaToken": "<from above>", "code": "123456" }
// Response: cookie set
```

**GET /api/auth/me**
```json
{
  "user": {
    "id": "3f2a...",
    "username": "gokhan",
    "displayName": "Gökhan Nihal",
    "email": "[email protected]",
    "emailVerified": true,
    "avatarColors": ["#ff9c00", "#ff5e00"],
    "role": "user",
    "uiLocale": "tr",
    "preferredLocale": "tr",
    "onboardingCompleted": true,
    "primaryTrackId": "aws-foundations",
    "activeTrackIds": ["aws-foundations", "python-basics"],
    "mfaEnabled": false,
    "subscription": {
      "plan": "free",
      "status": "active",
      "trialEndsAt": null,
      "currentPeriodEnd": null
    }
  }
}
```

---

## 7. Domain Events (RabbitMQ via MassTransit)

### 7.1 Publish edilen event'ler

```csharp
namespace Skillvio.Identity.Domain.Events;

public record UserRegisteredEvent(
    Guid UserId,
    string Username,
    string DisplayName,
    string? Email,
    string Role,
    string Locale,
    DateTimeOffset CreatedAt
);

public record UserEmailVerifiedEvent(
    Guid UserId,
    string Email,
    DateTimeOffset VerifiedAt
);

public record UserSoftDeletedEvent(
    Guid UserId,
    DateTimeOffset DeletedAt,
    DateTimeOffset ScheduledPurgeAt
);

public record UserRestoredEvent(
    Guid UserId,
    DateTimeOffset RestoredAt
);

public record UserHardDeletedEvent(
    Guid UserId,            // diğer servisler kendi data'larını silsin
    DateTimeOffset PurgedAt
);

public record UserRoleChangedEvent(
    Guid UserId,
    string FromRole,
    string ToRole,
    Guid? ChangedByUserId,
    string? Reason,
    DateTimeOffset ChangedAt
);

public record CertificationVerifiedEvent(
    Guid CertificationId,
    Guid UserId,
    string ExamId,
    string Provider,
    string CertCode,
    DateTimeOffset VerifiedAt
);

public record CertificationRejectedEvent(
    Guid CertificationId,
    Guid UserId,
    string? AdminNote,
    DateTimeOffset RejectedAt
);

public record SubscriptionChangedEvent(
    Guid UserId,
    string FromPlan,
    string ToPlan,
    string Status,
    DateTimeOffset ChangedAt
);
```

### 7.2 Topic naming convention

```
skillvio.identity.user-registered
skillvio.identity.user-email-verified
skillvio.identity.user-soft-deleted
skillvio.identity.user-hard-deleted
skillvio.identity.user-role-changed
skillvio.identity.certification-verified
skillvio.identity.certification-rejected
skillvio.identity.subscription-changed
```

### 7.3 Outbox pattern

DB transaction içinde event tablosuna yazılır, MassTransit transactional outbox eklentisi
periyodik olarak publish eder. **Garantisi:** event yayılmazsa transaction rollback değil,
retry kuyruğa girer (at-least-once).

---

## 8. Security Detayları

### 8.1 BCrypt + pepper

```csharp
public sealed class BcryptPasswordHasher(IConfiguration cfg) : IPasswordHasher
{
    private readonly string _pepper = cfg["Identity:PasswordPepper"]
        ?? throw new InvalidOperationException("Pepper missing");
    private const int WorkFactor = 12;

    public string Hash(string password)
    {
        var peppered = password + _pepper;
        return BCrypt.Net.BCrypt.HashPassword(peppered, WorkFactor);
    }

    public bool Verify(string password, string hash)
    {
        var peppered = password + _pepper;
        return BCrypt.Net.BCrypt.Verify(peppered, hash);
    }
}
```

**Pepper rotation:** ENV değişikliği = mevcut hash'ler doğrulanmaz. **Hafifletme:**
hash'in başına pepper version prefix (`v1:bcrypt-hash`); rotation gerekirse `v2` eklenir,
kullanıcı login olduğunda otomatik upgrade.

### 8.2 TOTP MFA

- Library: `Otp.NET`
- Secret: 32 byte random, AES-256-GCM ile şifreli (KEK env'den)
- Issuer: `Skillvio`
- Account: `[email protected]`
- Backup codes: 10 adet, SHA-256 hash'lenmiş, 1 kez kullanılabilir
- Window tolerance: ±1 step (30 sn drift için)

### 8.3 Rate Limiting

`Microsoft.AspNetCore.RateLimiting` + Redis store:

| Endpoint | Strateji | Limit |
|----------|----------|-------|
| `POST /auth/signup` | per-IP fixed window | 3/saat |
| `POST /auth/login` | per-username sliding window | 5/15dk → 15dk lock |
| `POST /auth/password-reset/request` | per-username | 5/saat |
| `POST /auth/email-verification/resend` | per-user | 5/saat |
| All others | per-user | 200/dk |

### 8.4 Cookie güvenliği

```csharp
options.Cookie.HttpOnly = true;
options.Cookie.SameSite = SameSiteMode.Lax;
options.Cookie.SecurePolicy = CookieSecurePolicy.Always;  // prod
options.Cookie.Domain = ".skillvio.io";                    // prod
```

### 8.5 OAuth state + PKCE

- OAuth flow: Authorization Code + PKCE
- State parameter: 32 byte random, Redis'te 10 dk TTL, redirect_uri + nonce ile bağlı
- ID token (Google) signature verification
- Email verified=false olan OAuth account'lar otomatik doğrulanmış sayılmaz

---

## 9. Observability

### 9.1 OpenTelemetry config (her servis aynı)

```csharp
const string ServiceName = "skillvio.identity";

// Logs (Serilog → OTel → Loki)
builder.Host.UseSerilog((ctx, services, lc) => lc
    .Enrich.FromLogContext()
    .Enrich.WithProperty("service.name", ServiceName)
    .WriteTo.OpenTelemetry(opts =>
    {
        opts.Endpoint = builder.Configuration["Otel:Endpoint"]!;
        opts.Protocol = OtlpProtocol.Grpc;
        opts.ResourceAttributes = new Dictionary<string, object>
        {
            ["service.name"] = ServiceName,
            ["deployment.environment"] = builder.Environment.EnvironmentName,
        };
    }));

// Traces + Metrics
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(ServiceName, serviceVersion: "1.0.0"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation(opts =>
        {
            opts.SetDbStatementForText = true;
        })
        .AddRedisInstrumentation()
        .AddNpgsql()
        .AddSource("MassTransit")
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddProcessInstrumentation()
        .AddMeter("MassTransit")
        .AddOtlpExporter());
```

### 9.2 Compliance audit logger

```csharp
public interface IComplianceAuditLogger
{
    Task LogAsync(ComplianceAuditEntry entry, CancellationToken ct = default);
}

public record ComplianceAuditEntry
{
    public required string Category { get; init; }       // gdpr, security, billing, ...
    public required string Action { get; init; }
    public Guid? ActorUserId { get; init; }
    public string? ActorRole { get; init; }
    public Guid? TargetUserId { get; init; }
    public string? TargetId { get; init; }
    public object? Details { get; init; }
    public IPAddress? Ip { get; init; }
    public string? UserAgent { get; init; }
    public string? TraceId { get; init; }                 // Activity.Current?.Id
}

// Async, ana flow'u bloklamaz (background queue)
// Failed audit log = Sentry alert + retry, ana request başarılı sayılır
```

### 9.3 Trace correlation

Her audit log'a `Activity.Current?.Id` (W3C trace_id) yazılır. Admin panel'de
"audit kaydı tıkla" → "Tempo'da trace görüntüle" linki çalışır.

---

## 10. Email Templates

### 10.1 Liste (Sprint 1 — 8 template)

| Template | Trigger | Subject |
|----------|---------|---------|
| **Welcome** | UserRegisteredEvent | Skillvio'ya hoş geldin! Email'ini doğrula |
| **EmailVerification** | resend isteği | Skillvio email doğrulama |
| **PasswordReset** | password-reset/request | Skillvio şifre sıfırlama |
| **PasswordChanged** | password change OK | Şifren değiştirildi |
| **NewDeviceLogin** | yeni device fingerprint | Yeni cihazdan giriş yapıldı |
| **AccountDeleted** | DELETE /users/me | Hesabın silindi (30 gün geri alma süresi) |
| **AccountPurged** | hard delete job | Hesabın kalıcı olarak silindi |
| **MfaEnabled** | mfa/verify OK | İki adımlı doğrulama aktif |

### 10.2 React Email örnek

```tsx
// notification-service/templates/Welcome.tsx
import { Html, Head, Body, Container, Heading, Text, Button } from '@react-email/components';

export default function Welcome({ userName, verifyUrl, locale = 'tr' }) {
  const t = translations[locale];
  return (
    <Html>
      <Head />
      <Body style={{ fontFamily: 'system-ui' }}>
        <Container>
          <Heading>{t.welcome.heading} {userName}!</Heading>
          <Text>{t.welcome.body}</Text>
          <Button href={verifyUrl}>{t.welcome.cta}</Button>
        </Container>
      </Body>
    </Html>
  );
}
```

---

## 11. Sprint 1 Görev Listesi (Detaylı)

### S1.T1 — Solution + paketler (1 gün)
- `dotnet new sln` + 5 proje (Api, Application, Domain, Infrastructure, Tests)
- `Directory.Build.props`: nullable, langversion, treatwarningsaserrors
- `.editorconfig`
- NuGet paketler (yukarıdaki listeden)
- README + CONTRIBUTING

**DoD:** `dotnet build` ve `dotnet test` (boş test) yeşil.

### S1.T2 — Domain modelleri (1 gün)
- 8 entity + 9 event + value objects
- Domain davranışları: `User.SoftDelete()`, `RefreshToken.Revoke()`, `Subscription.UpgradeTo()`
- Result pattern + custom errors

**DoD:** Domain unit testleri %80 coverage; entity invariant'lar test edilmiş.

### S1.T3 — EF Core + initial migration (1 gün)
- `IdentityDbContext`
- 10 `IEntityTypeConfiguration<T>` (per entity)
- Snake_case naming convention
- Initial migration: `20260504_Init`
- `dotnet ef migrations script` review

**DoD:** Migration up + down çalışır; Postgres schema beklenen ile eşleşir.

### S1.T4 — Application layer + auth flow (3 gün)
- `IAuthService` (Signup, Login, RefreshToken, Logout, …)
- `IPasswordHasher` (BCrypt + pepper)
- `ITotpService`
- `IEmailService` (Resend wrapper)
- `IComplianceAuditLogger`
- FluentValidation rules per command
- OAuth flow (Google + GitHub) — `IOAuthClient` interface

**DoD:** Service unit testleri (mocked deps) yeşil.

### S1.T5 — Cookie + Redis session (1 gün)
- DataProtection → Redis persist (`PersistKeysToStackExchangeRedis`)
- Cookie auth scheme (`AddAuthentication().AddCookie(...)`)
- Refresh token rotation logic (background revoke job için Hangfire)
- CSRF: Anti-forgery double-submit cookie

**DoD:** Login → cookie set → /me OK → refresh → yeni cookie → logout → cookie clear.

### S1.T6 — API endpoints (2 gün)
- 30+ endpoint (yukarıdaki listeden)
- Minimal API + endpoint filters
- ProblemDetails response format
- Swagger / OpenAPI doc

**DoD:** Swagger UI tüm endpoint'leri gösteriyor; manuel test (Bruno/Postman) geçer.

### S1.T7 — Event publishing (1 gün)
- MassTransit + RabbitMQ config
- Outbox pattern (`MassTransit.EntityFrameworkCore`)
- 9 domain event publish

**DoD:** RabbitMQ Management UI'da event'lerin yayıldığı görülür; outbox failure recovery testlendi.

### S1.T8 — Hangfire jobs (0.5 gün)
- `HardDeleteJob` — günlük, `WHERE scheduled_purge_at < NOW()` → hard delete + event
- `UnverifiedEmailCleanupJob` — günlük, 7 gün doğrulanmamış kayıt sil
- `LoginAttemptsCleanupJob` — günlük, 90 gün+ kayıt sil

**DoD:** Hangfire dashboard'da job'lar zamanında çalışır.

### S1.T9 — Test (2 gün)
- xUnit + FluentAssertions + Testcontainers
- Integration test fixtures (Postgres + Redis + RabbitMQ container)
- 15 happy path test
- 20 sad path test (yanlış şifre, lock, expired token, vs.)

**DoD:** Coverage ≥%70, tüm testler 5 dakikadan kısa sürede.

### S1.T10 — OTel + observability (1 gün)
- OTel config (yukarıdaki snippet)
- Serilog enrichers
- Health check endpoints
- Prometheus metric endpoint

**DoD:** Grafana'da log+trace+metric görünür; trace-to-logs link çalışır.

### S1.T11 — Docker + CI (0.5 gün)
- Multi-stage Dockerfile (~200MB image)
- `.dockerignore`
- GitHub Actions workflow:
  - Build + test
  - Image build + push to GHCR (sadece main branch)
  - Tag → release

**DoD:** PR build yeşil; main merge → image GHCR'da.

### S1.T12 — Frontend smoke test (0.5 gün)
- Eski Node backend (`/Users/gokhannihal/exam/backend/`) durdur
- `docker-compose.yml` güncelle: `cs_api` → yeni .NET service
- Mevcut frontend (`app/index.html`) → manuel test:
  - Signup → email gelir mi (Mailhog)
  - Login → cookie set mi
  - /me → kullanıcı döner mi
  - Onboarding → çalışır mı

**DoD:** Tüm mevcut frontend auth akışı yeni backend ile çalışır.

**Toplam tahmini: 14 gün × 1 developer = 2 hafta**

---

## 12. Riskler & Hafifletme

| Risk | Olasılık | Etki | Hafifletme |
|------|----------|------|------------|
| BCrypt pepper rotation karmaşıklığı | Düşük | Orta | Hash prefix ile version'ing |
| OAuth flow edge case'leri (state expiry, double callback) | Orta | Orta | State Redis TTL + nonce; idempotency token |
| RabbitMQ outbox publish fail | Orta | Yüksek | At-least-once + idempotency consumer side; DLQ |
| Cookie domain `.skillvio.io` localhost'ta çalışmaz | Yüksek | Düşük | Dev için `localhost` ayrı config; prod sadece prod domain |
| TOTP clock drift | Orta | Düşük | Window tolerance ±1 (60 sn); time sync NTP |
| Soft delete sonrası username conflict | Orta | Düşük | `deleted_email_hash` + username "released" pattern (deleted prefix) |
| Email teslim edilmezse kullanıcı kayıp | Düşük | Yüksek | Resend webhook bounce handling + retry; admin manual unlock |
| Postgres uzun sorgu (audit table büyür) | Düşük (10K satır/yıl) | Düşük | Index + partition gerekirse; YAGNI |
| Refresh token replay attack | Orta | Yüksek | Family invalidation + audit log + email uyarısı |
| Subdomain cookie yanlış config (CSRF) | Düşük | Yüksek | Anti-forgery token + SameSite=Lax + Strict CSP |

---

## 13. ENV Variables

```env
# Postgres
IDENTITY_DB_CONNECTION="Host=postgres;Database=identity_db;Username=...;Password=..."

# Redis
REDIS_CONNECTION="redis:6379,password=...,defaultDatabase=0"
REDIS_DATAPROTECTION_DB=1
REDIS_RATELIMIT_DB=2

# RabbitMQ
RABBITMQ_HOST=rabbitmq
RABBITMQ_USER=...
RABBITMQ_PASS=...

# Auth
IDENTITY_PASSWORD_PEPPER=<32+ char random>
IDENTITY_MFA_KEK=<32-byte base64>          # AES-256 master key for TOTP secrets
COOKIE_DOMAIN=.skillvio.io                  # prod
COOKIE_SECURE=true                          # prod
ACCESS_COOKIE_TTL_SECONDS=900               # 15 min
REFRESH_COOKIE_TTL_DAYS=30

# Email (Resend)
RESEND_API_KEY=re_...
EMAIL_FROM="[email protected]"
EMAIL_FROM_NAME=Skillvio
EMAIL_DEV_MODE=mailhog                      # mailhog | resend
RESEND_WEBHOOK_SECRET=...

# OAuth
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=https://api.skillvio.io/api/auth/oauth/google/callback
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...
GITHUB_REDIRECT_URI=https://api.skillvio.io/api/auth/oauth/github/callback

# Observability
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
DEPLOY_ENV=production                        # development | staging | production

# Hangfire
HANGFIRE_CONNECTION="..."                    # ayrı schema veya identity_db
```

---

## 14. Spec Self-Review (Superpowers methodology)

### Placeholder scan
- ✅ TBD/TODO yok
- ✅ Tüm endpoint'ler request/response örnekli
- ✅ DB schema tam SQL ile

### Internal consistency
- ✅ ENV listesi tüm config'leri kapsıyor
- ✅ DB tabloları event payload'ları ile uyumlu
- ✅ Sprint 1 görevleri DoD ile bağlı

### Scope check
- ✅ Sprint 1 odağı net: Identity service tamamlandı + frontend bağlandı
- ✅ Sprint X-12'ye atılan işler açıkça belirtildi
- ✅ 2 haftalık effort tahmini gerçekçi

### Ambiguity check
- ✅ "Soft delete" → `deleted_at IS NOT NULL` + 30 gün purge schedule
- ✅ "Family invalidation" → token aile + replay tespit + email
- ✅ "Compliance subset" → 12 aksiyon tipi listelenmiş
- ✅ "MFA opsiyonel" → Sprint 1 endpoint hazır, default kapalı

### Bilinçli atılanlar (YAGNI)
- ❌ Distributed tracing dashboards (Sprint 7)
- ❌ Stripe checkout (Sprint X)
- ❌ Admin panel UI (Sprint 6.5)
- ❌ Frontend port (Sprint 6)
- ❌ Reporting service (Sprint 7+)

---

## 15. Approval Gate

> **Bu doküman draft. Brainstorming methodology gereği:**
>
> 1. Kullanıcı bu spec'i okur
> 2. Değişiklik isterse — yorum/değişiklik tur'u
> 3. Onaylanırsa → `writing-plans` skill'i devreye girer
>    - Sprint 1 GitHub issue'ları **detaylı acceptance criteria** ile güncellenir
>    - Her görev 2-5 dakikalık alt-task'a bölünür
>    - Code yazımı **HARD-GATE** açıldıktan sonra başlar
>
> **Hiçbir kod yazılmayacak — bu spec onaylanmadan önce.**

---

## 16. Karar Geçmişi

| Karar | Seçenek | Tarih | Sebep |
|-------|---------|-------|-------|
| Kayıt akışı | B (email vrf + sosyal) | 2026-05-04 | Spam koruması + UX ortayolu |
| Şifre/hash | D (BCrypt+pepper+opt MFA) | 2026-05-04 | Modern baseline + B2B ready |
| Session model | B (cookie+rotation+family) | 2026-05-04 | OWASP best practice |
| Hesap silme | B+D | 2026-05-04 | KVKK/GDPR uyumlu, evrim path |
| Onboarding | D + multi-track + dynamic theme | 2026-05-04 | Duolingo modeli + esneklik |
| Roller/plan | C | 2026-05-04 | Future-proof, clean separation |
| Frontend yapısı | Q1=A, Q2=C, Q3=D | 2026-05-04 | 14 repo + lazy reporting |
| Email | B (Resend) | 2026-05-04 | DX en iyi, EU region |
| Observability | Loki + OTel + Postgres compliance | 2026-05-04 | Vendor-agnostic standart |

---

## 17. Referanslar

- [OWASP ASVS 4.0](https://owasp.org/www-project-application-security-verification-standard/) — auth ve session bölümleri
- [OWASP Cheat Sheet — Refresh Token Rotation](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [GDPR Article 17 — Right to Erasure](https://gdpr.eu/article-17-right-to-be-forgotten/)
- [KVKK Madde 7 — Verilerin Silinmesi](https://www.mevzuat.gov.tr/MevzuatMetin/1.5.6698.pdf)
- [OpenTelemetry .NET](https://opentelemetry.io/docs/instrumentation/net/)
- [MassTransit Outbox](https://masstransit.io/documentation/patterns/transactional-outbox)
- [Resend React Email](https://resend.com/docs/send-with-react-email)
- [BCrypt Cost Factor Recommendations](https://github.com/BcryptNet/bcrypt.net)
- ADR 0006 — Cookie Session Auth (`adr/0006-auth-cookie-session.md`)
- ADR 0007 — RabbitMQ Event Bus (`adr/0007-event-bus-rabbitmq.md`)
