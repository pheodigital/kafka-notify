# kafka-notify

A production-style, event-driven notification service built with FastAPI, Apache Kafka, and deployed on Kubernetes via DigitalOcean.

---

## Final Confirmed Stack

| Layer | Technology | Why |
|---|---|---|
| API Framework | FastAPI (Python) | Async-native, modern, auto Swagger docs |
| Message Broker | Aiven Kafka (cloud) | Fully managed Kafka, free tier, zero local install |
| Notification Delivery | SendGrid | Free tier, reliable email delivery |
| Containerization | Docker | Industry standard (used in deployment PRs) |
| Orchestration | DigitalOcean Kubernetes (DOKS) | Managed K8s, $200 free credit |
| Image Registry | DockerHub | Free, simple, CI/CD friendly |
| Monitoring | Aiven Console | Built-in topic & message browser, no setup needed |
| CI/CD | GitHub Actions | Free, integrates with DockerHub + DOKS |
| Local Dev | GitHub Codespaces | Browser-based dev environment, no local install needed |
| Testing | Pytest | Python standard testing framework |
| Schema Validation | Pydantic (built into FastAPI) | Type-safe event models |

### Why Aiven Instead of a Self-Hosted Kafka Broker?

Aiven is fully managed Apache Kafka hosted in the cloud — you get a real Kafka endpoint with zero installation. Your FastAPI app connects to it exactly the same way it would connect to a local or self-hosted Kafka broker. The `aiokafka` library doesn't know the difference. This removes the need for Docker during development and lets you focus on learning FastAPI and Kafka rather than infrastructure setup.

> **Note:** Upstash discontinued their Kafka product in March 2025. Aiven is the recommended replacement — it offers a genuine free tier with no credit card required.

Free tier limits: 1 free service per organization. The service auto-sleeps after 24 hours of inactivity — simply click **Power On** in the Aiven Console to resume. No data loss on sleep.

### How It Works

Instead of running a Kafka broker on your machine or in Docker, you get a Kafka endpoint URL that your FastAPI app connects to over the internet. From your code's perspective — it's just Kafka. The `aiokafka` library doesn't know or care that it's Aiven vs a local broker.

```
Your Machine (just Python)
        │
        │  SSL / SASL
        ▼
┌─────────────────┐        ┌──────────────────────────┐
│  FastAPI App    │───────►│   Aiven Kafka (cloud)    │
│  (runs locally) │        │   Free tier              │
│                 │        │   Fully managed          │
└─────────────────┘        └──────────────────────────┘
                                      │
                                      ▼
                           ┌──────────────────────────┐
                           │  Consumer Service        │
                           │  (runs locally too)      │
                           └──────────────────────────┘
```

For the Kafka UI (visual browser) — Aiven has a **built-in console** in their dashboard where you can see topics, messages, and consumer group lag in real time. No Kafka UI container needed during development.

> **Important — SSL:** Unlike Upstash, Aiven requires SSL with a CA Certificate. When you create your Aiven service, download the `ca.crt` file from the **Overview** tab. This file is referenced in your `.env` and used by `aiokafka` to establish a secure connection. The file must never be committed to git.

---

## Accounts You Need — Set These Up First

Before writing any code, make sure the following accounts are ready. No local software installation is required.

### 1. GitHub
Used for source control, monorepo hosting, and GitHub Actions CI/CD pipelines. Also used to launch GitHub Codespaces — your browser-based development environment.

### 2. Aiven
Used as the Kafka broker during development. Fully managed, free tier, no credit card needed.

- Sign up at [console.aiven.io](https://console.aiven.io) — GitHub login works
- Click **Create Service** → Select **Apache Kafka** → Choose **Free** tier
  - Name: `kafka-notify`
  - Region: closest to you (e.g. `google-europe-west1`)
- Wait ~2 minutes for status to show **Running**
- Go to **Topics** tab and create two topics:
  - `app.events` — 3 partitions
  - `app.events.dlq` — 1 partition
- Go to the **Overview** tab and save your connection details:
  - **Service URI** (bootstrap server): `kafka-notify-xxxx.aivencloud.com:12345`
  - **Username**
  - **Password**
  - **CA Certificate** — click **Download** and save as `ca.crt`
- These go into your `.env` file — **never commit them to git**
- The `ca.crt` file goes in the root of your project — add it to `.gitignore`

### 3. SendGrid
Used for sending notification emails (welcome emails, order confirmations, payment failure alerts).

- Sign up at [sendgrid.com](https://sendgrid.com) — free tier allows 100 emails/day
- Navigate to **Settings → API Keys → Create API Key** (Full Access)
- Save the API key immediately — it is shown only once
- Navigate to **Settings → Sender Authentication** and verify your sender email address
  - This is the `from` address that appears on all outgoing notifications

### 4. DockerHub
Used as the container image registry. Not needed until PR 3.

- Sign up at [hub.docker.com](https://hub.docker.com)
- Note your **DockerHub username** — it prefixes every image name
  - Example: `yourusername/producer-service:latest`
- Create an **Access Token** under Account Settings → Security → New Access Token

### 5. DigitalOcean
Used for managed Kubernetes hosting. Not needed until PR 4.

- Sign up at [digitalocean.com](https://digitalocean.com) — $200 free credit for new accounts
- The Kubernetes cluster is created in PR 4, not during development

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
- `POST /events/user/signup` → message visible in Kafka topic via Aiven Console
- `POST /events/order/placed` → message visible in Kafka topic via Aiven Console
- `POST /events/payment/failed` → message visible in Kafka topic via Aiven Console
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

## Development Environment

This project requires **no local software installation**. Everything runs via GitHub Codespaces (browser-based VS Code) and Aiven (cloud Kafka).

### Why GitHub Codespaces?
- Full Linux environment with Python pre-installed, runs entirely in your browser
- No admin rights or local installation needed
- Ports are automatically forwarded — `localhost:8001/docs` works in your browser even though the code runs on a GitHub server
- Free tier: 120 core-hours/month (more than enough for this project)

### Getting Started

**Step 1 — Open in Codespaces**
```
GitHub Repo → Code button → Codespaces tab → Create codespace on main
```

**Step 2 — Verify Python is ready**
```bash
python3 --version    # Should be 3.10+
pip3 --version
```

**Step 3 — Copy environment variables and add your Aiven credentials**
```bash
cp .env.example .env
# Fill in your Aiven bootstrap server, username, password
# Copy your downloaded ca.crt into the project root
```

**Step 4 — Install dependencies and run Producer**
```bash
cd producer-service
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8001

# Swagger UI auto-opens at localhost:8001/docs (Codespaces forwards the port)
```

**Step 5 — Run Consumer (separate terminal)**
```bash
cd consumer-service
pip install -r requirements.txt
python -m app.main
```

### Monitoring Kafka
No Kafka UI container needed. Use the **Aiven Console** at [console.aiven.io](https://console.aiven.io):
- View topics and partition details
- Browse messages in real time
- Monitor consumer group lag
- Power the service back on if it has auto-slept

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
│   ├── Dockerfile          ← Added in PR 3
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
│   ├── Dockerfile          ← Added in PR 3
│   └── requirements.txt
│
├── k8s/                    ← Added in PR 4
│   ├── producer/
│   ├── consumer/
│   ├── kafka/
│   └── kafka-ui/
│
├── docker/                 ← Added in PR 3
│   └── docker-compose.yml
│
├── tests/
│   └── integration/
│
├── ca.crt                  ← Aiven SSL certificate (never commit to git)
├── .env.example
├── .gitignore
└── .github/
    └── workflows/
        └── ci-cd.yml       ← Added in PR 5
```

## PR Roadmap

| PR | Focus | Kafka | Docker | Cloud |
|---|---|---|---|---|
| PR 1 | Producer Service (FastAPI) | Aiven | ✗ | ✗ |
| PR 2 | Consumer Service + SendGrid | Aiven | ✗ | ✗ |
| PR 3 | Docker Compose (local) | Self-hosted | ✓ | ✗ |
| PR 4 | Kubernetes on DigitalOcean | Self-hosted | ✓ | ✓ |
| PR 5 | CI/CD Pipeline | Self-hosted | ✓ | ✓ |
