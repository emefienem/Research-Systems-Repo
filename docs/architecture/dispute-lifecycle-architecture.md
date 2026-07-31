# Dispute Lifecycle Architecture

## Overview

Disputes are fundamentally different from merchant-initiated refunds.

A live card dispute originates outside EBINUM. The customer's bank and
card network initiate the dispute process, Stripe reports the resulting
event, and EBINUM reconciles that external lifecycle into its local
dispute state.

---

## Dispute Architecture

```text
Customer
   │
   ▼
Issuing Bank
   │
   ▼
Card Network
   │
   ▼
Stripe
   │
   ├── charge.dispute.created
   │
   ▼
StripeWebhookController
   │
   ▼
DisputeService
   │
   ▼
Local Dispute Record
```

Later:

```text
Stripe
   │
   └── charge.dispute.closed
            │
            ▼
     DisputeService
            │
            ▼
      Final local outcome
```

---

## Why Disputes Are External-First

A merchant request should not be able to fabricate a live card dispute.

The `DisputeGateway` therefore exposes:

```text
retrieveDispute()
```

rather than:

```text
createDispute()
```

The external processor is the source of truth for the existence and
current state of the live dispute.

---

## Creation Lifecycle

When Stripe reports:

```text
charge.dispute.created
```

the webhook controller forwards the Stripe dispute to:

```text
DisputeService.createDisputeFromStripeEvent()
```

The local service can then create or reconcile the corresponding EBINUM
dispute record.

---

## Closure Lifecycle

When Stripe reports:

```text
charge.dispute.closed
```

the controller calls:

```text
DisputeService.resolveDisputeFromStripeEvent()
```

The dispute service determines whether the external outcome is terminal
and updates local state accordingly.

---

## Intermediate States

The implementation documentation indicates that non-terminal states such
as:

```text
needs_response
under_review
```

are filtered inside the dispute resolution logic.

This prevents an intermediate external state from being incorrectly
treated as a final dispute outcome.

---

## Retrieval and Reconciliation

The `StripeDisputeGateway` provides:

```text
retrieveDispute(gatewayDisputeId)
```

This allows EBINUM to re-fetch a dispute from Stripe when local state
needs to be checked against the external processor.

The pattern is:

```text
Local dispute
     │
     ▼
Stripe dispute ID
     │
     ▼
Stripe retrieval
     │
     ▼
External current state
     │
     ▼
Reconciliation
```

---

## Webhook Integration

Dispute lifecycle events enter through the same verified webhook
boundary used by payments and refunds.

```text
Stripe
  │
  ▼
signature verification
  │
  ▼
event dispatch
  │
  ├── dispute.created
  │
  └── dispute.closed
```

This means dispute state changes are not accepted from arbitrary client
requests.

---

## Security Boundary

The webhook controller rejects requests without a Stripe signature.

It also verifies the signature against the configured webhook secret
before dispatching the event.

Therefore:

```text
Unverified event
      │
      ▼
Rejected

Verified Stripe event
      │
      ▼
Dispute lifecycle processing
```

---

## Lifecycle Model

The overall lifecycle can be represented as:

```text
No local dispute
       │
       │ Stripe reports creation
       ▼
Dispute created
       │
       ▼
Under external review / response
       │
       ▼
Stripe closes dispute
       │
       ▼
Terminal outcome
```

The local state follows externally reported financial reality rather
than allowing the merchant API to invent a dispute lifecycle.

---

## Design Principles

### Stripe originates live disputes

EBINUM reconciles them rather than creating them.

### Webhooks are verified

Only authenticated Stripe events enter the lifecycle.

### Intermediate states are not terminal

A dispute is only resolved when Stripe reports an appropriate final
state.

### Retrieval supports reconciliation

EBINUM can retrieve the external dispute when a local record needs
verification.

### Domain logic remains in DisputeService

The Stripe gateway handles Stripe communication while the dispute
service decides how the external result maps into EBINUM's domain state.
