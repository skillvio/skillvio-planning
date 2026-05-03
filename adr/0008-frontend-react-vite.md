# ADR 0008 — Frontend: React + Vite + TypeScript

- **Status:** Accepted

## Karar
**React 18 + Vite + TypeScript** + ekosistem:
- **TanStack Query** (data fetching + cache)
- **Zustand** (minimal state)
- **React Router** (navigation)
- **i18next + react-i18next** (i18n)
- **NSwag-generated TS client** (backend tip-güvenli)
- **ShadCN/UI components** (mevcut tema CSS değişkenleriyle uyumlu)
- **Vitest + Playwright** (test)

## Gerekçe

### React vs Alternatifler
| Çerçeve | Avantaj | Dezavantaj |
|---------|---------|------------|
| **React** | TR developer havuzu büyük, ekosistem en geniş | Boilerplate |
| Vue | Daha temiz | Daha az job piyasası |
| Svelte | En az kod | Olgun değil, ekosistem küçük |
| SolidJS | Hızlı | Çok yeni |

### Vite vs Webpack/Next.js
- Vite: dev hot reload anlık, build hızlı
- Next.js: SSR gerekirse, ama Skillvio CSR yeter (SEO public sayfalar için sonradan eklenebilir)

## Mevcut Tema Korunması

Vanilla JS'den port'ta:
- `app/style.css` aynen `apps/web/src/styles/theme.css`
- CSS değişkenleri (--bg-0, --brand, …) aynen kalır
- Component'ler aynı `class`ları kullanır
- Görsel olarak değişiklik yok

## Build & Deploy

- **Dev:** `pnpm dev` (Vite localhost:5173)
- **Prod:** `pnpm build` → static dist → Cloudflare Pages / Vercel
- **API base URL:** env (`VITE_API_BASE`)
