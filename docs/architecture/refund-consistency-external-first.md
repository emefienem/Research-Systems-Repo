# Refund Consistency & External-First Financial Operations

## Overview

Refunds are a financial consistency boundary between EBINUM and an
external payment processor.

The central rule of the refund architecture is:

> The external financial operation must succeed before EBINUM records a
> local successful refund or changes the merchant balance.

This prevents a dangerous state where EBINUM believes money was returned
when the external processor did not actually move the funds.

---

## Architecture

```text
Merchant Refund Request
          │
          ▼
     RefundService
          │
          ▼
     RefundGateway
          │
     ┌────┴────┐
     │         │
     ▼         ▼
   TEST      LIVE
     │         │
     ▼         ▼
 Simulation  Stripe
     │         │
     └────┬────┘
          ▼
    Gateway Result
          │
          ▼
 Local financial state
```

---

## External-First Principle

For LIVE refunds:

```text
EBINUM request
      │
      ▼
Stripe refund
      │
      ├── failed
      │     └── stop
      │
      ├── pending
      │     └── persist pending
      │
      └── succeeded
            │
            ▼
      update local state
```

The local database does not claim a successful refund before Stripe
confirms it.

---

## Why This Ordering Matters

The dangerous sequence would be:

```text
Write local refund = SUCCEEDED
        │
        ▼
Move merchant balance
        │
        ▼
Stripe refund fails
```

That creates a financial mismatch.

EBINUM would have reduced the merchant's balance while the customer did
not actually receive the external refund.

The implementation therefore reverses the order:

```text
Stripe
  │
  ▼
External outcome
  │
  ▼
Local financial mutation
```

---

## Gateway Result

The `RefundGateway` returns:

```text
success
status
gatewayRefundId
failureReason
```

The status can be:

```text
pending
succeeded
failed
```

This allows the refund service to distinguish final outcomes from
asynchronous outcomes.

---

## Failed Refund

When Stripe returns failure:

```text
Stripe
  │
  ▼
failed
  │
  ▼
No successful Refund row
No merchant balance movement
```

The failure reason can be surfaced for investigation and operational
handling.

---

## Pending Refund

A pending result represents an operation whose final financial state is
not yet known.

The expected flow is:

```text
Stripe
  │
  ▼
pending
  │
  ▼
Persist PENDING
  │
  ▼
refund.updated webhook
  │
  ▼
finalizeRefund()
```

The balance should not be finalized as refunded until the external state
is finalized.

---

## Successful Refund

Only a successful external result permits local financial completion.

```text
Stripe succeeded
      │
      ▼
Local Refund = SUCCEEDED
      │
      ▼
Merchant balance adjustment
      │
      ▼
Audit record
```

The exact local transaction ordering remains inside the refund service's
database transaction boundary.

---

## Idempotency

The internal refund ID is used as the Stripe idempotency key.

```text
Internal Refund ID
       │
       ▼
Stripe idempotency key
```

This protects retries from accidentally creating duplicate external
refunds.

---

## Test Mode

Test mode uses:

```text
SimulatedRefundGateway
```

The simulated implementation returns:

```text
success = true
status = succeeded
```

No real money moves.

This allows the refund service to exercise the same high-level financial
workflow without connecting test checkout behavior to real Stripe funds.

---

## Gateway Abstraction

The refund service depends on:

```text
RefundGateway
```

rather than directly depending on Stripe.

That creates this boundary:

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

This keeps external processor behavior replaceable and makes
environment-specific behavior explicit.

---

## Consistency Principle

A refund is not merely a database update.

It represents an external financial state transition followed by an
internal accounting state transition.

Therefore:

```text
External money movement
        ↓
Confirmed outcome
        ↓
Internal financial state
```

is the primary consistency model.
