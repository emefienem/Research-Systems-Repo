# Sharded Data Access Architecture

## Overview

EBINUM uses a two-stage data-access strategy for resources whose
physical database location depends on merchant ownership.

The `ShardRouter` exposes two important access patterns:

```text
executeGlobal()
executeScoped()
```

The architecture uses global access to locate a resource and scoped
access to perform merchant-owned operations on the appropriate shard.

---

## Merchant Resolution

For merchant-scoped operations, the system first resolves the merchant's
owning user.

`SubscriptionPlanService` uses:

```text
CacheService.getMerchantUserId(merchantId)
```

If the merchant cannot be resolved, the operation stops.

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
```

---

## Scoped Operations

Creating and listing subscription plans are naturally merchant-scoped.

```text
Merchant ID
    │
    ▼
Resolve merchant user
    │
    ▼
Scoped shard
    │
    ▼
subscriptionPlan query
```

This keeps tenant-owned reads and writes on the merchant's assigned
shard.

---

## Global Resource Lookup

Fetching a subscription plan by plan ID illustrates the opposite
problem.

The caller has:

```text
planId
```

but does not necessarily know which shard contains the plan.

The service therefore performs a global lookup.

```text
planId
 │
 ▼
executeGlobal()
 │
 ▼
merchantId
environment
merchant.userId
 │
 ▼
executeScoped(merchantUserId)
 │
 ▼
full plan lookup
```

The first query is intentionally selective.

It retrieves only the locator information required to determine the
correct shard.

---

## Locator Pattern

The global query returns:

```text
merchantId
environment
merchant.userId
```

It does not load the complete subscription plan.

This makes the global operation a locator rather than a full business
read.

---

## Scoped Verification

After locating the resource, the service performs the actual plan lookup
on the merchant shard.

The scoped query verifies:

```text
id = planId
merchantId = located merchant
environment = requested environment
status = ACTIVE
```

This creates multiple ownership and environment checks before a public
plan is returned.

---

## Why Two Queries?

The two-stage lookup solves a distributed-data problem.

A resource ID alone does not tell the application where the tenant data
lives.

The sequence becomes:

```text
Unknown shard
     │
     ▼
Global locator
     │
     ▼
Known merchant
     │
     ▼
Known shard
     │
     ▼
Scoped query
```

The global lookup is therefore not redundant. It establishes the routing
information required for the tenant-scoped operation.

---

## Cache-Assisted Routing

Merchant-scoped operations use `CacheService` to resolve the merchant's
user ID.

This avoids requiring every merchant operation to independently
reconstruct routing information.

The pattern is:

```text
merchantId
   │
   ▼
cache
   │
   ├── hit → merchantUserId
   │
   └── miss → merchant not found
```

The resolved user ID is then passed to `executeScoped()`.

---

## Consistency Boundary

The service performs authorization and environment checks inside the
scoped query path.

For plan retrieval:

```text
plan exists?
      │
      ▼
environment matches?
      │
      ▼
merchant matches?
      │
      ▼
active?
      │
      ▼
return plan
```

If any required condition fails, the service returns:

```text
Subscription plan not found
```

This intentionally avoids exposing unnecessary information about
resources belonging to another environment or merchant.

---

## Design Principles

### Global access should be selective

Global queries should retrieve routing information, not unnecessarily
load tenant data.

### Tenant work should be scoped

Once ownership is known, the operation should move to the merchant's
shard.

### Routing information can be cached

Merchant-to-user resolution is suitable for cache-assisted routing.

### Environment belongs in the scoped query

Routing to the correct merchant shard is not sufficient. The resource
must also belong to the requested environment.

---

## Result

This architecture allows EBINUM to preserve merchant isolation while
supporting horizontally distributed tenant data.

The important distinction is:

```text
Global lookup = locate
Scoped lookup = operate
```
