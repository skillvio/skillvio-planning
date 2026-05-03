# ADR 0001 — Multi-repo (her servis ayrı repo)

- **Status:** Accepted
- **Date:** 2026-05-04

## Bağlam

Skillvio bir microservices sistemidir (10+ servis). Üç ana yaklaşım vardı:

1. **Monorepo** — tek `skillvio` repo, alt klasörler
2. **Multi-repo** — her servis ayrı repo
3. **Hibrit** — backend tek monorepo, frontend ayrı, infra ayrı

## Karar

**Multi-repo** — toplam 11 repo:
- 1 planning (public)
- 1 frontend (private)
- 1 infra (private)
- 1 content (private)
- 7 backend (private; gateway + 5 service + shared)

## Sonuçlar

### Pozitif
- Servis-bazlı bağımsız sürüm + deploy
- Repo bazlı erişim kontrolü (içerik yazarına sadece content)
- CI build hızlı (sadece değişen servisi build eder)
- Senior takım büyürken takımları repo'ya bağlamak doğal

### Negatif
- Cross-service refactor PR koordinasyonu zor
- Versiyon eşitleme overhead (`shared-contracts` paketi)
- 11 repo bakımı (CI workflow, branch protection, vs.)

## Hafifletme

- `skillvio-shared-contracts` NuGet package — semver ile sürümleniyor
- Cross-cutting değişiklikler için "umbrella issue" planning'de
- GitHub Actions reusable workflow ile CI duplikasyonu azaltılır

## Alternatifler

- **Monorepo + Nx/Turborepo:** karmaşık tooling, .NET dünyasında nadir
- **Hibrit:** orta yol ama "neyin monorepo'da neyin ayrı" kuralı belirsiz olur

## Referanslar
- [Microservices.io — How to organize](https://microservices.io/patterns/decomposition/decompose-by-business-capability.html)
- Spotify, Uber multi-repo örnekleri
