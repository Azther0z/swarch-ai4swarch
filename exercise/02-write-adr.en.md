**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | Mentioned in "Context and problem" section, but some numbers are not appeared in "Decision drivers" section e.g. MailFast p95 1.5s| Mentioned in "Context and problem" section, but some numbers are not appeared in "Decision drivers" section e.g. MailFast p95 1.5s and p99 8s|
| Options considered (list them) | Synchronous gRPC fan-out, RabbitMQ event with independent consumers, Kafka event stream with independent consumers | Synchronous gRPC, RabbitMQ with independent action queues, Self-hosted Kafka|
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | Make the loyalty consumer idempotent using the stable order identifier and a durable uniqueness check in its points ledger. Rate-limit the email consumer to MailFast’s 50 emails/s and monitor the backlog| The loyalty consumer must use the order ID as an idempotency key and atomically record that key with the points award. Configure the email consumer to respect MailFast’s 50 emails/s limit and retry transient failures with backoff. |
| Anything invented (a number, tool or fact not in the scenario)? |None |None |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 |Explicitly include more number constraints in decision drivers | To ensure consistent context in the document|
| 2 | Remix the list of open questions| Drop trivial questions, drop out-of-scope questions, |
