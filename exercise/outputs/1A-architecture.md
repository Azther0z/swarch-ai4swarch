# Online Boutique: Recovered Architecture (Run A)

## Agent

OpenAI Codex, GPT-6 (exact model variant not exposed).

## Files read

- `prompts/1-recover-architecture.md`
- `system/online-boutique/protos/demo.proto`
- `exercise/outputs/1A-architecture.md` (read back to verify the generated output)

## Containers table

The proto defines nine named RPC service interfaces. The table and diagram represent those interfaces as logical service containers; this file does not establish their deployed process boundaries or implementation languages.

| Name | Technology/language | Responsibility | Evidence |
|---|---|---|---|
| CartService | Protocol Buffers v3 RPC interface; implementation language not evidenced | Adds items, retrieves a user's cart, and empties a cart | `system/online-boutique/protos/demo.proto` — `service CartService`, `AddItem`, `GetCart`, `EmptyCart` |
| RecommendationService | Protocol Buffers v3 RPC interface; implementation language not evidenced | Returns product recommendations | `system/online-boutique/protos/demo.proto` — `service RecommendationService`, `ListRecommendations` |
| ProductCatalogService | Protocol Buffers v3 RPC interface; implementation language not evidenced | Lists, retrieves, and searches products | `system/online-boutique/protos/demo.proto` — `service ProductCatalogService`, `ListProducts`, `GetProduct`, `SearchProducts` |
| ShippingService | Protocol Buffers v3 RPC interface; implementation language not evidenced | Gets shipping quotes and ships orders | `system/online-boutique/protos/demo.proto` — `service ShippingService`, `GetQuote`, `ShipOrder` |
| CurrencyService | Protocol Buffers v3 RPC interface; implementation language not evidenced | Lists supported currencies and converts money | `system/online-boutique/protos/demo.proto` — `service CurrencyService`, `GetSupportedCurrencies`, `Convert` |
| PaymentService | Protocol Buffers v3 RPC interface; implementation language not evidenced | Charges the provided amount and returns a transaction ID | `system/online-boutique/protos/demo.proto` — `service PaymentService`, `Charge` |
| EmailService | Protocol Buffers v3 RPC interface; implementation language not evidenced | Sends an order confirmation | `system/online-boutique/protos/demo.proto` — `service EmailService`, `SendOrderConfirmation` |
| CheckoutService | Protocol Buffers v3 RPC interface; implementation language not evidenced | Places an order | `system/online-boutique/protos/demo.proto` — `service CheckoutService`, `PlaceOrder` |
| AdService | Protocol Buffers v3 RPC interface; implementation language not evidenced | Returns ads for supplied context keywords | `system/online-boutique/protos/demo.proto` — `service AdService`, `GetAds`, `AdRequest.context_keys` |

## Dependencies table

No runtime caller-to-callee dependencies are evidenced in the provided file. It declares RPC methods and their message types, but not which component invokes them.

| Caller → callee | Protocol | Evidence |
|---|---|---|

## Mermaid diagram

```mermaid
C4Container
  title Containers: Online Boutique (service interfaces evidenced by demo.proto)
  Person(end_user, "End user", "Identity and interactions are not evidenced in demo.proto")
  System_Boundary(boutique, "Online Boutique") {
    Container(cart, "CartService", "Protocol Buffers v3 RPC interface", "Adds items, retrieves a user's cart, and empties a cart")
    Container(recommendation, "RecommendationService", "Protocol Buffers v3 RPC interface", "Returns product recommendations")
    Container(catalog, "ProductCatalogService", "Protocol Buffers v3 RPC interface", "Lists, retrieves, and searches products")
    Container(shipping, "ShippingService", "Protocol Buffers v3 RPC interface", "Gets shipping quotes and ships orders")
    Container(currency, "CurrencyService", "Protocol Buffers v3 RPC interface", "Lists supported currencies and converts money")
    Container(payment, "PaymentService", "Protocol Buffers v3 RPC interface", "Charges the provided amount and returns a transaction ID")
    Container(email, "EmailService", "Protocol Buffers v3 RPC interface", "Sends an order confirmation")
    Container(checkout, "CheckoutService", "Protocol Buffers v3 RPC interface", "Places an order")
    Container(ad, "AdService", "Protocol Buffers v3 RPC interface", "Returns ads for supplied context keywords")
  }
  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

### Zed-compatible fallback

If this Zed version does not render the C4 diagram above, this standard Mermaid flowchart shows the same service-interface nodes. It adds no caller-to-callee links because none are evidenced.

```mermaid
flowchart TB
  end_user["End user: identity and interactions not evidenced"]
  subgraph boutique["Online Boutique"]
    cart["CartService: Protocol Buffers v3 RPC interface"]
    recommendation["RecommendationService: Protocol Buffers v3 RPC interface"]
    catalog["ProductCatalogService: Protocol Buffers v3 RPC interface"]
    shipping["ShippingService: Protocol Buffers v3 RPC interface"]
    currency["CurrencyService: Protocol Buffers v3 RPC interface"]
    payment["PaymentService: Protocol Buffers v3 RPC interface"]
    email["EmailService: Protocol Buffers v3 RPC interface"]
    checkout["CheckoutService: Protocol Buffers v3 RPC interface"]
    ad["AdService: Protocol Buffers v3 RPC interface"]
  end
```

## Gaps

- **Runtime dependencies and protocols:** Not evidenced in the provided file. A caller's service implementation or client-wiring configuration showing which RPC stubs it invokes and how targets are addressed would settle this.
- **Deployed container boundaries and implementation technologies:** The proto names service interfaces but does not show whether each runs as a separate process or its implementation language. Service entry-point/build files and deployment manifests would settle this.
- **Data stores and external systems:** None are described. The relevant service implementation or storage/integration configuration would identify them.
- **End-user identity and entry point:** Not evidenced. A frontend or user-facing entry-point file would identify the actor and its connection to the system.
