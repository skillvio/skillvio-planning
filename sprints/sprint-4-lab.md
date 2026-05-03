# Sprint 4 — Lab Service

**Süre:** 2 hafta
**Hedef:** Editör tabanlı lab altyapısı; HTML/CSS/JS in-browser, Python (Pyodide), SQL (sql.js).

## Çıktılar
- `skillvio-lab-service` çalışır
- Frontend lab oynatıcı (mevcut `onboarding.js` üzerine)
- 3 dil için sandbox (HTML/CSS/JS, Python, SQL)
- Submission tracking + güvenlik

## Görevler

### S4.T1 — Solution + DB schema
- `labs` (id, kind, language, difficulty, sandbox_image, time_limit_sec, xp_reward)
- `lab_translations` (lab_id, locale, title, description, hints[])
- `lab_test_cases` (id, lab_id, ord, input, expected_output, weight)
- `lab_submissions` (id, user_id, lab_id, code, status, score, runtime_ms)
- `sandbox_sessions` (faz 2 için ileride)

### S4.T2 — Frontend lab oynatıcı
- HTML lab → iframe srcdoc (mevcut)
- CSS lab → iframe + sample HTML inject (mevcut)
- JS lab → sandboxed Function() + console capture (mevcut)
- **Yeni:** Python — Pyodide (3.11+ WASM)
- **Yeni:** SQL — sql.js (SQLite WASM)

### S4.T3 — Submission API
- `GET /api/labs/{id}` (lab + progress)
- `POST /api/labs/{id}/submit` (code + passed)
- `GET /api/labs/{id}/submissions` (kendi history)

### S4.T4 — Test case execution
- Python: stdin/stdout match
- SQL: query result diff (expected JSON ↔ actual)
- HTML/CSS: visual snapshot (faz 2)

### S4.T5 — Güvenlik
- Code size limit (50KB)
- Time limit (10s frontend execution)
- Rate limit (saatte 100 submit per-user)
- `Content-Security-Policy` iframe için strict

### S4.T6 — Faz 2 hazırlığı (Bash lab)
- Docker.DotNet entegrasyonu (lib reference)
- xterm.js component
- WebSocket bridge (ileride implementasyon)

### S4.T7 — Event consume
- Consume: `UserDeletedEvent` → lab data temizliği
- Publish: `LabSubmittedEvent` (learning service XP versin)

## Tamamlanma Kriteri
- [ ] HTML/CSS/JS labları çalışır
- [ ] En az 5 Python lab Pyodide ile çalışır
- [ ] En az 5 SQL lab sql.js ile çalışır
- [ ] Submission persist + retrieve
- [ ] Frontend lab oynatıcı responsive
