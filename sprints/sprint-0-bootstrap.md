# Sprint 0 — Bootstrap

**Süre:** 3 gün
**Hedef:** İskeletleri kur, bütün repo'lar GitHub'da, sprint planları yayında.

## Çıktılar

- ✅ GitHub org `skillvio` oluşturuldu
- ✅ 11 repo iskelet GitHub'da
- ✅ `skillvio-planning` repo'sunda 8 sprint dokümanı + 10 ADR + vizyon
- ✅ Custom Claude agent'ları (`dotnet-architect`, `ef-core-specialist`)
- ✅ Domain alındı (skillvio.io veya skillvio.dev)
- ✅ `skillvio-infra` içinde `docker-compose.dev.yml` iskelet (postgres + redis + rabbitmq)
- ✅ Her servis repo'sunda boş .NET solution iskeleti

## Görevler

### S0.T1 — Org + repo'lar
- [ ] GitHub org `skillvio` (yapıldı)
- [ ] 11 repo iskelet local'de
- [ ] `gh repo create` ile push
- [ ] Branch protection rules (`main` korumalı, PR + 1 review zorunlu)

### S0.T2 — Planning repo dokümantasyonu
- [ ] `docs/vision.md`
- [ ] `docs/architecture.md`
- [ ] `docs/microservices-overview.md`
- [ ] `docs/i18n.md`
- [ ] `docs/lab-strategy.md`
- [ ] `docs/content-model.md`
- [ ] `docs/gamification.md`
- [ ] `docs/badges-by-domain.md`
- [ ] `docs/tech-stack.md`
- [ ] `docs/glossary.md`
- [ ] `meta/repos.md`
- [ ] `meta/conventions.md`
- [ ] `meta/definition-of-done.md`

### S0.T3 — ADR'ler
- [ ] 0001 multi-repo
- [ ] 0002 microservices
- [ ] 0003 .NET 8 + EF Core
- [ ] 0004 Postgres + Redis
- [ ] 0005 i18n translations table
- [ ] 0006 cookie-session auth
- [ ] 0007 RabbitMQ event bus
- [ ] 0008 React + Vite + TS
- [ ] 0009 YARP gateway
- [ ] 0010 lab sandbox strategy

### S0.T4 — Domain + DNS
- [ ] skillvio.io veya skillvio.dev satın al
- [ ] Cloudflare DNS yönetimi
- [ ] SSL (Let's Encrypt prod, mkcert dev)

### S0.T5 — Infra iskelet
- [ ] `skillvio-infra/docker-compose.dev.yml`
   - postgres (3 DB: identity, learning, lab, notification)
   - redis
   - rabbitmq
   - mailhog (dev SMTP)
- [ ] `Makefile` (`make up`, `make down`, `make logs`)

### S0.T6 — Backend service iskeletleri
Her backend repo için:
- [ ] `dotnet new sln`
- [ ] Clean architecture (4 proje: Api / Application / Domain / Infrastructure + Tests)
- [ ] `Dockerfile` multi-stage
- [ ] `.editorconfig`, `Directory.Build.props`
- [ ] `README.md` servis-spesifik

### S0.T7 — GitHub konfigürasyonu
- [ ] Issue template (bug, feature, sprint-task)
- [ ] PR template
- [ ] CODEOWNERS
- [ ] `.github/workflows/ci.yml` her repo için iskelet

### S0.T8 — Claude agent'ları
- [x] `dotnet-architect.md`
- [x] `ef-core-specialist.md`
- [ ] `microservices-architect.md`
- [ ] `react-typescript-pro.md`
- [ ] `i18n-specialist.md`
- [ ] `qa-engineer.md`
- [ ] `devops-engineer.md`

## Tamamlanma Kriteri (DoD)

- [ ] 11 repo GitHub'da, README ve .gitignore ile
- [ ] `skillvio-planning` repo'sunda en az 13 doküman
- [ ] Her servis repo'sunda boş .NET solution + Dockerfile + CI workflow
- [ ] `docker-compose.dev.yml` ile postgres+redis+rabbitmq ayağa kalkar
- [ ] `Sprint 1` issue'ları GitHub'da açıldı

## Süre Tahmini

| Görev | Süre |
|-------|------|
| S0.T1 (repo'lar) | 1 saat |
| S0.T2 (planning docs) | 4 saat |
| S0.T3 (ADR'ler) | 3 saat |
| S0.T4 (domain) | 30 dk |
| S0.T5 (infra) | 2 saat |
| S0.T6 (backend iskeletler) | 6 saat (7 repo × ~1 saat) |
| S0.T7 (GitHub config) | 1 saat |
| S0.T8 (agent'lar) | 2 saat |
| **TOPLAM** | **~20 saat (3 gün)** |
