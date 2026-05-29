# NestJS Microservices Demo

A production-style microservices demo built with **NestJS**, **RabbitMQ**, and **TypeORM**, showcasing distributed systems patterns including the **Transactional Outbox Pattern** and **Saga Pattern**.

## Architecture Overview

```
                   ┌─────────────┐
                   │  RabbitMQ   │
                   │  (Broker)   │
                   └──────┬──────┘
          ┌───────────────┼───────────────┐
          │               │               │
    ┌─────▼─────┐   ┌────▼────┐   ┌──────▼──────┐
    │   Users   │   │  Orders │   │  Payments   │
    │  Service  │   │ Service │   │  Service    │
    └─────┬─────┘   └────┬────┘   └──────┬──────┘
          │               │               │
    ┌─────▼─────┐        │               │
    │   Notif.  │        │               │
    │  Service  │        │               │
    └───────────┘        └───────────────┘
                                   ↑
                            ┌──────┴──────┐
                            │ API Gateway │◄──── Client (HTTP)
                            └─────────────┘
```

| Service | Role | Database | Queue |
|---|---|---|---|
| **api-gateway** | HTTP entry point, JWT auth, request routing | None | N/A (HTTP) |
| **users-service** | User CRUD, outbox event publishing | SQLite | `users_queue` |
| **orders-service** | Order lifecycle, Saga coordination | SQLite | `orders_queue` |
| **payments-service** | Payment processing (simulated gateway) | SQLite | `payments_queue` |
| **notification-service** | Async email notifications | None | `notifications_queue` |

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 |
| Language | TypeScript 5 |
| Framework | NestJS 10 |
| Message Broker | RabbitMQ (AMQP) |
| ORM | TypeORM 0.3 |
| Database | SQLite |
| Auth | JWT + Passport |
| Containerization | Docker / Compose |

## Key Patterns Implemented

### Transactional Outbox Pattern

Ensures reliable event publishing without dual-write problems. When a user is created, the user record **and** an outbox event are persisted in the same database transaction. A background relay polls for unprocessed outbox records and emits them to RabbitMQ, guaranteeing **at-least-once delivery**.

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  DB Transaction │     │  Outbox Relay    │     │  RabbitMQ        │
│                  │     │  (polls every 5s)│     │                  │
│  INSERT user    │────►│  SELECT unprocessed│────►│  emit("user.created")│
│  INSERT outbox  │     │  Mark as processed│     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

### Saga Pattern (Compensating Transactions)

The Orders Service coordinates a saga for order creation:

1. **Verify user** → call Users Service
2. **Create order** → status `PENDING`
3. **Process payment** → call Payments Service
4. **On success** → status `PAID`
5. **On failure** → **compensating action** → status `CANCELLED`

## Getting Started

### Prerequisites

- Node.js 20+
- RabbitMQ (or `docker compose up` to start one)
- npm

### Installation

```bash
npm install
```

### Run with Docker Compose

```bash
docker compose up
```

This starts RabbitMQ and all 5 services. The API gateway is available at `http://localhost:3000`.

### Run locally (separate terminals)

```bash
# Terminal 1 - Start RabbitMQ (if not running)
docker compose up rabbitmq redis

# Terminal 2+ - Start each service
npm run start:gateway
npm run start:users
npm run start:orders
npm run start:payments
npm run start:notification
```

### Environment Variables

Copy `.env.example` to `.env` and configure:

| Variable | Default | Description |
|---|---|---|
| `RABBITMQ_URL` | `amqp://guest:guest@localhost:5672` | RabbitMQ connection string |
| `MY_SUPER_SECRET_KEY` | — | JWT signing secret |

## API Endpoints

### Auth

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/login` | Login → returns JWT token |

### Users (requires JWT)

| Method | Path | Description |
|---|---|---|
| `GET` | `/users` | List all users |
| `POST` | `/users` | Create a user |
| `GET` | `/users/:id` | Get user by ID |
| `PUT` | `/users/:id` | Update user |
| `DELETE` | `/users/:id` | Deactivate user |

### Orders (requires JWT)

| Method | Path | Description |
|---|---|---|
| `POST` | `/orders` | Create an order |
| `GET` | `/orders/user/:userId` | Get orders by user |
| `GET` | `/orders/:id` | Get order by ID |
| `DELETE` | `/orders/:id/cancel` | Cancel an order |

## Project Structure

```
├── api-gateway/            # HTTP gateway, auth, request proxying
├── users-service/          # User CRUD + outbox pattern
├── orders-service/         # Order management + saga pattern
├── payments-service/       # Payment processing (simulated)
├── notification-service/   # Email notification handler
├── shared/                 # Shared message pattern constants
├── docker-compose.yml       # Service orchestration
├── Dockerfile              # Shared container image
└── package.json            # Monorepo root (single package)
```
