# Architecture — kafka-notify

## Overview

kafka-notify is an event-driven notification system. It is intentionally decoupled into two independent services — a **Producer** and a **Consumer** — connected exclusively through Kafka. Neither service knows about the other directly. This is the core principle of event-driven architecture.

---

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENT / CALLER                            │
│              (curl, Swagger UI, another microservice)               │
└───────────────────────────┬─────────────────────────────────────────┘
                            │  HTTP POST
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     PRODUCER SERVICE                                │
│                    (FastAPI — Port 8001)                            │
│                                                                     │
│   /events/user/signup    ──┐                                        │
│   /events/order/placed   ──┼──► Pydantic Validation                │
│   /events/payment/failed ──┘         │                             │
│                                      ▼                             │
│                            Kafka Producer Client                    │
│                           (aiokafka async)                          │
└───────────────────────────┬─────────────────────────────────────────┘
                            │  Publish Message
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        KAFKA BROKER                                 │
│                    (Apache Kafka + Zookeeper)                       │
│                                                                     │
│   Topic: app.events          (3 partitions)                        │
│   Topic: app.events.dlq      (1 partition)                         │
└───────────────────────────┬─────────────────────────────────────────┘
                            │  Subscribe / Poll
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     CONSUMER SERVICE                                │
│              (FastAPI + Background Worker — Port 8002)              │
│                                                                     │
│   Event Router                                                      │
│        │                                                            │
│        ├── user.signup    ──► Signup Handler    ──► SendGrid        │
│        ├── order.placed   ──► Order Handler     ──► SendGrid        │
│        └── payment.failed ──► Payment Handler   ──► SendGrid        │
│                                                                     │
│   On Failure:                                                       │
│        └── Retry (3x exponential backoff) ──► DLQ Publisher        │
└─────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         SENDGRID                                    │
│                    (External Email API)                             │
│                    Free Tier: 100 emails/day                        │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                         KAFKA UI                                    │
│                (Provectus — Port 8080)                              │
│         Visual monitoring of topics, messages, consumer lag         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Service Breakdown

### Producer Service

**Responsibility:** Accept HTTP requests, validate the payload, and publish a structured event to Kafka.

**Technology:** FastAPI (Python), aiokafka

**Key Design Decisions:**
- Stateless — no database, no in-memory state
- Validates every incoming request against a strict Pydantic schema before touching Kafka
- Returns a `202 Accepted` response immediately after publishing — it does not wait for the consumer to process the event. This is intentional and reflects true async event-driven behavior
- Each endpoint maps to exactly one event type. The producer enriches the payload with `event_id`, `timestamp`, and `version` before publishing

**Endpoints:**

| Method | Path | Event Published |
|---|---|---|
| POST | `/events/user/signup` | `user.signup` |
| POST | `/events/order/placed` | `order.placed` |
| POST | `/events/payment/failed` | `payment.failed` |
| GET | `/health` | — (health check) |

---

### Consumer Service

**Responsibility:** Poll Kafka for new events, route them to the correct handler, send the notification via SendGrid, and handle failures gracefully.

**Technology:** FastAPI (Python), aiokafka, SendGrid Python SDK

**Key Design Decisions:**
- Runs as a **long-running background task** alongside a minimal FastAPI app (the FastAPI app exists purely to expose a `/health` endpoint for Kubernetes liveness probes)
- Uses a **consumer group** (`notification-service`) so Kafka tracks offset progress. If the consumer restarts, it resumes from where it left off — no events are missed or double-processed under normal conditions
- **Idempotency consideration:** In production, you would store processed `event_id` values to prevent double-sends if Kafka delivers the same message twice (at-least-once delivery). This is noted as a future improvement
- Event routing is table-driven — a dictionary maps `event_type` strings to handler functions. Adding a new event type requires only adding a handler and one line to the routing table

**Retry & Dead Letter Queue Strategy:**
```
Message received
       │
       ▼
  Handler called
       │
  ┌────┴────┐
  │ Success │──► Commit offset ──► Done
  └────┬────┘
  │ Failure │
  └────┬────┘
       │
  Retry #1 (wait 1s)
       │
  Retry #2 (wait 2s)
       │
  Retry #3 (wait 4s)
       │
  All retries exhausted
       │
       ▼
  Publish to app.events.dlq
  Commit offset (move on)
  Log error with full context
```

---

### Kafka

**Topics:**

| Topic | Partitions | Replication Factor | Purpose |
|---|---|---|---|
| `app.events` | 3 | 1 | Primary event stream |
| `app.events.dlq` | 1 | 1 | Dead Letter Queue |

**Why 3 partitions for app.events?**
Partitions allow parallel consumption. With 3 partitions and 1 consumer instance, our consumer reads all 3. If we later scale to 3 consumer instances, Kafka automatically distributes one partition per instance — horizontal scaling with zero config changes.

**Why replication factor 1?**
We are running a single broker. Replication requires multiple brokers. This is a known trade-off for the free tier.

**Message Key Strategy:**
Events are published with the `user_id` as the Kafka message key. This ensures all events for the same user land on the same partition, preserving ordering per user.

---

## Kubernetes Architecture (DigitalOcean)

```
┌──────────────────────────────────────────────────────────┐
│               DigitalOcean Kubernetes Cluster             │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                   kafka Namespace                   │ │
│  │                                                     │ │
│  │   [Kafka StatefulSet]   [Zookeeper StatefulSet]     │ │
│  │   [Kafka UI Deployment]                             │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                  app Namespace                      │ │
│  │                                                     │ │
│  │   [Producer Deployment]    [Consumer Deployment]    │ │
│  │   [Producer Service]       [Consumer Service]       │ │
│  │   [Producer ConfigMap]     [Consumer ConfigMap]     │ │
│  │          └──────────┬──────────────┘                │ │
│  │                     │                               │ │
│  │            [Shared Secret]                          │ │
│  │         (SENDGRID_API_KEY,                          │ │
│  │          KAFKA_BOOTSTRAP_SERVERS)                   │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  [LoadBalancer Service] ◄── External traffic             │
│       └── Routes to Producer Service                     │
└──────────────────────────────────────────────────────────┘
```

**Kubernetes Resource Summary:**

| Resource | Count | Purpose |
|---|---|---|
| Namespaces | 2 (`kafka`, `app`) | Logical separation |
| Deployments | 3 (producer, consumer, kafka-ui) | Manages pod lifecycle |
| StatefulSets | 2 (kafka, zookeeper) | Stateful broker pods |
| Services | 4 | Internal/external networking |
| ConfigMaps | 2 | Non-sensitive configuration |
| Secrets | 1 | API keys and credentials |
| LoadBalancer | 1 | Exposes producer to internet |

---

## Data Flow — Step by Step

```
1. Client sends POST /events/order/placed with payload

2. Producer Service validates payload with Pydantic
   └── Invalid? Return 422 immediately, nothing published

3. Producer enriches payload:
   └── Adds event_id (UUID), timestamp, version

4. Producer publishes JSON message to Kafka topic: app.events
   └── Message key = user_id (ensures per-user ordering)
   └── Returns 202 Accepted to client

5. Consumer polls app.events (continuous loop)

6. Consumer deserializes message, reads event_type

7. Event Router dispatches to matching handler:
   └── order.placed → OrderHandler

8. OrderHandler calls SendGrid API with:
   └── recipient email, subject, template data

9a. SendGrid success:
    └── Consumer commits offset
    └── Kafka knows this message is processed

9b. SendGrid failure:
    └── Retry with exponential backoff (3 attempts)
    └── All retries fail → publish to app.events.dlq
    └── Commit offset (consumer moves forward)
    └── Error logged with full event context
```

---

## Local Development Architecture

When running locally with Docker Compose, all services run as containers on a shared Docker network (`kafka-network`).

```
docker-compose up
│
├── zookeeper      (port 2181)
├── kafka          (port 9092)
├── kafka-ui       (port 8080)  ← Browse in browser
├── producer       (port 8001)  ← Swagger at /docs
└── consumer       (port 8002)  ← Health at /health
```

Services communicate via Docker service names (e.g., `kafka:9092`) which Docker DNS resolves internally. No hardcoded IPs.

---

## Security Considerations (Production Notes)

These are out of scope for this project but documented for awareness:

- Producer endpoints should be protected by JWT middleware (already solved in your auth layer)
- SendGrid API key and Kafka credentials must never be committed to git — use Kubernetes Secrets and GitHub Actions Secrets exclusively
- Kafka should be configured with TLS and SASL authentication in production
- Consumer should implement idempotency by storing processed `event_id` values in Redis or a database to prevent duplicate notifications on redelivery
