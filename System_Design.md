# System Design — kafka-notify

## Problem Statement

Traditional notification systems are often tightly coupled to the services that trigger them. When a user signs up, the signup service directly calls the email service — if the email service is slow or down, the signup flow is blocked or broken. This is a single point of failure embedded in a critical user path.

**The goal:** Decouple notification delivery entirely from the services that trigger it. A signup should succeed regardless of whether the email goes out in 1 second or 10 seconds. Failures in the notification layer must never propagate back to the caller.

---

## Core Design Principles

### 1. Asynchronous by Default
The producer never waits for the consumer. Once an event is on Kafka, the producer's job is done. This means API response times are consistently fast (just the time to validate + publish to Kafka) regardless of downstream processing complexity.

### 2. Decoupled Services
The producer and consumer share only one thing: the event schema (contract). They do not share code, databases, or network calls. Either service can be deployed, restarted, or scaled independently without affecting the other.

### 3. At-Least-Once Delivery
Kafka guarantees that a committed message will be delivered to the consumer at least once. Combined with our retry logic and DLQ, we ensure no notification event is silently dropped. The trade-off is that in edge cases, a message could be processed twice (covered under Idempotency below).

### 4. Fail Gracefully, Recover Explicitly
Consumer failures do not crash the system. They trigger retries, and ultimately land in a Dead Letter Queue where they can be inspected and replayed. The consumer always moves forward — a bad message does not block the queue.

---

## Capacity & Scale Assumptions

This is a learning project with free-tier constraints. The design decisions below acknowledge this while being honest about what a production system would look like.

| Dimension | This Project | Production Equivalent |
|---|---|---|
| Kafka Brokers | 1 | 3+ (for replication) |
| Kafka Partitions | 3 | Scale with throughput needs |
| Consumer Instances | 1 | Up to N (where N = partition count) |
| Replication Factor | 1 | 3 |
| Message Retention | 7 days (Kafka default) | Configurable per topic |
| Expected throughput | <10 events/min | Kafka handles millions/sec |

---

## Kafka Design Decisions

### Why Kafka Over Alternatives?

**vs. RabbitMQ:** RabbitMQ is a message queue — once a consumer reads and acknowledges a message, it's gone. Kafka retains messages for a configurable period regardless of consumption. This means you can replay events, add new consumers without losing history, and debug issues after the fact. For an event-driven system, this is a significant advantage.

**vs. Redis Pub/Sub:** Redis Pub/Sub is fire-and-forget. If the consumer is down when a message is published, the message is lost. Kafka's durable log ensures the consumer picks up from where it left off on restart.

**vs. Direct HTTP calls:** Synchronous, tight coupling. If the notification service is down, the caller fails. No retry, no queue, no resilience.

Kafka is the right tool here because we want **durability, replayability, and decoupling** — not just message passing.

### Topic Naming Convention

```
app.events        ← domain.noun (all events)
app.events.dlq    ← domain.noun.dlq (dead letter)
```

In a larger system with multiple domains, you might partition by domain:
```
orders.events
payments.events
users.events
```

For this project, a single `app.events` topic with `event_type` as the discriminator is simpler and sufficient.

### Partition Key Strategy

Messages are published with `user_id` as the partition key.

**Why?** Kafka guarantees ordering only within a single partition. By keying on `user_id`, all events for the same user are routed to the same partition. This means if a user signs up and immediately places an order, those events will be consumed in order. Without a key, Kafka distributes messages round-robin and ordering is not guaranteed across partitions.

### Consumer Group

Consumer group ID: `notification-service`

Kafka tracks the consumer group's committed offset per partition. This means:
- If the consumer pod restarts, it resumes from the last committed offset — no events missed
- Offset is only committed after successful processing (or after DLQ publish on failure)
- Scaling to multiple consumer instances automatically distributes partitions across them

---

## Event Schema Design

### Why a Shared Envelope?

Both producer and consumer agree on a top-level envelope:

```json
{
  "event_id": "uuid4",
  "event_type": "string",
  "timestamp": "ISO 8601",
  "version": "1.0",
  "payload": {}
}
```

**event_id:** Unique identifier per event. Used for idempotency checks (future) and tracing events across logs.

**event_type:** The routing key. The consumer's event router uses this string to dispatch to the correct handler. New event types require zero changes to the routing infrastructure — just add a new handler and register it.

**timestamp:** Set by the producer at publish time. Provides an audit trail independent of Kafka's internal timestamps.

**version:** Allows schema evolution. A consumer can check `version` and apply different parsing logic if needed. Currently `"1.0"` for all events.

**payload:** Event-specific data. Validated by Pydantic models specific to each event type.

### Schema Evolution Strategy

If the event schema needs to change:
- **Additive changes** (new optional fields): Safe. Old consumers ignore unknown fields.
- **Breaking changes** (rename/remove fields): Bump `version` to `"2.0"`. Consumer handles both versions during a migration window. This is where a Schema Registry would become valuable in production.

---

## Retry & Dead Letter Queue Design

### Why Retry?

Transient failures are real: SendGrid may return a 429 (rate limit) or 503 (temporary outage). A single failed attempt should not cause a notification to be lost. Three retries with exponential backoff handles the vast majority of transient issues.

### Retry Parameters

| Attempt | Wait Before Retry |
|---|---|
| 1st retry | 1 second |
| 2nd retry | 2 seconds |
| 3rd retry | 4 seconds |
| All failed | Publish to DLQ |

Total maximum time before DLQ: ~7 seconds per message.

### Why a Dead Letter Queue?

Events that fail all retries are not silently dropped. They are published to `app.events.dlq` with the original event payload and a failure reason. This enables:
- Manual inspection of failed events
- Replay into `app.events` once the root cause is fixed
- Alerting on DLQ message count (future monitoring)

### What Goes Into the DLQ Message?

```json
{
  "original_event": { ... },
  "failure_reason": "SendGrid API returned 500",
  "retry_count": 3,
  "failed_at": "2024-01-15T11:20:00Z"
}
```

---

## Idempotency Consideration

Kafka's at-least-once delivery guarantee means a message can, in rare circumstances (consumer restart mid-commit), be delivered more than once. This could result in duplicate emails being sent.

**Current approach:** Accepted trade-off for this project scope. The probability is low under normal operation.

**Production approach:** Before processing each event, check if `event_id` has been seen before. Store processed `event_id` values in Redis with a TTL equal to Kafka's message retention period. If already processed, skip and commit offset.

This is documented as a future improvement, not a current implementation.

---

## Kubernetes Design Decisions

### Why Two Namespaces?

Separating `kafka` and `app` namespaces provides:
- Clear ownership boundaries — the Kafka infrastructure is managed separately from application workloads
- Independent RBAC policies per namespace
- Easier cleanup — `kubectl delete namespace app` removes all application resources without touching Kafka

### Why StatefulSets for Kafka?

Kafka and Zookeeper are stateful — they need stable network identities and persistent storage. StatefulSets provide:
- Stable pod names (`kafka-0`, `kafka-1`) that don't change on restart
- Persistent Volume Claims that survive pod restarts
- Ordered startup and shutdown (Zookeeper must be ready before Kafka)

### Why a LoadBalancer for the Producer Only?

The producer is the only externally-facing service. The consumer has no HTTP endpoints that need to be reached from outside the cluster (only its `/health` endpoint, which Kubernetes accesses internally). Exposing only what is necessary reduces the attack surface.

---

## CI/CD Pipeline Design

```
Git push to main
       │
       ▼
┌─────────────────┐
│   GitHub Actions │
│                 │
│  1. Run Tests   │◄── Pytest (unit + integration)
│     (Pytest)    │    Fail fast — no build if tests fail
│                 │
│  2. Build Images│◄── docker build for producer & consumer
│                 │    Tagged with git SHA + "latest"
│                 │
│  3. Push to     │◄── DockerHub push
│     DockerHub   │    yourusername/producer-service:sha
│                 │    yourusername/consumer-service:sha
│                 │
│  4. Deploy to   │◄── doctl kubectl apply
│     DOKS        │    K8s pulls new image from DockerHub
│                 │    Rolling update — zero downtime
└─────────────────┘
```

### Secrets Management in CI/CD

GitHub Actions Secrets store:
- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `DIGITALOCEAN_ACCESS_TOKEN`
- `SENDGRID_API_KEY`

These are injected at runtime into the pipeline. They are never written to files or printed in logs.

---

## Observability

### Current (MVP)

| Tool | What It Shows |
|---|---|
| Kafka UI | Topics, messages, consumer group lag, partition distribution |
| FastAPI `/docs` | Live API documentation and manual testing |
| FastAPI `/health` | Service liveness (used by K8s probes) |
| Container logs | `kubectl logs` for real-time debugging |

### Future Improvements (Not In Scope)

- **Prometheus + Grafana:** Metrics on consumer lag, processing latency, error rates
- **Structured logging with correlation IDs:** Trace a single event through producer → Kafka → consumer → SendGrid
- **Alerting on DLQ growth:** Notify engineers when failed events accumulate
- **Distributed tracing (OpenTelemetry):** End-to-end trace visualization

---

## Failure Mode Analysis

| Failure | Impact | Recovery |
|---|---|---|
| Producer pod crashes | No new events published | K8s restarts pod automatically (Deployment) |
| Kafka broker crashes | Producer publish fails, consumer stalls | K8s restarts StatefulSet pod, messages on disk are preserved |
| Consumer pod crashes | No notifications sent during downtime | K8s restarts pod, consumer resumes from last committed offset |
| SendGrid API down | Notifications fail | Retry logic → DLQ, manual replay when SendGrid recovers |
| DLQ full / ignored | Failed events permanently lost | Operational concern — DLQ monitoring and replay process required |
| Invalid event schema | Consumer cannot parse event | Handler catches exception, publishes to DLQ with parse error reason |
