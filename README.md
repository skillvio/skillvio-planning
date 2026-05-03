# Skillvio — Planning

> Vizyon, sprint planları, ADR'ler ve mimari dokümantasyonu.

**Skillvio** — Tech alanlarında çoklu dil destekli, oyunlaştırılmış öğrenme platformu.
Roadmap'lerle yol gösteren, Duolingo tarzı ünitelerle pratik yaptıran, TryHackMe tarzı
laboratuvarlarla elini kirleten bir platform.

## 📁 İçerik

```
skillvio-planning/
├── docs/
│   ├── vision.md                   ← Ürün vizyonu, kullanıcı kişaları
│   ├── architecture.md             ← Yüksek seviye mimari (microservices)
│   ├── microservices-overview.md   ← Servis sorumlulukları + iletişim
│   ├── i18n.md                     ← Çoklu dil stratejisi
│   ├── lab-strategy.md             ← Lab altyapısı (Pyodide → Docker → Cloud)
│   ├── content-model.md            ← Track / Unit / Lesson / Roadmap modeli
│   ├── gamification.md             ← XP, level, rozet, day streak, hearts
│   ├── badges-by-domain.md         ← Domain bazlı rozet matrisi
│   ├── tech-stack.md               ← Stack kararları + alternatifler
│   ├── runbook.md                  ← Operasyonel runbook
│   └── glossary.md                 ← Proje terim sözlüğü
├── adr/                            ← Architecture Decision Records
│   ├── 0001-multi-repo.md
│   ├── 0002-microservices.md
│   ├── 0003-dotnet-10-ef-core.md
│   ├── 0004-postgres-redis.md
│   ├── 0005-i18n-translations-table.md
│   ├── 0006-auth-cookie-session.md
│   ├── 0007-event-bus-rabbitmq.md
│   ├── 0008-frontend-react-vite.md
│   ├── 0009-yarp-gateway.md
│   └── 0010-lab-sandbox-strategy.md
├── sprints/                        ← Detaylı sprint planları
│   ├── sprint-0-bootstrap.md
│   ├── sprint-1-identity.md
│   ├── sprint-2-gateway.md
│   ├── sprint-3-learning.md
│   ├── sprint-4-lab.md
│   ├── sprint-5-ai-events.md
│   ├── sprint-6-frontend-react.md
│   ├── sprint-7-production.md
│   └── sprint-8-i18n-en.md
└── meta/
    ├── repos.md                    ← 11 repo'nun amaçları
    ├── conventions.md              ← Kod ve commit kuralları
    └── definition-of-done.md       ← Bir görevin tamamlanma kriterleri
```

## 🗂 Repo Haritası

| Repo | Amacı |
|------|-------|
| **skillvio-planning** *(burası)* | Vizyon + sprint + ADR |
| **skillvio-frontend** | React + Vite + TS web istemcisi |
| **skillvio-infra** | Docker, K8s, Terraform, CI/CD |
| **skillvio-content** | Roadmap/lesson/lab JSON içerikleri |
| **skillvio-gateway** | YARP API Gateway |
| **skillvio-identity-service** | Auth, users, certifications |
| **skillvio-learning-service** | Tracks, lessons, roadmaps, exams |
| **skillvio-lab-service** | Editor labs, sandbox |
| **skillvio-ai-service** | Gemini orchestration |
| **skillvio-notification-service** | Email, push |
| **skillvio-shared-contracts** | Paylaşılan DTO'lar / events |

## 🎯 Aktif Sprint

**Sprint 0 — Bootstrap (3 gün)** → `sprints/sprint-0-bootstrap.md`

## 📜 Lisans

MIT
