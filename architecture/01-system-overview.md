# System Overview

Andromeda is a ticketing and event management platform composed of four
independent frontend applications sharing a common backend infrastructure.
All communication between frontends and backend services is routed through
a single API Gateway.

---

## High-Level Architecture

![System Architecture](../diagrams/01-system-architecture.png)

---

## Frontends

Four independent applications, each with a specific role and audience.
They share no code between them — each is deployed and versioned independently.

| App        | Domain                               | Technology         | Audience              |
|------------|--------------------------------------|--------------------|-----------------------|
| User App   | andromeda.andrescortes.dev           | Blazor WASM PWA    | End users             |
| Admin App  | admin.andromeda.andrescortes.dev     | Laravel + Livewire | Admins / SuperAdmins  |
| POS App    | tickets.andromeda.andrescortes.dev   | Blazor WASM PWA    | Ticket operators      |
| Access App | access.andromeda.andrescortes.dev    | Blazor WASM PWA    | Door staff            |

---

## API Gateway — YARP

All frontend requests pass through YARP (Yet Another Reverse Proxy), a
.NET 9 native reverse proxy library. It handles cross-cutting concerns
before any request reaches a service.

Responsibilities:
- JWT RS256 validation via JWKS endpoint — services never validate tokens themselves
- Rate limiting backed by Redis counters
- CORS policy enforcement per origin
- Request routing to the correct service cluster
- Injecting `userId` and `role` as headers into forwarded requests

---

## Microservices

Eight independent services, each owning a specific domain.

| Service               | Technology    | Responsibility                                      |
|-----------------------|---------------|-----------------------------------------------------|
| Auth Service          | Laravel 12    | Registration, login, JWT issuance, refresh tokens   |
| Events Service        | .NET 9        | Event CRUD, lifecycle, immutability rules           |
| Seats Service         | .NET 9        | Seat map, reservation locking, availability updates |
| Tickets Service       | .NET 9        | Ticket creation, QR token generation (JWT RS256)    |
| Payments Service      | .NET 9        | Payment intents, webhook handling, POS confirmation |
| Access Control        | .NET 9        | QR validation, access logging, offline sync         |
| Notifications Service | .NET 9        | Email via SMTP, SignalR push, in-app notifications  |
| Audit Service         | .NET 9        | Immutable log of price/date changes and access events |

---

## Data Layer

### PostgreSQL — Single Database

All services share a single PostgreSQL instance with logical ownership
per domain. Three core tables drive the primary business flow:

- `users` — identity and authentication
- `events` — the business entity all operations revolve around
- `tickets` — the transaction record linking a user to an event and seat

Supporting tables (`event_sections`, `ticket_scans`, `employees`,
`favorites`, `notifications`, `pqrs`, `metrics`) extend core behavior
without creating cross-service coupling.

### Redis — Flow Control

Redis is not used as a primary store. It controls flow and protects
the database from high-contention and high-read scenarios.

| Key pattern              | TTL    | Purpose                          |
|--------------------------|--------|----------------------------------|
| `seat:lock:{seatId}`     | 720s   | Atomic seat reservation lock     |
| `rate:login:{ip}`        | 300s   | Brute-force login protection     |
| `cache:events:{id}`      | 60s    | High-read event response cache   |
| `email:queue:{userId}`   | 30s    | Email deduplication flag         |

---

## Async Messaging — RabbitMQ

Services communicate asynchronously via a single RabbitMQ Topic Exchange
named `nebula.events`. A service publishes an event when something meaningful
happens and never calls other services directly.

This means:
- A slow or unavailable consumer does not affect the publisher
- Messages persist in the queue until successfully processed
- Failed messages are retried up to 3 times before moving to a Dead Letter Queue

See [Event Bus diagram](../diagrams/05-event-bus.png) for the full
publisher → event → consumer mapping.

---

## External Services

| Service      | Provider    | Used by              | Purpose               |
|--------------|-------------|----------------------|-----------------------|
| Payments     | MercadoPago | Payments Service     | Online payment processing |
| Email        | SMTP        | Notifications Service| Transactional email delivery |

---

## Request Lifecycle — Summary

1. Frontend sends an HTTPS request with a JWT bearer token
2. YARP validates the JWT signature using the JWKS public key
3. YARP injects `userId` and `role` as headers and forwards the request
4. The target service processes the request and writes to PostgreSQL
5. If the operation triggers a side effect, the service publishes an event to RabbitMQ
6. Consumers (Notifications, Audit, etc.) process the event independently
7. Real-time updates reach the frontend via SignalR where applicable