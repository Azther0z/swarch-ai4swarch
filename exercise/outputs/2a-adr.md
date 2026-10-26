Agent: Codex (GPT-6)

# ADR-0002: Decouple post-order actions with RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Online Boutique architecture team (proposed)

## Context and problem

The R-12 scenario requires order confirmation through MailFast, loyalty points from a new \`loyaltyservice\`, and an “order placed” marketing analytics event. Marketing expects SMS, fraud scoring, and a recommendations update next year. Today, \`checkoutservice.PlaceOrder\` calls payment, shipping, and email synchronously over gRPC in sequence; email failure is logged as a warning and does not fail the order. The stated current p95 is about 300 ms.

The new design must keep \`PlaceOrder\` p95 at or below 500 ms, handle a peak of 200 orders/s, and ensure an order does not fail because email, loyalty, or analytics is unavailable. Loyalty must be awarded within one minute and exactly once. MailFast has p95 1.5 s, p99 8 s, about 0.5% call failures, and a 50 emails/s rate limit. The team has four developers, strong Go/gRPC skills, and two developers with RabbitMQ experience; operations prefers open-source self-hosted components and already uses Kubernetes.

**Scenario evidence:** the facts above come from the supplied R-12 requirement, especially “How it works today” and “Quality goals and constraints.” No repository implementation files were provided as context.

## Decision drivers

1. **Availability:** post-order provider or consumer outages must not make order placement fail.
2. **Performance:** keep checkout p95 within 500 ms when MailFast can take up to 8 seconds at p99.
3. **Correctness:** award points within one minute and prevent duplicate awards during retry or redelivery.
4. **Throughput:** support the stated 200 orders/s peak while respecting MailFast’s 50 emails/s rate limit.
5. **Modifiability and operations:** make it straightforward to add future post-order actions, using a self-hosted open-source component compatible with Kubernetes and team skills.

## Considered options

1. **Keep synchronous gRPC fan-out from checkout** to MailFast, loyalty, and analytics.
2. **Publish a durable post-order event to RabbitMQ** and have independent consumers handle email, loyalty, and analytics.
3. **Use a self-hosted Kafka event stream** for post-order events and independent consumers.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous gRPC fan-out | Uses existing Go/gRPC skills and avoids operating a broker (operability, implementation effort). | Slow or unavailable dependencies can push checkout beyond its 500 ms p95 or couple order availability to those calls (performance, availability). MailFast’s 50/s limit is below the 200/s peak (throughput). |
| B: RabbitMQ event with independent consumers | Keeps provider calls off the checkout request path and buffers temporary outages (availability, performance). Separate consumers support adding future actions (modifiability). RabbitMQ is open-source, self-hostable, Kubernetes-compatible, and two developers have used it (operability, delivery risk). | Adds broker operations and eventual consistency (operability, consistency). Retries/redelivery require idempotent consumers; broker delivery alone does not guarantee exactly-once effects (correctness). At 200 orders/s, MailFast’s 50/s limit can create a backlog (throughput, email delay). |
| C: Kafka event stream with independent consumers | Decouples producers and consumers and supports multiple downstream event consumers (modifiability, throughput). | Adds an unfamiliar operational component for the team and may be more operational complexity than these post-order actions require (operability, delivery risk). Exactly-once loyalty effects still require application-level deduplication at the points ledger (correctness). |

## Decision

Choose **Option B: publish a durable post-order event to RabbitMQ and process it asynchronously**. After the order is successfully placed, checkout should publish an event containing a stable order identifier and the minimum data needed by consumers. The email, loyalty, and analytics consumers should operate independently so one action’s latency or failure does not block the others. Add later post-order actions as separate consumers.

Use durable queues, acknowledgements, bounded retries, and a dead-letter path for failed messages. Make the loyalty consumer idempotent using the stable order identifier and a durable uniqueness check in its points ledger, so redelivery cannot award points twice. If order persistence and event publication share a database, use a transactional outbox to avoid losing an event between committing the order and publishing it; the scenario does not specify the current persistence model, so that implementation detail must be confirmed. Rate-limit the email consumer to MailFast’s 50 emails/s and monitor the backlog.

This option best balances checkout availability and latency, future modifiability, the team’s RabbitMQ experience, and the preference for self-hosted open-source components. The one-minute loyalty objective needs explicit queue and consumer capacity targets, since the scenario gives peak ingress but no loyalty processing rate or expected outage duration.

## Consequences

- **Positive:** checkout no longer waits for MailFast, loyalty, or analytics processing, protecting the 500 ms p95 target and order availability (quality attributes: performance, availability).
- **Positive:** independent consumers make adding SMS, fraud scoring, and recommendation updates easier (quality attribute: modifiability).
- **Positive:** queue buffering and email rate limiting accommodate MailFast’s lower stated throughput during short peaks (quality attributes: availability, throughput).
- **Negative:** post-order actions become eventually consistent; email and analytics may happen after the order response (quality attribute: consistency / user experience).
- **Negative:** operating RabbitMQ, monitoring queues, and handling poison messages adds work for a four-person team (quality attributes: operability, cost).
- **Risk and mitigation:** redelivery can cause duplicate points; enforce a unique order-based idempotency record in the loyalty ledger and acknowledge only after its durable write (quality attribute: correctness).
- **Risk and mitigation:** email arrival can lag during sustained 200 orders/s load because MailFast is limited to 50 emails/s; alert on queue age and backlog, and agree on an email delivery objective and peak duration before launch (quality attributes: timeliness, availability).
- **Risk and mitigation:** a broker outage or an order/event dual-write gap could lose events; use durable replicated broker storage and, if supported by the order storage design, a transactional outbox with replay monitoring (quality attribute: durability).

## Validation

Before rollout, verify with a load test at 200 orders/s that checkout \`PlaceOrder\` p95 remains at or below 500 ms while consumers are slowed or stopped. Simulate MailFast’s 1.5 s p95 / 8 s p99 latency, 0.5% failures, and 50 emails/s limit; confirm checkout remains available and observe queue age and recovery after the backlog drains. Redeliver loyalty events and confirm each order contributes points exactly once and within one minute under the agreed peak and outage conditions. Verify independent consumer failure does not block the other actions, and review broker durability, retry, dead-letter, and outbox/replay behavior before launch.

## Open questions

- What is the maximum expected duration of the 200 orders/s peak, and what email delivery delay is acceptable while MailFast is limited to 50 emails/s?
- What latency and availability objective applies to analytics event recording?
- What storage transaction currently commits an order, and can event publication use a transactional outbox?
- What are the points ledger’s uniqueness key, retention rules, and recovery procedure for a delayed event?
- What RabbitMQ durability/replication configuration and operational ownership can the team support on Kubernetes?
- What payload and versioning contract should future SMS, fraud scoring, and recommendation consumers use?