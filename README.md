# micro-one

A hands-on **Python microservices playground** demonstrating the three most common ways services talk to each other:

- **REST** over HTTP (FastAPI)
- **gRPC** for fast synchronous service-to-service calls
- **Asynchronous messaging** with RabbitMQ (including a Dead Letter Queue for failure handling)

Everything is containerized with Docker Compose, fronted by a **Traefik** reverse proxy, and managed as a single [uv](https://github.com/astral-sh/uv) workspace.

---

## Architecture

```
                          ┌─────────────┐
                          │   Traefik   │  :80  (proxy)  /  :8080 (dashboard)
                          └──────┬──────┘
                                 │  /users, /docs, /openapi.json
                                 ▼
        ┌────────────────────────────────────────┐
        │            user-service (REST)          │
        │   FastAPI · JWT auth · SQLModel          │
        └───────┬───────────────────────┬─────────┘
                │ gRPC                   │ SQL
                ▼                        ▼
        ┌───────────────┐        ┌───────────────┐
        │ product-svc   │        │  Postgres db  │
        │  gRPC :50051  │        └───────────────┘
        └───────────────┘

        ┌───────────────┐   publish   ┌───────────────┐   consume   ┌──────────────────┐
        │ order-service │ ──────────► │   RabbitMQ    │ ──────────► │ inventory-service│
        │  REST :8002   │  order_     │ :5672 / :15672│  order_     │   REST :8003     │
        └───────────────┘  events     └───────────────┘  events     └──────────────────┘
                                              │ reject (no requeue)
                                              ▼
                                       Dead Letter Queue
                                       (order_events_dlq)
```

---

## Services

| Service             | Protocol        | Exposed Port            | Responsibility                                                                 |
| ------------------- | --------------- | ----------------------- | ------------------------------------------------------------------------------ |
| **traefik**         | HTTP proxy      | `80`, dashboard `8080`  | Routes incoming traffic to services via Docker labels                          |
| **user-service**    | REST (FastAPI)  | via Traefik (`/users`)  | User signup/login, JWT issuing & verification, fetches product info over gRPC  |
| **product-service** | gRPC            | `50051` (internal only) | Returns product details on request                                             |
| **order-service**   | REST (FastAPI)  | `8002`                  | Accepts orders and publishes `OrderPlaced` events to RabbitMQ                  |
| **inventory-service** | REST + consumer | `8003`                | Consumes order events, decrements stock, dead-letters failed messages         |
| **db**              | PostgreSQL 16   | `5433` → `5432`         | Persistent store for the user-service                                          |
| **rabbitmq**        | AMQP            | `5672`, mgmt `15672`    | Message broker between order and inventory services                            |

---

## Tech Stack

- **Python 3.13**
- **FastAPI** — REST APIs
- **gRPC** (`grpcio`, `grpcio-tools`) — synchronous service-to-service calls
- **RabbitMQ** + **aio-pika** — asynchronous event messaging with Dead Letter Exchange
- **SQLModel** + **PostgreSQL** (`psycopg`) — persistence
- **PyJWT** + **passlib[bcrypt]** — authentication
- **Traefik v3** — reverse proxy / API gateway
- **uv** — dependency & workspace management
- **Docker Compose** — orchestration

---

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) & Docker Compose
- (Optional, for local dev) [uv](https://github.com/astral-sh/uv)

### Run everything

```bash
docker compose up --build
```

Once the stack is healthy:

| What                          | URL                                              |
| ----------------------------- | ------------------------------------------------ |
| User Service API docs (Swagger) | http://localhost/docs                          |
| Order Service                 | http://localhost:8002/docs                        |
| Inventory Service             | http://localhost:8003/docs                        |
| Traefik dashboard             | http://localhost:8080                             |
| RabbitMQ management UI        | http://localhost:15672 (user: `guest` / `guest`) |

---

## Example Usage

### 1. Create a user

```bash
curl -X POST http://localhost/users/ \
  -H "Content-Type: application/json" \
  -d '{"name": "Ada", "email": "ada@example.com", "password": "secret"}'
```

### 2. Log in to get a JWT

```bash
curl -X POST http://localhost/login \
  -H "Content-Type: application/json" \
  -d '{"name": "Ada", "email": "ada@example.com", "password": "secret"}'
# → { "access_token": "...", "token_type": "bearer" }
```

### 3. Fetch a user's purchase (protected — triggers a gRPC call to product-service)

```bash
curl http://localhost/users/1/purchases/42 \
  -H "Authorization: Bearer <access_token>"
```

The JWT's `sub` must match the requested `user_id`, otherwise the request is rejected with `403`.

### 4. Place an order (publishes an async event)

```bash
curl -X POST "http://localhost:8002/orders/?product_id=99&user_id=1"
# → { "message": "Order received and is being processed in the background." }
```

The **inventory-service** consumes the event and decrements stock. Check it:

```bash
curl http://localhost:8003/inventory/99
```

> 💡 Sending `product_id=999` simulates a fatal error: the message is rejected without requeue and routed to the **Dead Letter Queue** (`order_events_dlq`), visible in the RabbitMQ management UI.

---

## Project Structure

```
micro_one/
├── compose.yml                 # Docker Compose stack
├── pyproject.toml              # uv workspace root
├── protos/
│   └── product.proto           # gRPC contract for the product service
├── shared/
│   └── jwt_utils.py            # Shared JWT verification helper
└── services/
    ├── user-service/           # FastAPI · Postgres · JWT · gRPC client
    ├── product-service/        # gRPC server
    ├── order-service/          # FastAPI · RabbitMQ publisher
    └── inventory-service/      # FastAPI · RabbitMQ consumer + DLQ
```

---

## Concepts Demonstrated

- **API Gateway routing** with Traefik using Docker labels (no manual config files).
- **Synchronous inter-service calls** via gRPC and Protocol Buffers.
- **Asynchronous, event-driven communication** with RabbitMQ — orders are accepted instantly and processed in the background.
- **Resilient consumers** using a Dead Letter Exchange (DLX) / Dead Letter Queue (DLQ) to isolate poison messages.
- **JWT-based authentication** with a secret shared across services.
- **Multi-stage Docker builds** with uv for small, reproducible images.
- **uv workspaces** to manage multiple services with isolated dependencies in one repo.

---

## Notes & Caveats

This is a learning/demo project, not production-ready. Notably:

- The JWT `SECRET_KEY` is hard-coded — load it from an environment variable in real deployments.
- The inventory store is an in-memory dict, so stock resets on restart.
- CORS is wide open (`allow_origins=["*"]`).
- `order-service` and `inventory-service` ports are exposed directly for easy testing.
