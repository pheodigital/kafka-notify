# kafka-notify

A production-style, event-driven notification service built with FastAPI, Apache Kafka, and deployed on Kubernetes via DigitalOcean.

---

## Final Confirmed Stack

| Layer | Technology | Why |
|---|---|---|
| API Framework | FastAPI (Python) | Async-native, modern, auto Swagger docs |
| Message Broker | Apache Kafka + Zookeeper | Core event backbone |
| Notification Delivery | SendGrid | Free tier, reliable email delivery |
| Containerization | Docker | Industry standard |
| Orchestration | DigitalOcean Kubernetes (DOKS) | Managed K8s, $200 free credit |
| Image Registry | DockerHub | Free, simple, CI/CD friendly |
| Monitoring | Kafka UI (Provectus) | Visual Kafka topic & consumer management |
| CI/CD | GitHub Actions | Free, integrates with DockerHub + DOKS |
| Local Dev | Docker Compose | One command local environment |
| Testing | Pytest | Python standard testing framework |
| Schema Validation | Pydantic (built into FastAPI) | Type-safe event models |

---

## Accounts You Need — Set These Up First

Before running anything locally or deploying to the cloud, make sure the following accounts and tools are ready.

### 1. GitHub
Used for source control, monorepo hosting, and GitHub Actions CI/CD pipelines. Assumed to be already set up.

### 2. DockerHub
Used as the container image registry. All built Docker images are pushed here and pulled by the Kubernetes cluster.

- Sign up at [hub.docker.com](https://hub.docker.com)
- Note your **DockerHub username** — it prefixes every image name
  - Example: `yourusername/producer-service:latest`
- Create an **Access Token** under Account Settings → Security → New Access Token
- Save the token — it is used in GitHub Actions secrets

### 3. SendGrid
Used for sending notification emails (welcome emails, order confirmations, payment failure alerts).

- Sign up at [sendgrid.com](https://sendgrid.com) — free tier allows 100 emails/day
- Navigate to **Settings → API Keys → Create API Key** (Full Access)
- Save the API key immediately — it is shown only once
- Navigate to **Settings → Sender Authentication** and verify your sender email address
  - This is the `from` address that appears on all outgoing notifications

### 4. DigitalOcean
Used for managed Kubernetes hosting.

- Sign up at [digitalocean.com](https://digitalocean.com) — $200 free credit for new accounts
- Install the `doctl` CLI: [Install Guide](https://docs.digitalocean.com/reference/doctl/how-to/install/)
- Authenticate: `doctl auth init` using your DigitalOcean API token
- The Kubernetes cluster is created in PR 4, not during local setup

---

## Detailed Event Schema

All messages on Kafka follow a shared envelope structure. Both the producer and consumer must respect this contract. The `payload` field varies by event type.

### Event Envelope (all events)

```json
{
  "event_id": "uuid4",
  "event_type": "user.signup | order.placed | payment.failed",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "1.0",
  "payload": {}
}
```

### user.signup

Triggered when a new user registers. Sends a welcome email.

```json
{
  "event_id": "a1b2c3d4-...",
  "event_type": "user.signup",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "1.0",
  "payload": {
    "user_id": "uuid",
    "email": "user@example.com",
    "full_name": "John Doe"
  }
}
```

### order.placed

Triggered when a user places an order. Sends an order confirmation email.

```json
{
  "event_id": "b2c3d4e5-...",
  "event_type": "order.placed",
  "timestamp": "2024-01-15T11:00:00Z",
  "version": "1.0",
  "payload": {
    "order_id": "uuid",
    "user_id": "uuid",
    "email": "user@example.com",
    "items": ["Item A", "Item B"],
    "total_amount": 99.99,
    "currency": "USD"
  }
}
```

### payment.failed

Triggered when a payment attempt fails. Sends a payment failure alert email.

```json
{
  "event_id": "c3d4e5f6-...",
  "event_type": "payment.failed",
  "timestamp": "2024-01-15T11:15:00Z",
  "version": "1.0",
  "payload": {
    "payment_id": "uuid",
    "user_id": "uuid",
    "email": "user@example.com",
    "amount": 49.99,
    "currency": "USD",
    "failure_reason": "insufficient_funds"
  }
}
```

### Kafka Topics

| Topic | Partitions | Purpose |
|---|---|---|
| `app.events` | 3 | Main event stream (all event types) |
| `app.events.dlq` | 1 | Dead Letter Queue for failed processing |

---

## Test Cases — Planned

### Unit Tests (Producer Service)
- Valid event payload passes Pydantic schema validation
- Invalid/malformed payload returns 422 with descriptive error
- Kafka producer publishes message to the correct topic (`app.events`)
- Each endpoint maps to the correct `event_type` string
- Missing required fields in payload are rejected before Kafka publish

### Unit Tests (Consumer Service)
- Consumer correctly routes `user.signup` to the signup handler
- Consumer correctly routes `order.placed` to the order handler
- Consumer correctly routes `payment.failed` to the payment handler
- Unknown `event_type` is handled gracefully without crashing
- SendGrid handler builds correct email payload for each event type
- Retry logic triggers exactly 3 times on handler failure
- After 3 failed retries, message is published to `app.events.dlq`
- Consumer group offset advances only after successful processing

### Integration Tests
- `POST /events/user/signup` → message visible in Kafka topic via Kafka UI
- `POST /events/order/placed` → message visible in Kafka topic via Kafka UI
- `POST /events/payment/failed` → message visible in Kafka topic via Kafka UI
- Message in Kafka → SendGrid API called with correct recipient and template data
- Simulated SendGrid failure → message appears in `app.events.dlq` topic
- Consumer group lag returns to zero after processing a batch of messages

---

## What We're NOT Building (And Why)

### Authentication on the Producer API
The JWT and OAuth layer is intentionally excluded. This project focuses on event-driven architecture. In production, you would place an API gateway or middleware in front of the producer endpoints to enforce authentication. A note is left in the producer service where this would hook in.

### A Database
Events are the source of truth for this service. The consumer processes and acts on events without persisting them. Adding a database would shift the focus away from Kafka and event-driven patterns.

### Confluent Schema Registry
A schema registry manages and enforces event schema versions across producers and consumers at runtime. For this project, Pydantic models on both services serve as the schema contract. Schema Registry adds meaningful operational complexity that is not appropriate for this learning scope.

### Multi-Broker Kafka Cluster
Production Kafka deployments use 3+ brokers for fault tolerance and replication. This project runs a single broker, which is sufficient for learning and free-tier budget constraints.

### Multiple Consumer Groups
We run one consumer group (`notification-service`). In a real system, multiple teams might have independent consumer groups reading the same topic for different purposes (analytics, audit logs, etc.).

---

## Local Development

```bash
# Clone the repo
git clone https://github.com/yourname/kafka-notify.git
cd kafka-notify

# Copy environment variables
cp .env.example .env

# Start all local services
docker-compose up -d

# Kafka UI available at
http://localhost:8080

# Producer API Swagger docs available at
http://localhost:8001/docs
```

---

## Project Structure

```
kafka-notify/
├── producer-service/
│   ├── app/
│   │   ├── main.py
│   │   ├── routes/
│   │   ├── models/
│   │   ├── kafka/
│   │   └── config.py
│   ├── tests/
│   ├── Dockerfile
│   └── requirements.txt
│
├── consumer-service/
│   ├── app/
│   │   ├── main.py
│   │   ├── handlers/
│   │   ├── models/
│   │   ├── kafka/
│   │   ├── notifications/
│   │   └── config.py
│   ├── tests/
│   ├── Dockerfile
│   └── requirements.txt
│
├── k8s/
│   ├── producer/
│   ├── consumer/
│   ├── kafka/
│   └── kafka-ui/
│
├── docker/
│   └── docker-compose.yml
│
├── tests/
│   └── integration/
│
└── .github/
    └── workflows/
        └── ci-cd.yml
```
