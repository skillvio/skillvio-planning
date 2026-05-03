# Sprint 8 — i18n EN İçerik

**Süre:** 2 hafta
**Hedef:** İlk EN dil rollout. Admin paneli AI taslak çevirisi yapabilir.

## Görevler

### S8.T1 — Frontend tam EN
- Tüm `tr.json` keys için `en.json` doldur
- Tarih/saat/sayı `Intl` API
- `<LanguageSwitcher />` aktif

### S8.T2 — İçerik çeviri pipeline
- Admin panelden track/lesson seç → "Translate to EN" butonu
- AI service Gemini batch çağrısı
- `_translations(status='draft')` olarak yazılır
- Admin manual onay → `status='published'`

### S8.T3 — İlk EN içerikler
- AWS DVA-C02 sınav soruları (zaten İngilizce kaynak)
- Python Basics track tam EN
- AI/ML Glossary tam EN
- IT İş İngilizcesi tam EN

### S8.T4 — URL routing
- `/tr/discover`, `/en/discover` her sayfada
- Cookie + browser tercihi otomatik dil seçimi
- `<link rel="alternate" hreflang>` etiketleri

### S8.T5 — SEO
- `sitemap.xml` her dil için
- Open Graph tags
- Schema.org Course markup

### S8.T6 — Edge cases
- AWS service isimleri çevirilmez (skip list)
- Code samples raw kalır
- RTL desteği hazır (Arapça için sonra)

## Tamamlanma Kriteri
- [ ] EN UI tam, eksik string yok
- [ ] 4 track EN'de yayında
- [ ] Admin "Translate" akışı end-to-end
- [ ] SEO tools (Lighthouse SEO score 100)
