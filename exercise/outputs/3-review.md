Agent: Codex (GPT-6)

# ADR-0002 review: Publish post-order events asynchronously through RabbitMQ

## Quality-attribute scenarios

| Quality attribute | Source · stimulus · environment · response · response measure |
|---|---|
| Performance | 11.11 customer · places an order while MailFast is slow or unavailable · 200 orders/s peak · return after durable acceptance without waiting for post-order actions · PlaceOrder p95 ≤ 500 ms (today ~300 ms). (Scenario: “Quality goals and constraints”; ADR: “Decision”) |
| Timeliness and correctness | Accepted order is retried or redelivered · peak load and transient loyalty/broker failure · deduplicate and retry loyalty work · exactly one award within one minute. (Scenario: “Quality goals and constraints”; ADR: “Decision”) |
| Availability and scalability | MailFast is down or limited to 50 emails/s as orders arrive at 200/s · 11.11 peak · isolate email backlog and resume after recovery · checkout succeeds and email rate stays ≤ 50/s; allowed backlog/catch-up time is unspecified. (Scenario: “Quality goals and constraints”; ADR: “Open questions”) |

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Non-risk** | Independent queues keep MailFast’s 50/s cap from throttling loyalty and analytics; async work removes its p95 1.5 s / p99 8 s latency from checkout (scenario: “Quality goals and constraints”; ADR: “Decision”). | Supports checkout performance and action isolation. |
| **Risk** | Durable acceptance depends on an outbox committed “with the accepted order,” but the storage choice and current order transaction are open (ADR: “Decision”, “Open questions”). | Without atomic order-plus-outbox commit and relay recovery, accepted work may be lost or checkout may fail, contrary to availability and correctness goals. |
| **Sensitivity point** | Peak duration, consumer capacity, outage allowance, and acceptable backlog are unspecified. Email demand exceeds its cap by 150/s at peak (scenario: “Quality goals and constraints”; ADR: “Open questions”). | These values control backlog growth, catch-up time, and whether loyalty can meet one minute. |
| **Trade-off point** | Durable writes, retries, RabbitMQ, and monitoring improve recovery but add operational work for a four-developer team and may add checkout-path latency (ADR: “Consequences”; scenario: “Quality goals and constraints”). | Availability/correctness gains carry cost and performance implications. |
| **Risk** | Loyalty deduplication works only if the idempotency record and points award are atomic. MailFast idempotency is unresolved (ADR: “Decision”, “Open questions”). | Redelivery can duplicate awards if the ledger update is not atomic; retry after an ambiguous email timeout can send duplicate confirmations. |

## Missing or weak

- Compare a database-backed outbox/worker option with RabbitMQ; the ADR compares only synchronous gRPC, RabbitMQ, and Kafka (ADR: “Considered options”).
- Define the order/outbox transaction and relay recovery, including publisher confirmation and duplicate publication handling (ADR: “Decision”, “Open questions”).
- Set peak duration, acceptable backlog/catch-up time, consumer capacity, and loyalty/broker outage budget before claiming the one-minute target is achievable (ADR: “Open questions”).
- Decide MailFast idempotency and safe retries. Also, implementation claims about current persistence are not verifiable from the allowed evidence: the scenario confirms synchronous calls and warning-only email failure, but not storage behavior (ADR: “Context and problem”; scenario: “How it works today”).

## Verdict

**Accept with changes.** (1) Define and demonstrate atomic order-plus-outbox acceptance and recovery. (2) Bound peak, outage, and backlog assumptions and show one-minute loyalty is feasible. (3) Resolve MailFast idempotency and retry behavior.
