**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
|frontend → productcatalogservice |system/online-boutique/kubernetes-manifests/frontend.yaml (PRODUCT_CATALOG_SERVICE_ADDR, lines 69–70); system/online-boutique/protos/demo.proto (ProductCatalogService, lines 71–74) |x |Correct interpretation from evidence|
|frontend → currencyservice |system/online-boutique/kubernetes-manifests/frontend.yaml (CURRENCY_SERVICE_ADDR, lines 71–72); system/online-boutique/protos/demo.proto (CurrencyService, lines 140–142) | x| Correct interpretation from evidence|
|frontend → cartservice |system/online-boutique/kubernetes-manifests/frontend.yaml (CART_SERVICE_ADDR, lines 73–74); system/online-boutique/protos/demo.proto (CartService, lines 23–26) | x| Correct interpretation from evidence|
|checkoutservice → productcatalogservice |system/online-boutique/kubernetes-manifests/checkoutservice.yaml (PRODUCT_CATALOG_SERVICE_ADDR, lines 57–58); system/online-boutique/protos/demo.proto (ProductCatalogService, lines 71–74) | x| Correct interpretation from evidence|
|recommendationservice → productcatalogservice |system/online-boutique/kubernetes-manifests/recommendationservice.yaml (PRODUCT_CATALOG_SERVICE_ADDR, lines 60–61); system/online-boutique/protos/demo.proto (ProductCatalogService, lines 71–74) | x| Correct interpretation from evidence|

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question): 
  - frontend set shopping assistant service address but there isn't any deployment of the service
  - it didn't specify that load generator is for development only and will not deploy to production

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?

Both A and B missed:
- The programming languages that were used to implement services, because .proto only tells parameter and return value of service
- Ambiguous service e.g. shopping assistant service, loadgenerator, because reading deployment config alone cannot make up undocumented configurations. To find better answer, it is either a documented decision or code context.

A missed:
- Dependencies between services, 
- Deployed configurations of the services
- Data stores and external systems
- HTTP service that .proto will not specify (there is no HTTP service in this case though)

because .proto only tells parameter and return value of service

2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?

The existence of services in production, including ambiguity of loadgenerator and shopping assistant service. One is overstated that it is deployed on production and one is missing in the diagram but it exists in the src and frontend deployment.
