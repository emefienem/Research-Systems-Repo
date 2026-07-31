# Environment Isolation Architecture

## Overview

EBINUM separates test and live payment execution at the API, domain,
gateway, and data-access boundaries.

The environment is not treated as a UI-only setting. It becomes part of
the request path and financial domain state.

---

# Design Objectives

Environment isolation aims to:

- prevent accidental production activity
- protect real customer data
- support safe experimentation
- simplify operational governance

---

## Request Path

A request enters through an environment-specific route.

Example:

```text
/v1/test/plan
```

The environment middleware resolves the path environment and exposes it
through:

```text
req.pathEnvironment
```

The controller then maps the path value to the internal enum:

```text
test → Environment.TEST
live → Environment.LIVE
```

---

## Subscription Plan Isolation

Subscription plans contain an environment.

A plan lookup therefore requires:

```text
planId
+
environment
```

A plan created in TEST cannot be retrieved through the LIVE environment.

The service explicitly validates:

```text
planLocator.environment === environment
```

before returning the plan.

---

## Payment Environment

The frontend also validates the route environment:

```text
test
live
```

The subscription checkout hook converts the validated value into the
internal `Environment` type and sets the API client environment.

```text
URL
 │
 ├── env
 │
 ▼
isEnvironment()
 │
 ▼
Environment
 │
 ▼
api.setEnvironmentDirect()
```

The React Query key includes the environment:

```text
["subscription-plan", planId, environment]
```

This prevents test and live plan data from sharing the same query
identity.

---

## Gateway Isolation

Test refunds use:

```text
SimulatedRefundGateway
```

Live refunds use:

```text
StripeRefundGateway
```

Therefore:

```text
TEST
 │
 ▼
Simulation
 │
 ▼
No real money movement

LIVE
 │
 ▼
Stripe
 │
 ▼
Real financial operation
```

This is an important safety boundary.

---

## Data Isolation

Subscription queries include the environment in their database filters.

For example:

```text
merchantId
environment
status
```

are used when retrieving active plans.

The same plan ID is therefore not sufficient to cross environments.

---

## Safety Properties

Environment isolation provides several protections.

### No accidental live processing from test checkout

The environment determines which gateway behavior is used.

### No cross-environment plan retrieval

Plan lookups require matching environment state.

### No shared React Query identity

The environment forms part of the client-side query key.

### No accidental live refund simulation

The refund gateway implementation is selected according to the
environment.

---

## Environment as Domain State

The architecture treats environment as part of financial identity rather
than as configuration alone.

A useful conceptual model is:

```text
Resource Identity
=
Resource ID
+
Merchant
+
Environment
```

This is especially important for:

- payment intents
- subscription plans
- subscriptions
- invoices
- refunds
- disputes

---

# Why Isolation Matters

Environment isolation protects both operators and users.

Benefits include:

- safer development cycles
- reduced operational risk
- improved compliance posture
- easier incident recovery

---

## Design Principle

Test mode should be capable of exercising the same application
architecture without creating real financial consequences.

Live mode should execute through real external financial rails.

The separation therefore exists at multiple layers rather than only in
the frontend.
