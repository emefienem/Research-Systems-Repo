# Performance Benchmarks

The benchmark suite uses **k6** to measure latency and throughput of the Payment API.

## Prerequisites

1. Start the backend locally (or point `BASE_URL` to staging).
2. Create a dedicated benchmark merchant.
3. Generate an authentication token for that merchant.

```bash
export AUTH_TOKEN=<merchant-token>
```

---

# Environment Variables

| Variable    | Description          | Default               |
| ----------- | -------------------- | --------------------- |
| BASE_URL    | Payment API base URL | http://localhost:3002 |
| AUTH_TOKEN  | Merchant JWT         | Required              |
| PAYMENT_ENV | test or live         | test                  |
| VUS         | Virtual users        | depends on benchmark  |
| DURATION    | Benchmark duration   | depends on benchmark  |

---

# Benchmarks

## 1. Create Payment Intent

Benchmarks only the `create-intent` endpoint.

```bash
BASE_URL=http://localhost:3002 \
k6 run benchmarks/k6/create-intent.js
```

Measures:

- Create Intent latency
- Throughput
- Rate limiting behaviour
- P95 latency

---

## 2. Process Payment

Measures only the **process-payment** endpoint.

Each iteration automatically performs:

```
Create Intent
      ↓
Confirm Intent
      ↓
Process Payment  ← measured
```

The setup calls are excluded from the latency metric.

```bash
BASE_URL=http://localhost:3002 \
k6 run benchmarks/k6/process-payment.js
```

Measures:

- Process Payment latency
- End-to-end setup time
- Rate limiting
- P95 latency

---

## 3. Create Dispute

Measures dispute creation.

Each iteration automatically performs:

```
Create Intent
      ↓
Confirm Intent
      ↓
Process Payment
      ↓
Create Dispute  ← measured
```

A dispute requires a successful transaction, so the benchmark creates one first.

```bash
BASE_URL=http://localhost:3002 \
k6 run benchmarks/k6/create-dispute.js
```

Measures:

- Dispute creation latency
- Balance hold performance
- Lock acquisition overhead
- P95 latency

---

## 4. Resolve Dispute

Measures dispute resolution.

Each iteration performs:

```
Create Intent
      ↓
Confirm Intent
      ↓
Process Payment
      ↓
Create Dispute
      ↓
Resolve Dispute ← measured
```

```bash
BASE_URL=http://localhost:3002 \
k6 run benchmarks/k6/resolve-dispute.js
```

Measures:

- Dispute resolution latency
- Balance release performance
- Database transaction performance

---

## 5. Refund Payment

Measures refund processing.

Each iteration performs:

```
Create Intent
      ↓
Confirm Intent
      ↓
Process Payment
      ↓
Refund Payment ← measured
```

```bash
BASE_URL=http://localhost:3002 \
k6 run benchmarks/k6/refund-payment.js
```

Measures:

- Refund latency
- Balance adjustment performance
- Partial/full refund processing
- P95 latency

---

# Benchmark Results

Each benchmark writes its summary to:

```
benchmarks/k6/results/
```

Example:

```
create-intent-summary.json
process-payment-summary.json
create-dispute-summary.json
resolve-dispute-summary.json
refund-payment-summary.json
```

These summaries are used to populate the public latency research repository.

---

## Benchmark Philosophy

These benchmarks are latency benchmarks, not stress tests.

The objective is to measure:

- P50 latency
- P95 latency
- P99 latency

under normal operating conditions.

If you want throughput testing, increase:

VUS
DURATION

and run separate load-test scenarios.

---

# Notes

- Every benchmark generates a fresh **Idempotency-Key** per iteration.
- All setup requests use newly created payment intents.
- Benchmarks should be run against a dedicated benchmark merchant.
- Use the **TEST** environment unless specifically benchmarking live infrastructure.
- Benchmarks intentionally treat HTTP **429 (Rate Limited)** as an expected outcome rather than a failure.
