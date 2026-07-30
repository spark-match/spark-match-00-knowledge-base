# Investigación: Arquitectura del Bounded Context Notifications (2026)

> **Status**: 🟢 Completa
> **Fecha inicio**: 2026-07-30
> **Fecha fin**: 2026-07-30
> **Investigadores**: @Angel

> **Documento canónico de decisión**: [ADR-017 en `spark-match-03-backend/docs/adr/`](https://github.com/spark-match/spark-match-03-backend/blob/dev/docs/adr/017-notifications-architecture.md). Este doc es el resumen cross-team de la investigación.

---

## Pregunta / Hipótesis

**Pregunta**: ¿Cómo debe implementar spark-match un sistema de notificaciones transaccionales (email + WhatsApp + SMS) en 2026, alineado con la arquitectura serverless + choreography + DLQ + idempotencia ya existente en `spark-match-03-backend` (ADRs-005, 006), resiliente a fallos y preparado para los volúmenes del mercado peruano (WhatsApp-primary)?

**Hipótesis**: Una arquitectura basada en **EventBridge → SQS FIFO → Lambda worker idempotente → channel adapters (SES/EUM Social/EUM Notify) → SNS callbacks → delivery tracker**, con templates en S3 versioned y per-user preferences, cumple los criterios sin introducir DynamoDB ni orquestador nuevo, y soporta los canales primarios de Perú (email + WhatsApp) en MVP con SMS como Fase 4.

## Criterios de éxito

- [x] Desacople total entre BCs productores y canales (ADR-006)
- [x] Idempotencia garantiza cero duplicados (crítico para no ser baneado por Meta WhatsApp)
- [x] Fail-closed: ningún error se pierde silenciosamente (DLQ + CloudWatch alarm)
- [x] Latencia E2E aceptable para transaccional (<15s p95)
- [x] Templates editables sin redeploy (S3 versioned)
- [x] Per-user preferences + i18n listos (Post-MVP)
- [x] Channel-agnostic: añadir push/SMS/in-app = 1 nuevo adapter, sin tocar productores
- [x] Cost control: budget alarm, SES sandbox exit temprano, EUM Basic tier para dev

## Alternativas evaluadas

| Alternativa | Pros | Contras |
|---|---|---|
| **A. In-handler sync (SES en el handler de Identity)** | Latencia cero, sin infra adicional | Acoplado, sin retry, sin DLQ, sin multi-canal, productor paga el costo |
| **B. Notifications BC + SQS FIFO + DLQ + idempotencia por notification_id** | Desacople, retry, multi-canal, observabilidad dedicada, channel-agnostic | +1 bounded context, latency +5-15s, onboarding WhatsApp Meta (1-2 sem) |
| **C. SaaS externo (Knock, OneSignal, Customer.io)** | Features avanzadas out-of-box | Vendor lock-in, costo 5-10x, datos fuera de control |
| **D. Lambda Powertools Idempotency + DynamoDB state** | Menos código de idempotencia | DynamoDB nuevo (no tenemos), costo extra, PostgreSQL ya está operativo |

**Recomendación**: **B**, en 4 fases incrementales (Fase 1: email MVP; Fase 2: WhatsApp; Fase 3: dashboard; Fase 4: SMS post carrier registration).

## Metodología

1. **Investigación documental** contra blogs oficiales de AWS jul-2026:
   - "Getting started with AWS End User Messaging Notify" (28-jul-2026): confirma Notify para SMS/OTP transaccional, Perú fully managed.
   - "Adding LINE Messenger to your omnichannel fallback solution" (5-jun-2026): arquitectura de referencia oficial WhatsApp + SMS + email + fallback automático.
   - "Build an AI-powered WhatsApp assistant" (25-jun-2026): patrón de integración WhatsApp + AI agents.
   - "Isolate email suppression per tenant" (7-jul-2026): feature nueva no aplicable a MVP (single tenant).
   - "Introducing Amazon SES pricing plans" (21-jul-2026): bundles 22% más baratos.
2. **Cross-check** contra ADRs existentes (005, 006, 011) para consistencia arquitectónica.
3. **Decisión** basada en:
   - Alineación con patrones ya establecidos (choreography + DLQ + idempotencia).
   - Minimizar infra nueva (reusar PostgreSQL existente vs introducir DynamoDB).
   - Mercado target Perú (WhatsApp-primary, SMS carrier registration cost).
   - Time-to-market MVP (email es inmediato, WhatsApp requiere Meta onboarding).

## Resultados

### Métricas cuantitativas (estimadas Fase 1 MVP)

| Métrica | Valor esperado | Fuente |
|---|---|---|
| Latencia E2E p95 | 5-15s | SQS standard visibility timeout + Lambda cold start ~3s |
| Costo email/1000 | $0.10 USD | Amazon SES pricing plan (jul-2026) |
| Costo WhatsApp utility/conv | ~$0.025 USD | Meta + EUM Social pricing |
| Costo SMS Perú/1000 (Advanced tier) | ~$0.40-0.80 USD | EUM Notify pricing + carrier rates |
| Throughput max Fase 1 | ~25 TPS | SQS + Lambda concurrent (default 1000) |
| Cold start Lambda | ~3s | Node 24, no VPC (excepto DB access via VPC peering) |

### Observaciones cualitativas

- **SES sandbox exit** = ~24h (form de producción). **Acción: hacerlo antes del primer release a prod.**
- **WhatsApp Business Meta verification** = 1-2 semanas. **Acción: arrancar el proceso en paralelo desde día 1 de Fase 2**, no esperar a codear.
- **SMS carrier registration Perú** (Movistar/Claro/Entel) = 4-6 semanas. **Acción: si Fase 4 está en roadmap, arrancar YA.**
- **WhatsApp requiere opt-in** del usuario. Sin opt-in, Meta banea la cuenta. Flag `user.whatsapp_opted_in` futuro.
- **Bedrock AI orchestrator** (channel selection por ML) es nice-to-have pero el costo vs uplift en engagement no justifica MVP.
- **Peru-specific**: WhatsApp penetration >90% en smartphones. Email universal pero baja tasa de apertura en jóvenes. SMS caro y requiere opt-in por normativa Osiptel.

### Failure modes cubiertos (matriz completa en ADR-017)

| Falla | Mitigación |
|---|---|
| Canal temporalmente caído | Retry con exp backoff + jitter (max 3) |
| Usuario deshabilitó canal | Fallback al siguiente preferido (email → WhatsApp → SMS) |
| Hard bounce / complaint | SES suppression list + flag `user.email_invalid` |
| WhatsApp rate limit | EUM Social maneja rate limits; circuit breaker si >N fallos/5min |
| Worker crash mid-flight | SQS visibility timeout (60s) → re-delivery automático |
| Evento duplicado | UNIQUE constraint en `(notification_id, channel)` → INSERT ON CONFLICT DO NOTHING |
| Provider quota excedido | CloudWatch alarm + AWS Budget alert |
| Template render falla | DLQ + alert; fail-closed (no bypass, no skip) |
| EventBridge down | Productor buffering local en PostgreSQL outbox (futuro, no MVP) |

## Conclusiones

**B** es la elección correcta. Justificación completa en [ADR-017 § Decisión](https://github.com/spark-match/spark-match-03-backend/blob/dev/docs/adr/017-notifications-architecture.md#decisi%C3%B3n).

**Plan de implementación** (resumen, completo en ADR-017 § Fases):

| Fase | Scope | Sprints |
|---|---|---|
| 1 | Email + SQS FIFO + DLQ + idempotencia + templates S3 | 2 |
| 2 | WhatsApp + channel router + delivery tracking | 1-2 |
| 3 | Observability dashboard + CloudWatch alarms | 0.5 |
| 4 | SMS (post carrier registration) + fallback automático | 1 |
| 5 | AI channel orchestrator (Bedrock) | 2 |

**Acciones previas a codear Fase 1:**
1. Solicitar SES production access (form, ~24h SLA).
2. Crear S3 bucket `spark-match-notifications-templates` con versioning.
3. Crear EventBridge rules (`user.*`, `assessment.*`) fan-out a SQS.

**Acciones previas a codear Fase 2:**
1. WhatsApp Business verification con Meta (1-2 sem). No empezar hasta approved.

**Acciones previas a codear Fase 4:**
1. SMS carrier registration con operadores Perú (4-6 sem). Empezar YA si Fase 4 está en roadmap.

## Referencias

- [ADR-017 canónico en `spark-match-03-backend`](https://github.com/spark-match/spark-match-03-backend/blob/dev/docs/adr/017-notifications-architecture.md) — detalle arquitectónico completo.
- [AWS Messaging Blog jul-2026: Notify](https://aws.amazon.com/blogs/messaging-and-targeting/getting-started-with-aws-end-user-messaging-notify/)
- [AWS Messaging Blog jun-2026: Omnichannel fallback con LINE](https://aws.amazon.com/blogs/messaging-and-targeting/adding-line-messenger-to-your-aws-omnichannel-fallback-solution/)
- [AWS Messaging Blog jun-2026: WhatsApp + Strands Agents](https://aws.amazon.com/blogs/messaging-and-targeting/build-an-ai-powered-real-estate-assistant-on-whatsapp-using-strands-agents-sdk-and-aws-end-user-messaging/)
- [ADR-005 EventBridge como bus principal](../../spark-match-03-backend/docs/adr/005-eventbridge-as-primary-event-bus.md)
- [ADR-006 Coreografía + DLQ + idempotencia](../../spark-match-03-backend/docs/adr/006-choreography-dlq-idempotency.md)
- [ADR-011 Idempotencia por eventId](../../spark-match-03-backend/docs/adr/011-idempotency-by-event-id.md)

## Anexos

- Investigación cruda: 4 blogs AWS leídos + cross-check con 3 ADRs existentes.
- Diagrama de arquitectura: en ADR-017 § Arquitectura recomendada.
- Matriz de failure modes completa: en ADR-017 § Failure modes cubiertos.