# Webhook Reconciliation Architecture

## Overview

EBINUM treats Stripe webhooks as an external reconciliation mechanism
rather than as ordinary API callbacks.

Some financial operations cannot be finalized entirely during the
original request. Stripe may require additional customer action, process
a refund asynchronously, or create and close a dispute independently of
a merchant request.

The webhook architecture closes those gaps.

---

## Architecture

```text
                         STRIPE
                           │
                           │ webhook
                           ▼
                StripeWebhookController
                           │
                  signature verification
                           │
                           ▼
                       dispatch()
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
 Payment handlers     Refund handler   Dispute handlers
          │                │                │
          ▼                ▼                ▼
 PaymentService      RefundService    DisputeService
```

---

## Raw Body Requirement

Stripe signature verification depends on the original request body.

The webhook controller therefore expects:

```text
raw request body
        +
stripe-signature
        +
webhook secret
        │
        ▼
constructWebhookEvent()
```

If signature verification fails, EBINUM returns HTTP 400.

This prevents unverified external events from reaching financial
state-changing handlers.

---

## Event Dispatch

The controller maintains an explicit event-to-handler map.

Current events include:

```text
payment_intent.succeeded
payment_intent.payment_failed
refund.updated
charge.dispute.created
charge.dispute.closed
```

Unknown event types are logged and ignored.

This keeps the webhook surface explicit while allowing Stripe to send
events that EBINUM does not currently consume.

---

## Payment Reconciliation

Payment events are delegated to:

```text
PaymentService.handleStripeWebhookEvent()
```

This is important for payments that cannot be finalized during the
initial request.

For example:

```text
Initial payment request
        │
        ▼
Stripe
        │
        ▼
requires_action
        │
        ▼
Customer authentication
        │
        ▼
Stripe finalizes payment
        │
        ▼
payment_intent.succeeded
        │
        ▼
EBINUM webhook
        │
        ▼
PaymentService
```

The webhook therefore acts as the external confirmation path.

---

## Refund Reconciliation

A live refund can return a pending state.

The architecture intentionally does not mark such a refund as succeeded
locally.

```text
Refund request
     │
     ▼
Stripe
     │
     ├── succeeded → finalize locally
     │
     ├── failed    → do not move balance
     │
     └── pending   → persist pending state
                         │
                         ▼
                   refund.updated
                         │
                         ▼
                  finalizeRefund()
```

The webhook completes the asynchronous financial state transition.

---

## Dispute Reconciliation

Disputes originate outside EBINUM.

```text
Stripe
  │
  ├── charge.dispute.created
  │
  ▼
createDisputeFromStripeEvent()

Later:

Stripe
  │
  └── charge.dispute.closed
           │
           ▼
resolveDisputeFromStripeEvent()
```

The local dispute record therefore follows the lifecycle reported by the
external processor.

---

## Failure Semantics

If webhook processing fails, the controller deliberately returns:

```text
HTTP 500
```

rather than:

```text
HTTP 200
```

The purpose is to signal that EBINUM has not successfully finalized the
event.

This allows Stripe's retry mechanism to remain useful.

---

## Reconciliation Principle

The webhook architecture follows a simple rule:

> Do not treat an external financial operation as complete until EBINUM
> has successfully reconciled the external result into its own state.

This is especially important for:

- payments requiring authentication
- asynchronous refunds
- externally created disputes
- externally finalized payment states

---

## Operational Logging

The controller records:

- event type
- error information
- webhook processing failures
- unhandled event types

Handlers also log domain-specific information during finalization.

This provides a trace from:

```text
Stripe event
   ↓
EBINUM webhook
   ↓
domain handler
   ↓
local financial state
```
