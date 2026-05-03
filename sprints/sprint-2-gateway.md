# Sprint 2 — API Gateway

**Süre:** 2 hafta
**Hedef:** YARP tabanlı `skillvio-gateway` çalışıyor, frontend gateway üzerinden identity'e ulaşıyor.

## Çıktılar
- YARP API Gateway tek host
- Cookie auth merkezi yerden geçer
- Rate limit + correlation id middleware
- Health-aware routing (servis kapalıysa 503)

## Görevler

### S2.T1 — Solution + YARP setup
```bash
dotnet new webapi -n Skillvio.Gateway
dotnet add package Yarp.ReverseProxy
```
`appsettings.json`'da:
```json
"ReverseProxy": {
  "Routes": {
    "auth": { "ClusterId": "identity", "Match": { "Path": "/api/auth/{**rest}" } },
    "users": { "ClusterId": "identity", "Match": { "Path": "/api/users/{**rest}" } },
    "certs": { "ClusterId": "identity", "Match": { "Path": "/api/certifications/{**rest}" } },
    "learn": { "ClusterId": "learning", "Match": { "Path": "/api/learning/{**rest}" } },
    "labs":  { "ClusterId": "lab",      "Match": { "Path": "/api/labs/{**rest}" } },
    "ai":    { "ClusterId": "ai",       "Match": { "Path": "/api/ai/{**rest}" } }
  },
  "Clusters": {
    "identity": { "Destinations": { "d1": { "Address": "http://identity:4001" } } },
    "learning": { "Destinations": { "d1": { "Address": "http://learning:4002" } } },
    "lab":      { "Destinations": { "d1": { "Address": "http://lab:4003" } } },
    "ai":       { "Destinations": { "d1": { "Address": "http://ai:4004" } } }
  }
}
```

### S2.T2 — Cross-cutting middleware
- `RequestId` (W3C trace context propagation)
- `RateLimitPolicy` (per-user fixed window, 200 req/min)
- `Serilog` request logging
- `Cors` config (frontend origin)
- Cookie auth validation; downstream'e `X-User-Id` header

### S2.T3 — Health-aware routing
- Active health check (15s interval)
- Servis 3 kez 5xx döndüyse circuit open
- Fallback response: 503 with retry-after

### S2.T4 — Frontend güncelleme
- `apiClient.js` `getBase()` artık gateway URL'i (`https://api.skillvio.io` veya `/api`)
- Cookie credentials zaten doğru
- Tüm path'ler aynı kalır

### S2.T5 — Eski Node backend'i devre dışı
- Eski `cs_api` container kapatılır
- `docker-compose.yml` güncellenir
- Smoke test: tüm UI path'leri

## Tamamlanma Kriteri
- [ ] Frontend tüm auth + profile path'lerinde çalışır
- [ ] Identity service kapalıyken gateway 503 döner
- [ ] Rate limit aktif (test edildi)
- [ ] Trace context cross-service akıyor
- [ ] CI yeşil, image GHCR'da
