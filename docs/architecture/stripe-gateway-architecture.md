# Stripe Gateway Architecture

## Overview

EBINUM isolates Stripe-specific payment behavior behind gateway-oriented
service boundaries. The objective is to keep payment-domain services
responsible for business state while the Stripe integration layer is
responsible for communicating with Stripe and translating Stripe
responses into EBINUM's internal model.

The current implementation covers:

- PaymentIntent creation and confirmation
- PaymentIntent retrieval
- Refund creation
- Dispute retrieval
- Stripe webhook signature verification
- Stripe-to-EBINUM status and failure-code mapping
- Stripe idempotency for payment and refund operations

This separation provides a clear boundary between EBINUM's internal
payment lifecycle and the external processor.

---

## Architecture

```text
                    EBINUM PAYMENT DOMAIN
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
       PaymentService   RefundService  DisputeService
             │              │              │
             ▼              ▼              ▼
       StripeService   RefundGateway   DisputeGateway
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                         Stripe
```

The `StripeService` contains the Stripe SDK interaction. Refunds and
disputes additionally expose domain-oriented gateway interfaces so the
surrounding services do not need to depend directly on Stripe-specific
implementation details.

---

## StripeService Responsibilities

`StripeService` owns the direct Stripe API operations.

### PaymentIntent confirmation

`createAndConfirmPaymentIntent()` accepts:

- amount
- currency
- Stripe payment method ID
- EBINUM payment intent ID
- merchant ID
- return URL
- optional off-session flag

The EBINUM payment intent ID is supplied as the Stripe idempotency key.

Stripe receives EBINUM metadata containing:

- `ebinumPaymentIntentId`
- `merchantId`

This creates a traceable relationship between the external Stripe object
and the internal payment record.

---

## Currency Conversion

EBINUM currently converts major currency units to minor units before
sending amounts to Stripe.

```text
EBINUM amount
    │
    ▼
amount × 100
    │
    ▼
Stripe minor units
```

The implementation currently assumes two-decimal currencies for the live
scope.

This conversion is intentionally centralized inside `StripeService` so
callers do not need to understand Stripe's amount representation.

---

## Payment Status Mapping

Stripe statuses are translated into EBINUM's internal result model.

```text
Stripe succeeded
       │
       ▼
success = true
status = succeeded

Stripe requires_action
       │
       ▼
status = requires_action

Stripe failed
       │
       ▼
status = failed
```

For successful payments, the service extracts the Stripe PaymentIntent
ID and associated charge ID.

For action-required states, the Stripe client secret is returned when
available so the higher-level payment flow can continue through the
required customer authentication step.

---

## Failure Mapping

Stripe errors are translated into EBINUM `PaymentFailureCode` values.

Examples include:

```text
insufficient_funds → INSUFFICIENT_FUNDS
expired_card       → EXPIRED_CARD
card_declined      → CARD_DECLINED
invalid_number     → INVALID_CARD
fraudulent         → FRAUD_SUSPECTED
```

Unknown Stripe failures fall back to:

```text
GATEWAY_ERROR
```

This prevents Stripe-specific error vocabulary from leaking throughout
the rest of the payment system.

---

## Idempotency

Payment confirmation uses the internal payment intent ID as the Stripe
idempotency key.

```text
EBINUM PaymentIntent
        │
        ▼
internalPaymentIntentId
        │
        ▼
Stripe idempotency key
```

Refunds similarly use the internal refund ID.

This gives external operations a stable retry boundary.

---

## Refund Gateway

Refund behavior is abstracted behind:

```text
RefundGateway
```

The interface returns:

- success
- status
- gateway refund ID
- failure reason

The live implementation is:

```text
StripeRefundGateway
```

The test implementation is:

```text
SimulatedRefundGateway
```

This allows test mode to simulate financial behavior without moving real
money.

---

## Dispute Gateway

Disputes differ from refunds.

A merchant does not create a live card dispute through EBINUM. A dispute
originates externally:

```text
Customer
   ↓
Bank
   ↓
Card Network
   ↓
Stripe
   ↓
charge.dispute.created
   ↓
EBINUM
```

The `DisputeGateway` therefore exposes retrieval rather than creation.

---

## Webhook Boundary

Stripe webhooks enter through `StripeWebhookController`.

The controller:

1.  Reads the Stripe signature.
2.  Verifies the raw request body.
3.  Constructs the Stripe event.
4.  Dispatches the event to the appropriate handler.
5.  Returns HTTP 500 when finalization fails so Stripe can retry.

This makes webhook processing part of the external-to-internal
reconciliation boundary.

---

## Design Principles

### External integration is isolated

Stripe SDK calls remain concentrated in the gateway/service layer.

### Internal state remains authoritative for EBINUM workflows

Stripe results are translated into EBINUM domain results rather than
allowing Stripe-specific objects to propagate through the application.

### Idempotency is explicit

External money movement uses stable internal identifiers as idempotency
boundaries.

### Test mode does not move money

Simulation gateways provide deterministic behavior for test
environments.

### External events are reconciled

Stripe webhooks complete asynchronous flows and synchronize external
outcomes with internal state.
