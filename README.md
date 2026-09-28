# Order Processor Service

Event-driven consumer service built with Go that processes order events from RabbitMQ, executes business logic, updates inventory in PostgreSQL, and emits downstream events. Implements clean architecture with strict layer boundaries for maintainability and testability.

---

## Table of Contents

- [Architecture](#architecture)
  - [Overview](#overview)
  - [Layer Boundaries and Rules](#layer-boundaries-and-rules)
  - [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Environment Variables](#environment-variables)
- [Running Locally](#running-locally)
- [Running Tests](#running-tests)
- [Docker](#docker)
- [Event Schema](#event-schema)
- [Known Issues & Gotchas](#known-issues--gotchas)
- [License](#license)

---

## Architecture

### Overview

This service follows the **Clean Architecture** pattern adapted for event-driven consumers. Incoming AMQP messages pass strictly through layers from message handling down to persistence:

```
RabbitMQ
  ↓
Consumer Handler (Decode payload, invoke service)
  ↓
Service (Business rules, entity ↔ DTO mapping, emit events)
  ↓
Repository (Data access with GORM/SQLx, returns entities and error sentinels)
  ↓
PostgreSQL Database
```

### Layer Boundaries and Rules

| Layer | Package | Depends on | Must NOT contain |
|---|---|---|---|
| **Consumer** | `consumer/` | `service`, `payload`, `helper` | SQL queries, business logic, direct DB access |
| **Business** | `service/` | `repository`, `payload`, `entities`, `messaging` | AMQP-specific code, direct SQL queries |
| **Persistence** | `repository/` | `entities` | DTOs, AMQP concepts, validation logic |
| **DTOs** | `payload/` | `entities` (mappers only) | Database logic, messaging protocol details |
| **Messaging** | `messaging/` | Standard library, AMQP driver | Core business/domain logic |
| **Shared** | `helper/`, `config/` | Standard library, drivers | Core business/domain logic |

### Project Structure

```
.
├── main.go                     # Application entry point, dependency wiring
├── config/
│   └── config.go               # Configuration loader, DB pool, RabbitMQ setup
├── consumer/                   # AMQP message handlers (decode, invoke, ack/nack)
│   └── OrderConsumer.go
├── service/                    # Business logic, inventory checks, event emission
│   └── OrderProcessorService.go
├── repository/                 # Data persistence layer (GORM/SQLx)
│   └── OrderRepository.go
├── entities/                   # Domain models for database
│   └── OrderEntity.go
├── payload/                    # Event DTOs and mappers
│   └── OrderEventPayload.go
├── messaging/                  # RabbitMQ publisher & consumer abstractions
│   ├── publisher.go
│   └── consumer.go
├── helper/                     # Shared utilities: logging, error handling
│   └── logger.go
├── test/                       # Unit and integration tests
│   ├── consumer/
│   ├── service/
│   └── repository/
├── .env.example                # Sample environment variables
├── go.mod                      # Go module definition
├── go.sum                      # Go module checksums
├── Dockerfile                  # Multi-stage Docker build
└── README.md                   # Project documentation
```

---

## Requirements

- **Go**: 1.26 or higher
- **PostgreSQL**: 16 or higher
- **RabbitMQ**: 3.13 or higher
- **Git**: For repository cloning
- *(Optional)* **Docker & Docker Compose**: For containerized infrastructure

---

## Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/jwjooth/order-processor.git
   cd order-processor
   ```

2. **Download dependencies:**
   ```bash
   go mod tidy
   ```

3. **Verify installation:**
   ```bash
   go version
   ```

---

## Environment Variables

Copy `.env.example` to `.env` and update values for your environment:

```bash
cp .env.example .env
```

### Available Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `APP_NAME` | `string` | `order-processor` | Service identifier for logging/tracing |
| `LOG_LEVEL` | `string` | `info` | Log verbosity: `debug`, `info`, `warn`, `error` |
| `DB_HOST` | `string` | `localhost` | PostgreSQL host address |
| `DB_PORT` | `int` | `5432` | PostgreSQL port number |
| `DB_USER` | `string` | `postgres` | PostgreSQL username |
| `DB_PASSWORD` | `string` | `secret` | PostgreSQL password |
| `DB_NAME` | `string` | `orders` | PostgreSQL database name |
| `DB_POOL_SIZE` | `int` | `10` | Connection pool size |
| `RABBITMQ_URL` | `string` | `amqp://guest:guest@localhost:5672/` | RabbitMQ connection URI |
| `ORDER_QUEUE` | `string` | `order.created` | Input queue name |
| `PROCESSED_EXCHANGE` | `string` | `order.events` | Outbound event exchange |
| `PROCESSED_ROUTING_KEY` | `string` | `order.processed` | Routing key for success events |
| `FAILED_ROUTING_KEY` | `string` | `order.failed` | Routing key for failure events |
| `DLQ_ENABLED` | `bool` | `true` | Enable dead letter queue for retries |
| `MAX_RETRIES` | `int` | `3` | Maximum retry attempts |

---

## Running Locally

### Prerequisites

Ensure PostgreSQL and RabbitMQ are running and accessible with credentials from `.env`:

```bash
# Start PostgreSQL (if using docker)
docker run -d --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=orders \
  -p 5432:5432 \
  postgres:16-alpine

# Start RabbitMQ (if using docker)
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3.13-management-alpine
```

### Run the Application

1. **Start the service:**
   ```bash
   go run .
   ```

2. **Expected output:**
   ```
   2026/01/15 10:23:45 [INFO] order-processor started, consuming from order.created
   2026/01/15 10:23:45 [INFO] RabbitMQ connected, listening for messages
   ```

3. **Verify consumer is active** via RabbitMQ Management UI:
   - Navigate to http://localhost:15672 (default: guest/guest)
   - Go to **Queues** tab
   - Confirm `order.created` shows active consumer connection

4. **Build a local binary:**
   ```bash
   go vet ./...
   go build -ldflags="-w -s" -o order-processor .
   ./order-processor
   ```

---

## Running Tests

```bash
# Run all tests with coverage
go test -v -cover ./...

# Run only fast unit tests
go test -short ./...

# Run a specific package
go test -v ./service

# Generate coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

---

## Docker

### Build and Run

1. **Build the Docker image:**
   ```bash
   docker build -t order-processor:latest .
   ```

2. **Run with environment file:**
   ```bash
   docker run --env-file .env \
     --network host \
     order-processor:latest
   ```

3. **Or use Docker Compose (see `docker-compose.yml` in monorepo root):**
   ```bash
   docker-compose up order-processor
   ```

### Multi-Stage Build Strategy

The provided `Dockerfile` uses a two-stage build:
- **Stage 1 (Builder)**: Compiles Go binary with minimal dependencies
- **Stage 2 (Runtime)**: Lean Alpine image with only runtime requirements

Result: Final image ~15–20 MB (vs. ~400 MB with full Go toolchain).

---

## Event Schema

### Consumed Events

Subscribed to queue: **`order.created`**

| Field | Type | Required | Example |
|---|---|---|---|
| `order_id` | `string` (UUID) | ✓ | `550e8400-e29b-41d4-a716-446655440000` |
| `customer_id` | `string` (UUID) | ✓ | `123e4567-e89b-12d3-a456-426614174000` |
| `items` | `array<OrderItem>` | ✓ | `[{"sku": "ABC123", "qty": 2}]` |
| `total_amount` | `float64` | ✓ | `299.99` |
| `idempotency_key` | `string` | ✓ | `req-2026-01-15-001` |

### Published Events

**Exchange**: `order.events`

#### Success Event

- **Routing Key**: `order.processed`
- **Payload**:
  ```json
  {
    "order_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "processed",
    "inventory_updated_at": "2026-01-15T10:23:45Z",
    "downstream_services": ["notification-svc", "analytics"]
  }
  ```

#### Failure Event

- **Routing Key**: `order.failed`
- **Payload**:
  ```json
  {
    "order_id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "failed",
    "error_code": "INSUFFICIENT_INVENTORY",
    "error_message": "SKU ABC123 out of stock",
    "failed_at": "2026-01-15T10:23:46Z"
  }
  ```

---

## Known Issues & Gotchas

### 1. Idempotency Key Uniqueness Constraint
**Severity**: HIGH | **Status**: Must fix before production

The idempotency check relies on a unique index on `orders.idempotency_key`. Without this, duplicate messages will create duplicate orders.

**Action**:
```sql
CREATE UNIQUE INDEX idx_orders_idempotency_key 
  ON orders(idempotency_key);
```

Verify this constraint exists on startup or use automated migrations.

### 2. Hardcoded DLQ Retry Limit
**Severity**: MEDIUM | **Status**: Config pending

`consumer/OrderConsumer.go:45` hardcodes max retries to `3`. This should be configurable via `MAX_RETRIES` environment variable before scaling to production.

**Workaround**: Manually update constant or add flag.

### 3. GORM AutoMigrate on Startup
**Severity**: MEDIUM | **Status**: Design debt

`config.go` calls `AutoMigrate()` on every startup. While safe for development, production deployments should use versioned migrations (e.g., `golang-migrate/migrate`) to avoid schema conflicts during multi-instance rollouts.

**Migration Path**:
- Generate `.sql` migration files from current schema
- Remove `AutoMigrate()` from `config.go`
- Run migrations in CI/CD pipeline before deployment

### 4. No Distributed Tracing
**Severity**: LOW | **Status**: v2 roadmap

AMQP message headers don't propagate trace IDs to downstream services. This makes end-to-end debugging difficult in polyglot systems.

**Planned Solution**: OpenTelemetry span propagation via RabbitMQ headers (v2.0).

### 5. Synchronous Event Publishing
**Severity**: LOW | **Status**: Performance optimization pending

Outbound events are published synchronously within the message handler. Under sustained load (>1000 msg/s), this can cause message processing to stall if RabbitMQ becomes unavailable.

**Mitigation**: Implement async publisher with dedicated goroutine pool.

---

## License

MIT. See `LICENSE` file for details.

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit changes: `git commit -m "feat: description"`
4. Push to branch: `git push origin feature/your-feature`
5. Open a pull request

Ensure `go vet` and tests pass before submitting.
