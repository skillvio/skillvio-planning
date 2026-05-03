# Sprint 6 — Frontend React + TS Port

**Süre:** 2 hafta
**Hedef:** Vanilla → React + Vite + TypeScript. Mevcut tema %100 korunur.

## Çıktılar
- `skillvio-frontend` repo
- React + Vite + TypeScript
- i18next çoklu dil
- TanStack Query + Zustand
- NSwag generated TS API client
- Mevcut style.css aynen
- Playwright e2e

## Görevler

### S6.T1 — Vite + TS kurulum
```bash
cd skillvio-frontend
pnpm create vite@latest . --template react-ts
pnpm add @tanstack/react-query react-router-dom zustand i18next react-i18next
pnpm add -D @nswag/cli playwright vitest @testing-library/react
```

### S6.T2 — Mevcut style.css aktarımı
- `apps/web/src/styles/theme.css` ← `app/style.css` aynen
- Tüm CSS değişkenleri ve component class'ları korunur

### S6.T3 — Sayfa sayfa port
1. `<AppShell>` — sidebar + main layout
2. Auth sayfaları (login/signup/onboarding)
3. Discover (roadmap list)
4. Roadmap viewer
5. Lesson player (Duolingo akışı)
6. Lab player (HTML/CSS/JS/Python/SQL)
7. Sınav modu (study/sequential/exam)
8. Profile (4 tab)
9. Admin (4 tab)

### S6.T4 — i18n
- `react-i18next` setup
- `tr.json`, `en.json` (mevcut Türkçe metinler aktarılır)
- URL routing: `/tr/...`, `/en/...`
- `<LanguageSwitcher />` sidebar'da

### S6.T5 — API client (NSwag)
```bash
nswag run --runtime=Net80 ../skillvio-gateway/swagger.json
```
Çıktı: `src/api/client.ts` — tip-güvenli generated client

### S6.T6 — TanStack Query patterns
- Query keys: `['tracks', { domain }]`
- Mutation invalidation
- Optimistic updates (lesson complete)

### S6.T7 — Test
- Vitest + React Testing Library (component tests)
- Playwright e2e: signup → onboarding → lesson → lab

### S6.T8 — Build + deploy
- `vite build` artifact
- Cloudflare Pages / Vercel deploy
- Production env: `VITE_API_BASE` config

## Tamamlanma Kriteri
- [ ] Tüm sayfa rotaları çalışır
- [ ] Görsel olarak değişiklik yok (tema korunur)
- [ ] Türkçe + İngilizce switch
- [ ] E2E happy path geçer
- [ ] Bundle < 500KB gzipped
