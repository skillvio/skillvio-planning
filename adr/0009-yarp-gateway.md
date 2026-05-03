# ADR 0009 — API Gateway: YARP

- **Status:** Accepted

## Karar
**YARP** (Yet Another Reverse Proxy) — Microsoft tarafından .NET için yazılmış gateway.

## Gerekçe vs Alternatifler

| Kriter | YARP | Nginx | Traefik | Kong |
|--------|------|-------|---------|------|
| .NET native | ✅ | ❌ | ❌ | ❌ |
| Hot reload config | ✅ | 🟡 | ✅ | ✅ |
| Auth middleware | ✅ ASP.NET | 🟡 module | 🟡 plugin | ✅ |
| Performance | ✅ | ✅ | ✅ | 🟡 |
| Operational | ✅ | ✅ | ✅ | 🟡 (Postgres/Cassandra) |

YARP avantajı: .NET ASP.NET Core middleware pipeline'a tam entegre — auth, rate limit,
custom logging vs. her şey C# ile.

## Sorumlulukları
- Routing (`/api/auth/...` → identity service)
- Cookie auth validation
- Rate limit (per-user)
- Request ID propagation (W3C trace context)
- Request logging
- CORS
- Health-aware routing (servis kapalıysa 503)

## Sınırları
- ❌ Business logic yok — sadece reverse proxy
- ❌ Direkt DB query yapmaz
- ❌ Authentication kararı veriyor mu? **Hayır**, sadece doğruluyor (giriş identity'de)
