# Webhook Ingestion Service

A high-performance, idempotent Go service designed to receive call-completion webhooks from our telephony provider, store them securely in Postgres, and maintain real-time, per-account call statistics.

## Overview

This service exposes a reliable webhook endpoint that ingests call events and processes them asynchronously. It is built with robustness in mind, ensuring that:
- **Idempotency:** Webhook redeliveries do not duplicate records or inflate call counts.
- **Asynchronous Processing:** Long-running tasks (like fetching recordings) are processed in the background without blocking the API response.
- **Graceful Shutdown:** In-flight processing jobs are safely completed during deployments or server termination.

## Technologies Used

- **Language:** Go 1.25+
- **Database:** PostgreSQL (for durable storage of events, calls, and stats)
- **Cache / PubSub:** Redis
- **Infrastructure:** Docker & Docker Compose

## Getting Started

You only need Docker and Go 1.25+ installed to run the service locally.

### Running Locally

```bash
# Spin up Postgres, Redis, and the Go API service
docker compose up -d --build

# Verify the service is healthy
curl localhost:8080/healthz
```

### Testing

The service includes a comprehensive automated test suite. To run it against the Dockerized Postgres and Redis instances:

```bash
go test -v ./...
```

If you ever need to wipe the database and start completely fresh, use:
```bash
make reset
```

## API Documentation

### Ingest Webhook
**`POST /webhooks/calls`**

Receives a call completion event.

**Payload:**
```json
{
  "event_id":      "evt_01H8XK2M9P",
  "call_id":       "call_9f2ab31c",
  "account_id":    "acc_123",
  "status":        "completed",
  "duration_sec":  143,
  "recording_url": "https://recordings.example.com/9f2ab31c.wav",
  "occurred_at":   "2026-08-13T09:12:00Z"
}
```

### Get Account Statistics
**`GET /accounts/{account_id}/stats`**

Retrieves the current call count and total duration in seconds for a specific account ID.
