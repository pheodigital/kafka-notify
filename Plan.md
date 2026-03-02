# Plan — kafka-notify

## Development Philosophy

Each PR is independently reviewable, independently deployable (in local dev), and leaves the system in a working state. No PR should depend on unreviewable code from a future PR. Tests are written alongside code, not after.

---

## PR Overview

| PR | Title | Depends On | Deliverable |
|---|---|---|---|
| PR 1 | Local Dev Foundation | None | `docker-compose up` → Kafka running locally |
| PR 2 | Producer Service | PR 1 | Publish events to Kafka via REST API |
| PR 3 | Consumer Service | PR 1, PR 2 | Consume events, send emails via SendGrid |
| PR 4 | Kubernetes on DigitalOcean | PR 2, PR 3 | Full system live in cloud |
| PR 5 | CI/CD Pipeline | PR 4 | Auto-deploy on push to main |

---

## PR 1 — Local Dev Foundation

**Goal:** Any developer clones the repo, runs one command, and has a working Kafka environment locally with a visual UI.

**Branch:** `feat/local-dev-foundation`

### Files Created

```
docker/
└── docker-compose.yml     ← Kafka, Zookeeper, Kafka UI, producer, consumer

.env.example               ← All required environment variables documented
.gitignore                 ← Python, Docker, IDE ignores
README.md                  ← Setup instructions
```

### docker-compose.yml Services

| Service | Image | Port | Purpose |
|---|---|---|---|
| zookeeper | confluentinc/cp-zookeeper | 2181 | Kafka coordination |
| kafka | confluentinc/cp-kafka | 9092 | Message broker |
| kafka-ui | provectuslabs/kafka-ui | 8080 | Visual topic browser |
| producer | (local build) | 8001 | Producer service |
| consumer | (local build) | 8002 | Consumer service |

### Environment Variables (.env.example)

```
# Kafka
KAFKA_BOOTSTRAP_SERVERS=kafka:9092
KAFKA_TOPIC=app.events
KAFKA_DLQ_TOPIC=app.events.dlq
KAFKA_CONSUMER_GROUP=notification-service

# SendGrid
SENDGRID_API_KEY=your_sendgrid_api_key_here
SENDGRID_FROM_EMAIL=your_verified_sender@example.com

# App
ENVIRONMENT=development
```

### Acceptance Criteria

- [ ] `docker-compose up -d` starts all services without errors
- [ ] Kafka UI is accessible at `http://localhost:8080`
- [ ] `app.events` and `app.events.dlq` topics are visible in Kafka UI
- [ ] `docker-compose down` cleanly stops all services
- [ ] `.env.example` documents every required variable

### What This PR Does NOT Include

- No application code (producer/consumer services are placeholder containers in compose)
- No Kubernetes manifests
- No CI/CD

---

## PR 2 — Producer Service

**Goal:** A working FastAPI service that validates incoming requests and publishes structured events to Kafka. Testable end-to-end using Swagger UI.

**Branch:** `feat/producer-service`

**Depends On:** PR 1 merged

### Files Created

```
producer-service/
├── app/
│   ├── main.py              ← FastAPI app entry point
│   ├── config.py            ← Settings loaded from environment
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── events.py        ← All 3 event endpoints
│   │   └── health.py        ← GET /health
│   ├── models/
│   │   ├── __init__.py
│   │   ├── base.py          ← EventEnvelope (shared wrapper)
│   │   ├── user.py          ← UserSignupPayload
│   │   ├── order.py         ← OrderPlacedPayload
│   │   └── payment.py       ← PaymentFailedPayload
│   └── kafka/
│       ├── __init__.py
│       └── producer.py      ← aiokafka producer client
├── tests/
│   ├── __init__.py
│   ├── conftest.py          ← Pytest fixtures (mock Kafka producer)
│   ├── test_routes.py       ← Endpoint tests
│   ├── test_models.py       ← Schema validation tests
│   └── test_producer.py     ← Kafka publish tests
├── Dockerfile
└── requirements.txt
```

### API Endpoints

| Method | Path | Request Body | Response |
|---|---|---|---|
| POST | `/events/user/signup` | `UserSignupPayload` | `202 Accepted` |
| POST | `/events/order/placed` | `OrderPlacedPayload` | `202 Accepted` |
| POST | `/events/payment/failed` | `PaymentFailedPayload` | `202 Accepted` |
| GET | `/health` | None | `200 {"status": "ok"}` |

**Why 202 and not 200?**
202 Accepted means "I have accepted your request and will process it asynchronously." This is the semantically correct HTTP status for async event publishing. 200 OK implies synchronous completion.

### Key Implementation Details

**aiokafka** is the async Kafka client for Python. It integrates cleanly with FastAPI's async lifecycle. The producer is initialized on app startup and closed on app shutdown using FastAPI's `lifespan` context manager.

**Pydantic models** enforce required fields, data types, and value constraints before anything reaches Kafka. A malformed payload returns 422 Unprocessable Entity with a descriptive error — Kafka is never touched.

**Event enrichment** in the route layer (not the model layer): `event_id` (UUID4), `timestamp` (UTC ISO 8601), and `version` ("1.0") are added by the route handler before publishing. The caller does not need to provide these.

### Unit Tests

| Test | What It Verifies |
|---|---|
| `test_signup_valid_payload` | 202 returned, Kafka publish called once |
| `test_signup_missing_email` | 422 returned, Kafka publish NOT called |
| `test_signup_invalid_email_format` | 422 returned with field-level error |
| `test_order_valid_payload` | 202 returned, correct event_type in message |
| `test_order_empty_items_list` | 422 returned |
| `test_payment_valid_payload` | 202 returned |
| `test_event_envelope_structure` | event_id, timestamp, version present in published message |
| `test_kafka_publish_correct_topic` | Message published to `app.events` not DLQ |
| `test_health_endpoint` | 200 returned with status ok |

### Acceptance Criteria

- [ ] `docker-compose up` → `http://localhost:8001/docs` shows Swagger UI with 3 event endpoints
- [ ] POST valid payload to each endpoint → 202 response
- [ ] POST invalid payload → 422 with descriptive error
- [ ] Message appears in `app.events` topic in Kafka UI after valid POST
- [ ] All unit tests pass: `pytest producer-service/tests/`
- [ ] Dockerfile builds without errors: `docker build ./producer-service`

---

## PR 3 — Consumer Service

**Goal:** A long-running service that polls Kafka, routes events to handlers, sends emails via SendGrid, retries on failure, and publishes to DLQ after exhausting retries.

**Branch:** `feat/consumer-service`

**Depends On:** PR 1, PR 2 merged

### Files Created

```
consumer-service/
├── app/
│   ├── main.py                  ← FastAPI app + background consumer task
│   ├── config.py                ← Settings from environment
│   ├── models/
│   │   ├── __init__.py
│   │   ├── base.py              ← EventEnvelope (mirrors producer)
│   │   ├── user.py
│   │   ├── order.py
│   │   └── payment.py
│   ├── kafka/
│   │   ├── __init__.py
│   │   ├── consumer.py          ← aiokafka consumer, poll loop
│   │   ├── producer.py          ← DLQ publisher
│   │   └── router.py            ← event_type → handler mapping
│   ├── handlers/
│   │   ├── __init__.py
│   │   ├── base.py              ← BaseHandler with retry logic
│   │   ├── user_signup.py       ← Handles user.signup events
│   │   ├── order_placed.py      ← Handles order.placed events
│   │   └── payment_failed.py    ← Handles payment.failed events
│   └── notifications/
│       ├── __init__.py
│       └── sendgrid.py          ← SendGrid API wrapper
├── tests/
│   ├── __init__.py
│   ├── conftest.py              ← Fixtures (mock SendGrid, mock Kafka)
│   ├── test_router.py           ← Event routing tests
│   ├── test_handlers.py         ← Handler logic tests
│   ├── test_retry.py            ← Retry and DLQ tests
│   └── test_sendgrid.py         ← SendGrid payload construction tests
├── Dockerfile
└── requirements.txt
```

### Consumer Poll Loop (Simplified Flow)

```python
async for message in consumer:
    event = deserialize(message)
    handler = router.get(event.event_type)
    await handler.execute_with_retry(event)
    await consumer.commit()
```

Offset is committed only after `execute_with_retry` completes (success or DLQ). This ensures no message is skipped.

### Handler Routing Table

```python
HANDLERS = {
    "user.signup":    UserSignupHandler,
    "order.placed":   OrderPlacedHandler,
    "payment.failed": PaymentFailedHandler,
}
```

Unknown event types log a warning and commit the offset (do not crash, do not retry).

### SendGrid Email Templates

| Event | Subject | Body Content |
|---|---|---|
| user.signup | Welcome to kafka-notify! | Greeting with user's full name |
| order.placed | Your order has been placed | Order ID, items, total amount |
| payment.failed | Payment issue on your account | Amount, failure reason, support CTA |

### Unit Tests

| Test | What It Verifies |
|---|---|
| `test_router_user_signup` | Routes to UserSignupHandler |
| `test_router_order_placed` | Routes to OrderPlacedHandler |
| `test_router_payment_failed` | Routes to PaymentFailedHandler |
| `test_router_unknown_type` | Logs warning, does not raise |
| `test_handler_success` | SendGrid called once, no retry |
| `test_handler_retry_on_failure` | SendGrid called 3 times on failure |
| `test_handler_dlq_after_retries` | DLQ publish called after 3 failures |
| `test_handler_offset_committed_on_success` | Offset committed after success |
| `test_handler_offset_committed_on_dlq` | Offset committed after DLQ publish |
| `test_sendgrid_user_signup_payload` | Correct recipient, subject, body |
| `test_sendgrid_order_placed_payload` | Order details present in body |
| `test_sendgrid_payment_failed_payload` | Failure reason present in body |

### Acceptance Criteria

- [ ] Consumer starts and connects to Kafka (visible in Kafka UI consumer groups tab)
- [ ] POST event via producer → email received in inbox within 5 seconds
- [ ] Kafka UI shows consumer group lag returns to 0 after processing
- [ ] Simulate SendGrid failure (wrong API key) → message appears in `app.events.dlq`
- [ ] Consumer pod restart → resumes from correct offset, no duplicate emails
- [ ] All unit tests pass: `pytest consumer-service/tests/`
- [ ] Dockerfile builds without errors

---

## PR 4 — Kubernetes on DigitalOcean

**Goal:** Deploy the full system to a real managed Kubernetes cluster. Every service running as a pod. Producer accessible via public IP.

**Branch:** `feat/kubernetes-deployment`

**Depends On:** PR 2, PR 3 merged and Docker images pushed to DockerHub

### Files Created

```
k8s/
├── kafka/
│   └── values.yaml              ← Bitnami Kafka Helm chart values
├── kafka-ui/
│   ├── deployment.yaml
│   └── service.yaml             ← LoadBalancer (public access to Kafka UI)
├── producer/
│   ├── deployment.yaml
│   ├── service.yaml             ← LoadBalancer (public access to producer API)
│   └── configmap.yaml
├── consumer/
│   ├── deployment.yaml
│   ├── service.yaml             ← ClusterIP (internal only)
│   └── configmap.yaml
└── secrets/
    └── app-secrets.yaml.example ← Template only — real secrets never committed
```

### Cluster Setup Steps

```bash
# 1. Create cluster
doctl kubernetes cluster create kafka-notify \
  --region fra1 \
  --node-pool "name=worker-pool;size=s-2vcpu-4gb;count=2"

# 2. Save kubeconfig
doctl kubernetes cluster kubeconfig save kafka-notify

# 3. Verify
kubectl get nodes

# 4. Install Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 5. Deploy Kafka
kubectl create namespace kafka
helm install kafka bitnami/kafka \
  --namespace kafka \
  -f k8s/kafka/values.yaml

# 6. Create app namespace and secrets
kubectl create namespace app
kubectl create secret generic app-secrets \
  --from-literal=SENDGRID_API_KEY=your_key \
  --namespace app

# 7. Deploy application services
kubectl apply -f k8s/producer/ -n app
kubectl apply -f k8s/consumer/ -n app
kubectl apply -f k8s/kafka-ui/ -n kafka
```

### Kubernetes Resource Configuration

| Resource | Replicas | CPU Request | Memory Request |
|---|---|---|---|
| Producer | 1 | 100m | 128Mi |
| Consumer | 1 | 100m | 256Mi |
| Kafka UI | 1 | 100m | 256Mi |

Note: Requests are conservative for free-tier / credit budget management.

### Health Checks

Both producer and consumer Deployments configure:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 800x
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 800x
  initialDelaySeconds: 10
  periodSeconds: 5
```

This ensures K8s does not route traffic to a pod that hasn't finished connecting to Kafka.

### Acceptance Criteria

- [ ] `kubectl get pods -A` shows all pods in Running state
- [ ] Producer public IP accessible: `curl http://<EXTERNAL_IP>/health` returns 200
- [ ] POST event to producer public IP → email received in inbox
- [ ] Kafka UI accessible via public IP → topics and consumer group visible
- [ ] `kubectl delete pod <producer-pod>` → K8s automatically restarts it
- [ ] Consumer offset resumes correctly after pod restart

---

## PR 5 — CI/CD Pipeline

**Goal:** Push to main → tests run → images built and pushed → cluster updated automatically. Zero manual deployment steps.

**Branch:** `feat/cicd-pipeline`

**Depends On:** PR 4 merged

### Files Created

```
.github/
└── workflows/
    └── ci-cd.yml              ← Full pipeline definition

tests/
└── integration/
    ├── __init__.py
    └── test_end_to_end.py     ← Fire event → verify Kafka message received
```

### Pipeline Stages

```yaml
on:
  push:
    branches: [main]

jobs:
  test:
    - pytest producer-service/tests/
    - pytest consumer-service/tests/

  build-and-push:
    needs: test
    - docker build producer-service → yourusername/producer-service:$SHA
    - docker build consumer-service → yourusername/consumer-service:$SHA
    - docker push both images

  deploy:
    needs: build-and-push
    - doctl kubernetes cluster kubeconfig save kafka-notify
    - kubectl set image deployment/producer producer=yourusername/producer-service:$SHA
    - kubectl set image deployment/consumer consumer=yourusername/consumer-service:$SHA
    - kubectl rollout status deployment/producer
    - kubectl rollout status deployment/consumer

  integration-test:
    needs: deploy
    - POST test event to producer public IP
    - Assert 202 response
    - Assert message appears in Kafka (via Kafka REST proxy or direct check)
```

### GitHub Actions Secrets Required

| Secret Name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your DockerHub username |
| `DOCKERHUB_TOKEN` | DockerHub access token |
| `DIGITALOCEAN_ACCESS_TOKEN` | DigitalOcean API token |
| `SENDGRID_API_KEY` | SendGrid API key |
| `PRODUCER_URL` | Public IP of producer LoadBalancer |

### Acceptance Criteria

- [ ] Push to main → GitHub Actions pipeline triggers automatically
- [ ] Test failure in PR 2/3 code → pipeline stops, no deploy
- [ ] Successful push → new image on DockerHub with correct git SHA tag
- [ ] `kubectl rollout status` confirms new pods are live
- [ ] Integration test job passes in pipeline
- [ ] No secrets are visible in pipeline logs

---

## Definition of Done (All PRs)

A PR is considered complete when:

1. All acceptance criteria above are checked off
2. All unit tests pass locally and in CI
3. Code is reviewed (self-review minimum — read your own diff)
4. No secrets, API keys, or `.env` files are committed
5. README is updated if setup steps change
6. Branch is merged via squash commit with a descriptive message

---

## Milestone Summary

| Milestone | PRs | Outcome |
|---|---|---|
| Local Environment | PR 1 | Kafka running locally, Kafka UI browsable |
| Core Services | PR 2 + PR 3 | Full event flow working locally end-to-end |
| Cloud Deployment | PR 4 | System live on real Kubernetes cluster |
| Production Readiness | PR 5 | Automated testing and deployment on every push |
