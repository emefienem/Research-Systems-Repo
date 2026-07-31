# Payment-to-Subscription Event Architecture

## Overview

EBINUM separates payment execution from subscription lifecycle
processing.

The payment service owns payment execution. The subscription service
reacts to finalized payment events through Kafka.

This creates an asynchronous boundary between the two domains.

---

## Architecture

```text
                PAYMENT SERVICE
                      │
                      │ payment finalized
                      ▼
               payments.events
                      │
                      ▼
        SubscriptionEventConsumer
                      │
                      │ event.type
                      ▼
       payment.intent.finalized
                      │
                      ▼
 CustomerSubscriptionService
```

The subscription service does not need to synchronously participate in
the payment transaction.

---

## Event Contract

The subscription consumer expects a finalized payment event containing:

```text
paymentIntentId
subscriptionId
merchantId
merchantUserId
succeeded
```

The subscription service uses these identifiers to update subscription
state after payment finalization.

---

## Why Kafka Exists Between the Domains

Payment finalization and subscription state management are related, but
they are separate responsibilities.

The payment service is responsible for:

- payment execution
- gateway interaction
- payment state
- financial transaction completion

The subscription service is responsible for:

- subscription lifecycle
- recurring billing state
- subscription-specific transitions

Kafka provides the asynchronous communication boundary.

---

## Event Flow

### Successful payment

```text
Customer
   │
   ▼
PaymentService
   │
   ▼
Payment finalized
   │
   ▼
payments.events
   │
   ▼
SubscriptionEventConsumer
   │
   ▼
handlePaymentIntentFinalized()
   │
   ▼
Subscription state updated
```

### Failed payment

The same event structure can carry:

```text
succeeded = false
```

The subscription service can then apply the appropriate subscription
lifecycle transition without the payment service needing to directly
mutate subscription state.

---

## Consumer Group

The consumer uses:

```text
subscription-service-group
```

This provides a distinct Kafka consumer identity for subscription
processing.

The consumer subscribes to:

```text
payments.events
```

and starts from the latest events:

```text
fromBeginning: false
```

---

## Event Filtering

The consumer intentionally ignores events other than:

```text
payment.intent.finalized
```

This prevents unrelated payment events from reaching subscription
business logic.

---

## Error Handling

If message processing fails:

1.  The error is logged.
2.  The consumer throws the error.
3.  Kafka can treat the message as unsuccessful according to the
    consumer's delivery behavior.

The consumer therefore does not silently acknowledge a message that
failed to update subscription state.

---

## Domain Boundary

The architecture avoids this coupling:

```text
PaymentService
      │
      └── directly updates subscription database
```

Instead:

```text
PaymentService
      │
      ▼
Payment Event
      │
      ▼
Kafka
      │
      ▼
SubscriptionService
```

This keeps payment and subscription concerns independently evolvable.

---

## Design Principle

The payment service publishes what happened.

The subscription service decides what that event means for subscription
state.

This distinction is important because the same payment event may
eventually be consumed by other domains such as:

- analytics
- notifications
- accounting
- transaction intelligence
- reporting
