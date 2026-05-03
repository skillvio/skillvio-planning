# ADR 0006 — Cookie + Redis Session Auth

- **Status:** Accepted

## Karar
**HttpOnly cookie + Redis session store** (JWT yerine).

Cookie: `forge.sid`, HttpOnly, SameSite=Lax, Secure (prod).
Session TTL: 30 gün rolling.

## Gerekçe vs JWT
- ✅ Revoke kolay (Redis'ten sil)
- ✅ XSS'e karşı HttpOnly güvenli
- ✅ Token boyut sorunu yok (cookie'de session id, gerisi DB'de)
- ❌ CSRF dikkat edilmeli (SameSite=Lax + token)
- ❌ Multi-domain SSO için ek iş

## Refresh Token
- Access token (cookie session) 30 gün rolling
- Refresh token rotation (her kullanım yenisini üret) — Sprint 1'de planlanıyor

## Cross-service auth
- Gateway cookie'yi doğrular
- Downstream servislere `X-User-Id`, `X-User-Role`, `X-Request-Id` header'ları
- Servisler bu header'lara güvenir (Gateway dışından gelene kapalı)

## Password Hash
- **BCrypt-net-next** (saf C# implementasyonu, cost factor 12)
- İleride Argon2id evaluasyonu (bcrypt-net-next yetersiz olursa)
