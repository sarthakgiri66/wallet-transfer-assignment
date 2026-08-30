# Wallet Transfer Service

A production-quality wallet-to-wallet transfer service implementing idempotent,
atomic, double-entry ledger transfers with safe concurrent execution.

## Prerequisites

- Go 1.22+
- Docker + Docker Compose (or a local Postgres 16 instance)

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `postgres://postgres:postgres@localhost:5432/wallet_transfer?sslmode=disable` | Postgres DSN |
| `MIGRATIONS_DIR` | `migrations` | Path to SQL migration files |
| `ADDR` | `:8080` | HTTP listen address |
| `TEST_DATABASE_URL` | `postgres://...wallet_transfer_test...` | DSN for the test database |

## Local Setup

```bash
# 1. Start Postgres
docker compose up -d

# 2. Run the server (migrations applied automatically on startup)
go run ./cmd/server/

# 3. Verify
curl http://localhost:8080/healthz
```

## Running Tests

```bash
# Unit tests (no database required)
go test ./internal/...

# Integration tests (requires a running Postgres)
go test -race -count=1 -timeout=60s ./tests/integration/...

# All tests with race detector
go test -race -count=1 -timeout=60s ./...
```

## Example Requests

### Create wallets

```bash
ALICE=$(curl -s -X POST http://localhost:8080/wallets \
  -H 'Content-Type: application/json' \
  -d '{"name":"Alice","initialBalance":1000}' | grep -o '"id":"[^"]*"' | cut -d'"' -f4)

BOB=$(curl -s -X POST http://localhost:8080/wallets \
  -H 'Content-Type: application/json' \
  -d '{"name":"Bob","initialBalance":500}' | grep -o '"id":"[^"]*"' | cut -d'"' -f4)
```

### Transfer funds

```bash
curl -X POST http://localhost:8080/transfers \
  -H 'Content-Type: application/json' \
  -d "{
    \"idempotencyKey\": \"txn-001\",
    \"fromWalletId\": \"$ALICE\",
    \"toWalletId\": \"$BOB\",
    \"amount\": 100
  }"
# → {"transferId":"...","status":"PROCESSED"}
```

### Idempotent retry (same result, no re-execution)

```bash
curl -X POST http://localhost:8080/transfers \
  -H 'Content-Type: application/json' \
  -d "{
    \"idempotencyKey\": \"txn-001\",
    \"fromWalletId\": \"$ALICE\",
    \"toWalletId\": \"$BOB\",
    \"amount\": 100
  }"
# → same {"transferId":"...","status":"PROCESSED"}
```

### Check balance

```bash
curl http://localhost:8080/wallets/$ALICE
# → {"id":"...","name":"Alice","balance":"900.00"}
```

## API

| Method | Path | Description |
|---|---|---|
| `POST` | `/transfers` | Create (or replay) a transfer |
| `POST` | `/wallets` | Create a wallet |
| `GET` | `/wallets/{id}` | Get wallet balance |
| `GET` | `/healthz` | Health check |

### Error codes

| Code | HTTP Status | Meaning |
|---|---|---|
| `INVALID_REQUEST` | 400 | Malformed JSON, negative/zero amount, same wallet |
| `WALLET_NOT_FOUND` | 404 | Source or destination wallet doesn't exist |
| `INSUFFICIENT_FUNDS` | 409 | Source balance too low |
| `IDEMPOTENCY_CONFLICT` | 409 | Key reused with different parameters |
| `INTERNAL_ERROR` | 500 | Unexpected server/database error |
