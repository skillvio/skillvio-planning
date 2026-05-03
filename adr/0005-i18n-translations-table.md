# ADR 0005 — i18n: Locale-per-Row (`_translations` table pattern)

- **Status:** Accepted

## Karar
Her translatable entity için sibling `_translations` tablosu.

```sql
tracks              (id, code, level, color, …)
track_translations  (track_id, locale, title, subtitle, description)
```

## Gerekçe
- Yeni dil eklemek migration gerektirmez (sadece insert)
- Boş içerik tablo şemasını şişirmez
- Locale fallback chain kolay (`tr` yok → `en` → default)
- EAV anti-pattern'inden kaçınır

## Alternatifler reddedildi
- ❌ **Locale kolonları** (`title_tr`, `title_en`) — yeni dil için migration
- ❌ **JSONB locale objesi** (`title: {tr: '...', en: '...'}`) — sorgu zor
- ❌ **Ayrı i18n servisi** — overkill, cross-service join sorunu

## Detay
Bkz. `docs/i18n.md`
