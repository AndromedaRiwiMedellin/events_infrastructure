# System Overview

Andromeda is a ticketing and event management platform composed of four
independent frontend applications sharing a common backend infrastructure.

All communication between frontend applications and backend services is routed
through a central API Gateway.

---

## High-Level Architecture

![System Architecture](../diagrams/01-system-architecture.png)

---

## Frontends

Four independent applications, each with a specific role and audience.
Applications are independently deployed and communicate with backend services
through REST APIs exposed by the gateway.

| App        | Domain                               | Technology         | Audience              |
|------------|--------------------------------------|--------------------|-----------------------|
| User App   | andromeda.andrescortes.dev           | React              | End users             |
| Admin App  | admin.andromeda.andrescortes.dev     | Laravel + Livewire | Admins / SuperAdmins  |
| POS App    | tickets.andromeda.andrescortes.dev   | React              | Ticket operators      |
| Access App | access.andromeda.andrescortes.dev    | React              | Door staff            |

---

## API Gateway — YARP

All frontend requests pass through YARP (Yet Another Reverse Proxy), a
.NET 9 native reverse proxy library. It handles cross-cutting concerns
before any request reaches a backend service.

Responsibilities:
- JWT RS256 validation via JWKS endpoint
- Rate limiting backed by Redis counters
- CORS policy enforcement per origin
- Request routing to the correct service
- Forwarding authenticated user context through request headers

---

## Backend Services

Backend functionality is divided into independent services, each responsible
for a specific business domain.

| Service               | Technology         | Responsibility                                      |
|-----------------------|-------------------|-----------------------------------------------------|
| Auth Service          | ASP.NET Core 9    | Authentication, JWT issuance, refresh tokens        |
| Event Service         | ASP.NET Core 9    | Event management, seat availability, business rules |
| Ticket Service        | ASP.NET Core 9    | Ticket purchases, QR generation                     |
| POS Service           | ASP.NET Core 9    | Physical ticket sales                               |
| Access Service        | ASP.NET Core 9    | QR validation, offline synchronization              |
| Notification Service  | Laravel 13        | Transactional email delivery                        |

---

## Data Layer

### PostgreSQL — Single Database

All services share a single PostgreSQL instance with logical ownership
per domain. Three core tables drive the primary business flow:

- `users` — identity and authentication
- `events` — platform event management
- `tickets` — ticket ownership and purchase records

Supporting tables extend platform functionality without introducing
tight service coupling.

### Redis — Flow Control

Redis is used for concurrency control, caching, and request protection.

| Key pattern            | TTL    | Purpose                        |
|------------------------|--------|--------------------------------|
| `seat:lock:{seatId}`   | 720s   | Atomic seat reservation lock   |
| `rate:login:{ip}`      | 300s   | Brute-force login protection   |
| `cache:events:{id}`    | 60s    | Event response caching         |
| `email:queue:{userId}` | 30s    | Email deduplication flag       |

---

## Async Messaging — RabbitMQ

Services communicate asynchronously through RabbitMQ using event-driven
communication patterns.

This means:
- Services remain loosely coupled
- Consumers can process events independently
- Messages persist until successfully processed
- Failed messages retry automatically before reaching a Dead Letter Queue

Typical events include:
- `ticket.purchased`
- `pos.sale.completed`
- `access.validated`

See [Event Bus diagram](../diagrams/05-event-bus.png) for the complete
publisher → event → consumer mapping.

---

## External Services

| Service      | Provider    | Used by               | Purpose                    |
|---------------|-------------|-----------------------|----------------------------|
| Payments      | MercadoPago | Ticket Service        | Online payment processing  |
| Email         | SMTP        | Notification Service  | Transactional email delivery |

---

## Request Lifecycle — Summary

1. A frontend application sends an HTTPS request with a JWT bearer token
2. YARP validates the JWT signature using the JWKS public key
3. The gateway forwards the authenticated request to the target service
4. The service processes the request and writes data to PostgreSQL
5. Business events are published to RabbitMQ when required
6. Consumer services process events asynchronously
7. Responses are returned to the frontend application
