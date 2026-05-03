# Sprint 7 — Production Hardening

**Süre:** 2 hafta
**Hedef:** Observability, backup, CI/CD, production-ready deploy.

## Görevler

### S7.T1 — OpenTelemetry distributed tracing
- Her servis OTel SDK
- W3C Trace Context propagation gateway → service
- Jaeger/Tempo collector

### S7.T2 — Metrics (Prometheus + Grafana)
- `/metrics` her servis
- Standart metrikler: request rate, latency, error rate
- Custom: lesson complete count, lab submission rate
- Grafana dashboard'lar

### S7.T3 — Logging (Loki / Seq)
- Serilog → Loki collector
- Structured logging her yerde
- Query: "Error in identity service in last 5 min"

### S7.T4 — Sentry
- Frontend + backend
- Source map upload (frontend)
- Alerting rules

### S7.T5 — Backup
- Postgres `pg_dump` cron → S3/B2
- 7 gün rolling, 4 hafta haftalık, 12 ay aylık
- Disaster recovery runbook

### S7.T6 — CI/CD per service
- GitHub Actions her repo:
  - PR: test + lint + build
  - main merge: image build → GHCR push
  - tag: prod deploy
- Helm chart (skillvio-infra/helm)

### S7.T7 — Production deploy
- Kubernetes cluster (DigitalOcean / Hetzner / AWS EKS)
- cert-manager + Let's Encrypt
- Cloudflare DNS + CDN
- Secrets: Sealed Secrets veya External Secrets Operator

### S7.T8 — Performance
- EF query log analizi (N+1 yakala)
- Bundle size budget
- Lighthouse score > 90

## Tamamlanma Kriteri
- [ ] Production cluster çalışıyor
- [ ] Tüm servisler OTel + Prometheus
- [ ] Grafana dashboard'lar canlı
- [ ] Sentry alerting tetiklenebilir
- [ ] Backup test edildi (restore başarılı)
- [ ] CI/CD tag → deploy < 10 dk
