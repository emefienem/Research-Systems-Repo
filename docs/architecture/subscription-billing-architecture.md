# Subscription Billing and Payment Gateway Architecture

**Document Type:** Technical Design
**Status:** Implemented
**Scope:** Subscription plans, subscription checkout, payment gateway abstraction, Stripe integration, webhooks, and asynchronous subscription events

---

## 1. Overview

EBINUM's subscription system is designed as a layered billing architecture rather than a direct integration between subscription logic and a payment provider.

The system separates:

- subscription plan management;
- subscription lifecycle management;
- payment processing;
- external payment gateways;
- webhook reconciliation;
- asynchronous event processing;
- test and live environments.

This separation allows subscription functionality to remain independent of the underlying payment provider while still allowing EBINUM to use Stripe for live payment execution.

The architecture also treats the external payment provider as the authority for actual money movement in live mode.

```text
                    EBINUM Subscription System
                              │
             ┌────────────────┴────────────────┐
             │                                 │
             ▼                                 ▼
      Subscription Plans              Customer Subscriptions
             │                                 │
             └──────────────┬──────────────────┘
                            │
                            ▼
                    Payment Processing
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
             TEST MODE              LIVE MODE
                 │                     │
                 ▼                     ▼
          Simulated Rails           Stripe
                                        │
                                        ▼
                                   Webhooks
                                        │
                                        ▼
                               EBINUM Reconciliation
```

---

# 2. Design Goals

The implementation is designed around several principles.

### 2.1 Environment isolation

Subscription plans and subscription operations are explicitly associated with either:

```text
TEST
LIVE
```

An object from one environment must never be used in another.

---

### 2.2 Gateway independence

Subscription and payment services should not need to know the implementation details of Stripe.

Instead, gateway interfaces define the operations required by the payment domain.

For example:

```text
RefundGateway
DisputeGateway
```

Stripe becomes one implementation of those interfaces.

This makes the domain layer independent from Stripe-specific APIs.

---

### 2.3 External payment authority

In LIVE mode, Stripe is responsible for actual card movement.

EBINUM records and reconciles the result rather than assuming that a local database write means money has moved.

The critical principle is:

```text
Stripe confirms money movement
              ↓
EBINUM records the resulting state
```

This prevents the system from producing a local `SUCCEEDED` state when the external gateway actually failed.

---

### 2.4 Transactional local state

Once the external payment operation has produced a valid result, EBINUM maintains its internal subscription, invoice, payment, and balance state transactionally.

This preserves consistency across related financial records.

---

### 2.5 Asynchronous finalization

Some external payment operations cannot be completed synchronously.

For example, Stripe may require additional customer authentication.

The architecture therefore supports:

```text
Initial request
      ↓
Stripe
      ↓
requires_action
      ↓
Customer authentication
      ↓
Stripe webhook
      ↓
EBINUM finalization
```

---

# 3. Subscription Plan Architecture

Subscription plans are owned by merchants and isolated by environment.

A plan contains configuration such as:

- name;
- description;
- amount;
- currency;
- billing interval;
- trial period;
- setup fee;
- features;
- metadata;
- status;
- environment.

The `SubscriptionPlanService` is responsible for plan lifecycle operations.

```text
Merchant
   │
   ▼
SubscriptionPlanService
   │
   ├── createPlan()
   ├── getPlans()
   ├── getPlan()
   └── updatePlan()
   │
   ▼
ShardRouter
   │
   ▼
Merchant-scoped database
```

---

# 4. Merchant Resolution and Sharding

EBINUM does not assume that a merchant's data can always be accessed directly from the current database connection.

The subscription plan service first resolves the merchant's owning user through the cache layer.

```text
merchantId
    │
    ▼
CacheService
    │
    ▼
merchantUserId
    │
    ▼
ShardRouter.executeScoped()
    │
    ▼
Merchant's database shard
```

The implementation centralizes this operation through:

```typescript
private async resolveMerchantUserId(
  merchantId: string
): Promise<string>
```

This prevents individual subscription operations from having to independently implement merchant-to-shard resolution.

---

# 5. Creating Subscription Plans

Plan creation validates financial and billing configuration before persisting the plan.

The service rejects:

- negative plan amounts;
- negative setup fees;
- negative trial periods.

The amount and setup fee are represented using Prisma `Decimal` values rather than ordinary floating-point database values.

The resulting operation is:

```text
Validate Input
     │
     ▼
Resolve Merchant
     │
     ▼
Resolve Merchant Shard
     │
     ▼
Create Subscription Plan
     │
     ▼
Log Creation
```

The newly created plan is assigned:

```text
ACTIVE
```

and its requested environment.

---

# 6. Environment Isolation

Environment is part of the subscription plan's domain identity.

A plan created in TEST is not interchangeable with a LIVE plan.

For example:

```text
Plan A
├── merchant: M1
└── environment: TEST

Plan B
├── merchant: M1
└── environment: LIVE
```

Although both plans may belong to the same merchant, they represent separate environments.

This is enforced during retrieval.

The public checkout route therefore determines its environment from the URL:

```text
/subscribe/test/:planId
/subscribe/live/:planId
```

The frontend validates the environment and configures the API client accordingly.

---

# 7. Public Subscription Checkout

The subscription checkout flow resolves the plan from:

```text
environment
+
planId
```

The frontend calls:

```text
GET /v1/{environment}/plan?planId={planId}
```

For example:

```text
GET /v1/test/plan?planId=...
```

The backend derives the environment from the environment route middleware rather than trusting an environment value supplied independently by the client.

```text
Route
  │
  ▼
environmentRouteMiddleware
  │
  ▼
req.pathEnvironment
  │
  ▼
Environment.TEST / Environment.LIVE
```

The plan ID is then extracted from the query parameters.

---

# 8. Secure Plan Resolution

A plan lookup uses a two-stage resolution process.

First, EBINUM performs a global lookup:

```text
planId
   │
   ▼
Global database lookup
   │
   ├── merchantId
   ├── environment
   └── merchant.userId
```

This provides enough information to locate the merchant's scoped database.

The service then performs the actual plan retrieval inside the merchant's shard:

```text
merchantUserId
      │
      ▼
Scoped database
      │
      ▼
SubscriptionPlan
```

The plan must satisfy:

```text
id = requested plan
merchantId = located merchant
environment = requested environment
status = ACTIVE
```

If any of these conditions fail, the system returns:

```text
Subscription plan not found
```

This prevents inactive plans or plans belonging to another environment from being exposed through checkout.

---

# 9. Payment Gateway Abstraction

EBINUM separates payment-provider logic from domain services through gateway interfaces.

For example:

```typescript
export interface RefundGateway {
  createRefund(...): Promise<RefundGatewayResult>;
}
```

and:

```typescript
export interface DisputeGateway {
  retrieveDispute(gatewayDisputeId: string): Promise<Stripe.Dispute>;
}
```

The domain service therefore depends on a capability rather than directly depending on Stripe.

```text
RefundService
     │
     ▼
RefundGateway
     │
     ├── StripeRefundGateway
     │
     └── SimulatedRefundGateway
```

This is particularly useful for environment isolation.

---

# 10. TEST Gateway

TEST mode does not move real money.

The simulated refund gateway therefore returns an immediate successful result:

```text
SimulatedRefundGateway
        │
        ▼
success = true
status = succeeded
```

This allows the rest of EBINUM's refund lifecycle to be exercised without connecting the operation to a real financial rail.

The simulation remains behind the gateway abstraction rather than being embedded throughout the refund service.

---

# 11. LIVE Stripe Gateway

LIVE refunds use Stripe as the external financial authority.

Before creating a refund, the implementation requires a recorded Stripe charge ID.

```text
EBINUM Refund
      │
      ▼
stripeChargeId exists?
      │
      ├── NO → Reject
      │
      └── YES
           │
           ▼
       Stripe Refund
```

The internal refund ID is passed to Stripe as the idempotency key.

```text
internalRefundId
        │
        ▼
Stripe idempotency key
```

This prevents retrying the same EBINUM refund from unintentionally creating another refund at Stripe.

---

# 12. External-First Financial Operations

LIVE refunds deliberately perform the Stripe operation before changing local financial state.

The intended ordering is:

```text
Stripe
  │
  ▼
Refund Result
  │
  ├── failed
  │      └── Do not modify local balance
  │
  ├── pending
  │      └── Persist PENDING
  │
  └── succeeded
         │
         ▼
    Finalize local state
```

This ordering prevents a dangerous inconsistency:

```text
EBINUM says refunded
        but
Stripe says refund failed
```

The local database must not claim that money moved when the external payment rail rejected the operation.

---

# 13. Pending External Operations

The architecture also accounts for operations that do not immediately reach a terminal state.

For example:

```text
Stripe
  │
  ▼
PENDING
  │
  ▼
refund.updated
  │
  ▼
EBINUM finalization
```

A pending refund does not immediately move the merchant's balance.

Instead, the refund remains pending until Stripe provides the final state through a webhook.

---

# 14. Stripe Payment Intent Architecture

Live subscription payments use Stripe PaymentIntents.

The Stripe service receives:

- amount;
- currency;
- Stripe payment method;
- internal payment intent ID;
- merchant ID;
- return URL;
- optional off-session flag.

The internal payment intent ID is also used as the Stripe idempotency key.

```text
EBINUM PaymentIntent ID
          │
          ├──────────────► Stripe metadata
          │
          └──────────────► Stripe idempotency key
```

This creates an explicit relationship between EBINUM's payment lifecycle and Stripe's payment lifecycle.

---

# 15. Currency Conversion

Stripe expects amounts in the smallest currency unit.

EBINUM therefore converts major units before sending the amount to Stripe.

For the current EUR-focused live implementation:

```text
12.34 EUR
   │
   ▼
1234
```

The conversion is centralized inside:

```typescript
private toMinorUnits(amount: number)
```

This prevents Stripe-specific amount conversion logic from being scattered throughout the payment system.

---

# 16. Stripe Result Mapping

Stripe exposes its own payment statuses.

EBINUM maps those statuses into its internal payment model.

For example:

```text
Stripe: succeeded
        ↓
EBINUM: succeeded
```

Authentication-related states are mapped into:

```text
requires_action
```

while failed payment states are translated into EBINUM's internal `PaymentFailureCode` values.

This prevents the rest of the payment system from having to understand Stripe-specific decline codes.

---

# 17. Decline Code Translation

Stripe exposes provider-specific error codes such as:

```text
insufficient_funds
expired_card
card_declined
incorrect_number
fraudulent
stolen_card
```

EBINUM maps these into its own domain-level failure categories.

For example:

```text
Stripe insufficient_funds
          ↓
INSUFFICIENT_FUNDS

Stripe expired_card
          ↓
EXPIRED_CARD

Stripe fraudulent
          ↓
FRAUD_SUSPECTED
```

The result is that application-level payment intelligence can operate on EBINUM's own failure taxonomy rather than becoming coupled to Stripe's vocabulary.

---

# 18. Stripe Webhook Architecture

Webhooks provide the asynchronous reconciliation path for Stripe events.

The webhook controller deliberately receives the raw request body because Stripe signature verification depends on the original payload.

```text
Stripe
   │
   ▼
Webhook HTTP Request
   │
   ▼
Raw Body
   │
   ▼
Signature Verification
   │
   ▼
Event Dispatch
```

Invalid signatures are rejected before any event processing occurs.

---

# 19. Webhook Event Dispatch

The webhook controller maintains an explicit event-to-handler mapping.

Currently supported events include:

```text
payment_intent.succeeded
payment_intent.payment_failed
refund.updated
charge.dispute.created
charge.dispute.closed
```

The flow is:

```text
Stripe Event
     │
     ▼
Signature Verification
     │
     ▼
Event Type
     │
     ▼
Handler
```

Unknown event types are logged and ignored rather than causing the webhook endpoint to fail.

---

# 20. Asynchronous Payment Finalization

A payment may initially require additional authentication.

In this case the synchronous payment request cannot always determine the final outcome.

The architecture therefore becomes:

```text
Client
  │
  ▼
EBINUM
  │
  ▼
Stripe
  │
  ▼
requires_action
  │
  ▼
Customer Authentication
  │
  ▼
Stripe
  │
  ▼
Webhook
  │
  ▼
PaymentService
  │
  ▼
Final State
```

This allows EBINUM to support asynchronous payment completion without treating the initial request as the final source of truth.

---

# 21. Dispute Architecture

LIVE disputes have a fundamentally different lifecycle from refunds.

A merchant does not create a Stripe dispute through EBINUM.

Instead:

```text
Customer
   │
   ▼
Bank
   │
   ▼
Card Network
   │
   ▼
Stripe
   │
   ▼
charge.dispute.created
   │
   ▼
EBINUM
```

Therefore, the `DisputeGateway` is intentionally limited to retrieving an existing Stripe dispute.

The gateway is used for reconciliation rather than dispute creation.

---

# 22. Dispute Webhooks

The system handles:

```text
charge.dispute.created
charge.dispute.closed
```

Creation is delegated to:

```text
DisputeService.createDisputeFromStripeEvent()
```

Closure is delegated to:

```text
DisputeService.resolveDisputeFromStripeEvent()
```

This keeps Stripe's external dispute lifecycle synchronized with EBINUM's internal dispute records.

---

# 23. Subscription Event Processing

Subscription state changes are also integrated with EBINUM's asynchronous event architecture.

The subscription service contains a Kafka consumer subscribed to:

```text
payments.events
```

The consumer belongs to:

```text
subscription-service-group
```

This allows subscription processing to consume finalized payment events independently from the payment request itself.

---

# 24. Payment → Subscription Event Flow

The subscription consumer is specifically interested in:

```text
payment.intent.finalized
```

The resulting architecture is:

```text
Payment Processing
       │
       ▼
Payment Finalized
       │
       ▼
Kafka
       │
       ▼
payments.events
       │
       ▼
Subscription Consumer
       │
       ▼
CustomerSubscriptionService
```

The event contains:

```text
paymentIntentId
subscriptionId
merchantId
merchantUserId
succeeded
```

The subscription service can therefore update subscription state without coupling the subscription lifecycle directly to the synchronous payment request.

---

# 25. Why Kafka Is Used Here

Payment processing and subscription lifecycle management have different responsibilities.

The payment service is responsible for determining whether a payment finalized successfully.

The subscription service is responsible for determining what that outcome means for the customer's subscription.

Kafka provides a boundary between those responsibilities.

```text
Payment Domain
      │
      │ finalized event
      ▼
    Kafka
      │
      ▼
Subscription Domain
```

This reduces direct coupling between the two services.

---

# 26. Failure Handling

Webhook processing deliberately returns HTTP `500` when an event cannot be finalized.

```text
Webhook
   │
   ▼
Handler fails
   │
   ▼
HTTP 500
   │
   ▼
Stripe retry
```

This is preferable to returning `200` for an event that EBINUM failed to process because the external provider may otherwise consider the event successfully delivered.

Similarly, Kafka message-processing errors are logged and rethrown so the consumer framework can handle the failed message according to its delivery semantics.

---

# 27. Architectural Boundaries

The resulting architecture establishes several explicit boundaries.

### Subscription domain

Responsible for:

- plans;
- subscriptions;
- billing intervals;
- trials;
- invoices;
- subscription state.

### Payment domain

Responsible for:

- payment intents;
- payment execution;
- payment state;
- payment finalization.

### Gateway layer

Responsible for:

- Stripe communication;
- provider-specific request formats;
- provider-specific response mapping;
- provider-specific error translation.

### Webhook layer

Responsible for:

- signature verification;
- provider event routing;
- asynchronous reconciliation.

### Event layer

Responsible for:

- publishing payment lifecycle events;
- allowing downstream services to react asynchronously.

---

# 28. Complete Architecture

The resulting system can be represented as:

```text
                         ┌──────────────────────┐
                         │       Customer       │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         Subscription Checkout
                                    │
                                    ▼
                         Subscription Plan API
                                    │
                                    ▼
                         SubscriptionPlanService
                                    │
                          ┌─────────┴─────────┐
                          │                   │
                          ▼                   ▼
                    CacheService         ShardRouter
                          │                   │
                          └─────────┬─────────┘
                                    │
                                    ▼
                              Subscription
                                 Database
                                    │
                                    ▼
                           Customer Subscription
                                    │
                                    ▼
                            Payment Processing
                                    │
                    ┌───────────────┴────────────────┐
                    │                                │
                    ▼                                ▼
                 TEST                             LIVE
                    │                                │
                    ▼                                ▼
              Simulated Rails                    Stripe
                                                     │
                          ┌──────────────────────────┤
                          │                          │
                          ▼                          ▼
                       Webhooks                  Payment Rails
                          │
              ┌───────────┼────────────┐
              │           │            │
              ▼           ▼            ▼
           Payment      Refund       Dispute
           Events       Events        Events
              │           │            │
              └───────────┴────────────┘
                          │
                          ▼
                    EBINUM Services
                          │
                          ▼
                        Kafka
                          │
                          ▼
                 Subscription Consumer
```

---

# 29. Key Design Decisions

### Environment is part of domain state

TEST and LIVE are not merely configuration flags. They influence which records can be accessed and which payment rails can execute.

### Providers are behind interfaces

Domain services do not need to know whether a refund is being processed through Stripe or a simulated implementation.

### External financial state is reconciled

Stripe's response and webhook lifecycle determine the state of live financial operations.

### Idempotency is propagated

Internal payment and refund identifiers are used as Stripe idempotency keys, connecting EBINUM operations to provider-side retry protection.

### Webhooks are treated as state transitions

A webhook is not simply a notification. It can be the mechanism that moves an internal financial object from a pending state to a finalized state.

### Kafka separates payment and subscription domains

The payment service publishes the outcome; the subscription service determines the subscription consequence.

### Sharding is part of the application architecture

Merchant resolution and scoped database execution are explicit parts of the subscription plan lifecycle rather than hidden assumptions.

---

# 30. Result

The implementation establishes a subscription and payment architecture that is designed around clear domain boundaries rather than provider-specific logic.

The resulting flow separates:

```text
Plan Configuration
        ↓
Subscription Lifecycle
        ↓
Payment Execution
        ↓
External Gateway
        ↓
Webhook Reconciliation
        ↓
Asynchronous Domain Events
```

This allows EBINUM to integrate real payment infrastructure while retaining control over its own internal payment, subscription, risk, and financial state models.

The architecture also leaves room for additional payment providers without requiring the subscription domain itself to be rewritten around provider-specific APIs.
