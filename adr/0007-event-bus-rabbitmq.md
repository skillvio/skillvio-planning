# ADR 0007 — Event Bus: RabbitMQ + MassTransit

- **Status:** Accepted

## Karar
**RabbitMQ** broker, **MassTransit** abstraction layer.

## Gerekçe vs Alternatifler

| Kriter | RabbitMQ | NATS | Kafka | Redis Streams |
|--------|----------|------|-------|----------------|
| Olgunluk | ✅ | ✅ | ✅ | 🟡 |
| .NET ekosistem | ✅ MassTransit | 🟡 | ✅ Confluent | 🟡 |
| Operational | 🟡 (daha karmaşık) | ✅ | ❌ Kafka cluster ağır | ✅ |
| Throughput | 🟡 | ✅ | ✅ | ✅ |
| Persistence | ✅ | 🟡 (JetStream) | ✅ | ✅ |
| Dead letter queue | ✅ | 🟡 | 🟡 | ❌ |

RabbitMQ + MassTransit en olgun .NET tarafında. Skillvio ölçeğinde Kafka aşırı.

## Event Schema

`skillvio-shared-contracts` NuGet package:
```csharp
public record UserRegisteredEvent(
    Guid UserId,
    string Username,
    string Email,
    string Locale,
    DateTime CreatedAt
);
```

Tüm servisler aynı paketi referans eder; semver ile sürümlenir.

## Reliability

- **Outbox pattern**: event publish DB transaction'ında
- **Idempotency**: consumer event ID'sine göre dedup
- **Dead letter queue**: 3 retry sonra DLQ
- **Saga**: long-running flows (örn. signup → email + onboarding)

## Konu Adlandırma

Topic: `skillvio.<bounded-context>.<event-name>` — örn:
- `skillvio.identity.user-registered`
- `skillvio.learning.lesson-completed`
- `skillvio.lab.lab-submitted`
