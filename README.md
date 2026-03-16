# microservices-shop

A sample Go microservices system for practising DevOps and software architecture.

## Services

| Service | Port | Role |
|---|---|---|
| **api-gateway** | 8080 | Single entry-point; reverse-proxies to downstream services + health aggregation |
| **order-service** | 8081 | Creates orders; orchestrates the Saga (reserve → deliver, with rollback) |
| **inventory-service** | 8082 | Manages product catalogue and stock reservations |
| **delivery-service** | 8083 | Registers and tracks shipments |

## Architecture patterns demonstrated

- **API Gateway** — single ingress, request-ID injection, structured logging
- **Saga (orchestration style)** — order-service drives the distributed transaction:
  1. Reserve stock in inventory
  2. Register delivery
  3. On step-2 failure: compensating transaction releases the reservation
- **In-memory stores with mutex** — easy to swap for Postgres/Redis later
- **Health checks** — every service exposes `GET /health`; the gateway aggregates them all

## Quick start

```bash
docker compose up --build
```

Wait for all health checks to pass, then hit the gateway on port 8080.

## API Reference

All requests go through `http://localhost:8080`.

### Browse products
```bash
curl http://localhost:8080/api/products | jq
```

### Place an order
```bash
curl -s -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer_id":      "cust-1",
    "customer_name":    "Alice",
    "delivery_address": "123 Main St, Amsterdam",
    "items": [
      {"product_id": "prod-1", "quantity": 1},
      {"product_id": "prod-2", "quantity": 2}
    ]
  }' | jq
```

A successful response looks like:
```json
{
  "id": "ord-00001",
  "status": "confirmed",
  "reservation_id": "res-00001",
  "delivery_id": "del-00001",
  ...
}
```

### Trigger the Saga rollback (order too much stock)
```bash
curl -s -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer_id":      "cust-2",
    "customer_name":    "Bob",
    "delivery_address": "99 Fail Lane",
    "items": [
      {"product_id": "prod-1", "quantity": 999}
    ]
  }' | jq
# → status: "failed", failure_reason: "insufficient stock ..."
```

### Check all orders
```bash
curl http://localhost:8080/api/orders | jq
```

### Get a specific order
```bash
curl http://localhost:8080/api/orders/ord-00001 | jq
```

### Update a delivery status
```bash
curl -s -X PATCH http://localhost:8080/api/deliveries/del-00001/status \
  -H "Content-Type: application/json" \
  -d '{"status": "in_transit"}' | jq
```

### Gateway health (checks all services)
```bash
curl http://localhost:8080/health | jq
```

## Running services individually (without Docker)

In four terminals:
```bash
cd inventory-service && go run .
cd delivery-service  && go run .
cd order-service     && go run .
cd api-gateway       && go run .
```

The services pick up their upstream URLs from environment variables
(defaults point to localhost).

## DevOps experiments to try

### 1. Kill a service mid-flight
Stop `delivery-service` with `docker compose stop delivery-service` and place
an order. Watch the order-service logs: the saga detects the failure and rolls
back the inventory reservation. Then restart the service.

### 2. Observe the gateway health endpoint
```bash
watch -n1 "curl -s http://localhost:8080/health | jq"
```
Stop individual services and watch the status turn `degraded`.

### 3. Add a Postgres backend
Replace the in-memory `Store` structs with `database/sql` + `pgx`. Each
service should own its own schema (separate databases or at least separate
schemas) — a key microservices principle.

### 4. Introduce a message queue
Replace the synchronous order-service saga with an async event-driven
approach: publish `OrderCreated` events to NATS or RabbitMQ, have inventory
and delivery consume them, and add an `OrderStatusUpdated` event that the
order-service listens for.

### 5. Add distributed tracing
Propagate the `X-Request-ID` header through all downstream calls. Then swap
it out for OpenTelemetry trace/span IDs and wire up Jaeger.

### 6. Write a Kubernetes deployment
Convert the docker-compose into Kubernetes manifests (Deployment + Service
for each microservice, an Ingress for the gateway). Practice rolling updates
and readiness probes.

### 7. Rate-limit the gateway
Add a token-bucket rate limiter in the gateway middleware. Try
`golang.org/x/time/rate`.

### 8. Add circuit breaking
Wrap the downstream HTTP calls in order-service with a circuit breaker
(e.g. `github.com/sony/gobreaker`) so that a slow inventory service fails
fast instead of blocking order processing.
