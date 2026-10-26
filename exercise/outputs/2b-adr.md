Agent: Codex (GPT-6)

# ADR-0002: Publish post-order events asynchronously through RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not specified

## Context and problem

Requirement R-12 adds three actions after an order is placed: send confirmation through MailFast, award loyalty points, and record an analytics event. More actions (SMS, fraud scoring, recommendations) are expected next year (docs/scenario.md, “The new requirement (R-12)”).

Today `checkoutservice.PlaceOrder` charges the card, ships the order, empties the cart, then calls `sendOrderConfirmation` before returning. The email error is logged but does not fail the order (system/online-boutique/src/checkoutservice/main.go, `PlaceOrder`, `sendOrderConfirmation`). The scenario describes the payment, shipping, and email calls as synchronous gRPC calls (docs/scenario.md, “How it works today”). The code constructs and returns an `OrderResult`; this file does not show durable order persistence or an event mechanism.

The checkout p95 must remain at or below 500 ms, while MailFast has a 1.5 s p95 and 8 s p99. MailFast fails about 0.5% of calls and is limited to 50 emails/s, against a peak of 200 orders/s. Loyalty must be awarded within one minute and exactly once. Orders must not fail because email, loyalty, or analytics is down (docs/scenario.md, “Quality goals and constraints”).

## Decision drivers

1. **Availability:** post-order provider outages must not fail order placement.
2. **Performance:** keep post-order provider latency out of the checkout response path and meet the 500 ms p95 target.
3. **Correctness:** award points within one minute without double-awarding on retries or redelivery.
4. **Scalability:** absorb 200 orders/s and MailFast’s 50 emails/s rate limit without coupling the other actions to email throughput.
5. **Modifiability and operations:** add future post-order actions independently, using an open-source, self-hosted component on Kubernetes that fits the team’s Go and gRPC experience and some RabbitMQ experience.

## Considered options

1. Keep synchronous gRPC calls from `checkoutservice` to each post-order service.
2. Publish a post-order event to RabbitMQ and process each action asynchronously.
3. Publish post-order events to a self-hosted Kafka cluster and process them asynchronously.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Synchronous gRPC | Uses the team’s existing gRPC experience; direct request/response is simple | MailFast’s p95 and p99 exceed the entire checkout p95 budget; rate limiting and provider outages complicate checkout; future actions add coupling and latency |
| RabbitMQ with independent action queues | Open-source and self-hostable; two developers have used it; queues buffer bursts and isolate email throughput from loyalty and analytics; new actions can subscribe independently | Adds broker operations and delivery/retry handling; redelivery means consumers must be idempotent; durable handoff and queue capacity need design and monitoring |
| Self-hosted Kafka | Durable event log can support replay and multiple consumers; suits growing analytics/event use | Adds operational and team learning cost; replay and consumer offset behavior still require idempotent loyalty handling; no Kafka experience is stated |

## Decision

Adopt asynchronous post-order processing with RabbitMQ. After the order is accepted, record a durable post-order event and publish it to a durable exchange with a separate queue for email, loyalty, and analytics. Keep each consumer independent so MailFast’s 50 emails/s limit does not throttle loyalty or analytics.

Use a transactional outbox with the accepted order so a broker outage cannot lose the handoff after the order succeeds. The existing `PlaceOrder` code does not show durable order storage, so this requires adding or selecting a durable persistence mechanism; that storage choice remains open.

Configure the email consumer to respect MailFast’s 50 emails/s limit and retry transient failures with backoff. The loyalty consumer must use the order ID as an idempotency key and atomically record that key with the points award, so duplicate delivery cannot award points twice. Retry transient failures and alert on queue age so a healthy loyalty service can meet the one-minute objective. Analytics failures must likewise be isolated from checkout.

## Consequences

- **Positive:** MailFast latency and outages are removed from the synchronous checkout response path (performance, availability).
- **Positive:** independent queues let email obey its provider rate limit while loyalty and analytics continue at peak order load (scalability, availability).
- **Positive:** consumers can be added for SMS, fraud scoring, and recommendations without adding synchronous calls to `PlaceOrder` (modifiability).
- **Negative:** RabbitMQ, durable event storage, retries, and queue monitoring add operational work for a four-developer team (operability, cost).
- **Negative:** loyalty completion is eventual rather than part of the order response; the one-minute target depends on consumer capacity and outage duration (timeliness).
- **Risks and mitigations:** RabbitMQ delivery can be repeated. Require durable idempotency records and an atomic points-ledger update in `loyaltyservice`; test duplicate and out-of-order delivery. Use publisher confirms, durable queues, retry limits, and a dead-letter queue, with alerts for undelivered events and queue age (correctness, availability, operability).
- **Risks and mitigations:** a long provider outage or sustained input above consumer capacity can exceed the one-minute loyalty target or grow the email backlog. Monitor queue depth and age, size consumers against measured load, and define escalation and recovery procedures (timeliness, scalability).

## Validation

- Load-test `PlaceOrder` at 200 orders/s and verify p95 latency is at most 500 ms while MailFast is delayed or unavailable.
- Simulate a MailFast outage and verify order placement succeeds, emails resume after recovery, and the email worker stays within 50 sends/s.
- Deliver the same order event repeatedly and verify the loyalty ledger awards points exactly once and records awards within one minute under the expected peak load.
- Stop RabbitMQ and verify accepted orders remain recoverable from the outbox; restart it and verify events are delivered.
- Review queue-age, retry, dead-letter, and provider-rate metrics after the test.

## Open questions

- Which durable datastore will hold accepted orders, the outbox, and loyalty idempotency/points records? The provided checkout code does not show an order datastore.
- What is the expected duration of the 200 orders/s peak, and what event backlog and catch-up time are acceptable?
- How long may `loyaltyservice` or RabbitMQ be unavailable while still meeting the one-minute award target?
- Does MailFast support idempotency keys, and what retry policy is safe when a timeout leaves the send outcome unknown?
- What are the analytics event schema, retention, and delivery expectations?
- Who are the ADR deciders?
