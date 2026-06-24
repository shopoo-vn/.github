---
name: marketplace-conventions
description: Cross-cutting engineering conventions for the shopoo Marketplace (all backend services + frontends). Use whenever writing or reviewing code in this project — defines the shared REST contract, error shape, pagination, RabbitMQ event envelope, JWT/auth rules, env/config, DB naming, Docker, logging, and security baseline that every service must follow. Always consult this alongside the per-tech skill.
---

# Marketplace Conventions (the glue every service obeys)

> These rules are **project-wide and non-negotiable**. Per-tech skills (golang/nestjs/express/nextjs/react-native) layer on top. If a per-tech idiom conflicts, this file wins for anything cross-service (API shape, events, auth, env).

## 1. HTTP API contract
- **Base style:** resource-oriented REST, JSON in/out. `Content-Type: application/json`.
- **Success:** return the resource (or `{items,...}` for lists) directly — no envelope.
- **Error (always this shape):** `{"error": "human readable message"}`. One field, lowercase message. Never leak stack traces or SQL.
- **Status codes:** 200 ok · 201 created · 204 no-content · 400 validation · 401 unauthenticated · 403 forbidden · 404 not-found · 409 conflict · 422 semantic · 500 server.
- **Pagination (lists):** query `?page=1&limit=20` (limit max 100). Response `{ "items": [...], "page": 1, "limit": 20, "total": 137 }`.
- **Timestamps:** ISO-8601 UTC in JSON. **IDs:** UUID v4 strings.
- **Health:** every service exposes `GET /health` → `{"status":"ok"}` (no auth, no DB requirement beyond liveness).

## 2. Auth (RS256 JWT)
- Auth Service signs **access tokens** with the RSA **private key** (`keys/jwt_private.pem`); every other service verifies with the **public key only** (`keys/jwt_public.pem`). No shared secrets.
- Access token: `sub` (userId), `role` (`user`|`admin`), `iss` = `marketplace-auth`, short TTL (~15m). Always verify `alg=RS256`, issuer, and expiry.
- Transport: `Authorization: Bearer <token>`. Refresh tokens are **opaque**, stored in Redis (`refresh:<token>`), rotated on use.
- Internal service-to-service endpoints (e.g. `GET /users/{id}`) must be marked internal and locked to the internal network / service auth before prod — never expose publicly.

## 3. RabbitMQ events (async)
- One topic exchange: **`marketplace.events`**. Each consumer owns a named, durable queue bound by routing key.
- **Envelope (every event):**
  ```json
  { "eventId": "<uuid>", "type": "listing.created", "occurredAt": "<iso8601>", "data": { } }
  ```
- Consumers are **idempotent**: track processed `eventId` (Redis/table) and ack only after success; on failure nack→DLQ, don't infinite-requeue.
- Routing keys in use: `user.registered`, `listing.created`, `listing.approved`, `listing.rejected`, `chat.message.created`. Add new ones here when introduced.
- Publishers must not block the request path on broker latency — publish after the DB commit; if the broker is down, log + (later) outbox, never lose the write.

## 4. Config & secrets (12-factor)
- All config via **environment variables**, read once at startup into a typed struct/object, **validated, fail-fast**. No reading `process.env`/`os.Getenv` scattered through the code.
- Commit `.env.example` (keys + safe defaults). **Never** commit `.env` or `keys/*.pem` (already gitignored).
- Per-service env vars are prefixed by service (`AUTH_*`, `LISTING_*`, …).

## 5. Database
- **Schema-per-service** on one Postgres (`auth`, `listing`, `chat`, …); each service has its own least-privilege role and never reads another service's schema directly — go through that service's API/events.
- `snake_case` columns, UUID PKs (`gen_random_uuid()`), `created_at`/`updated_at` `TIMESTAMPTZ NOT NULL DEFAULT now()`.
- Parameterized queries only (no string concatenation). Index every foreign key and every column you filter/sort on.
- Migrations are explicit and ordered; never auto-`synchronize` a real database.

## 6. Security baseline
- Passwords: bcrypt (cost ≥ 10). Validate & whitelist all input. CORS allow-list (no `*` with credentials).
- Never log secrets, tokens, passwords, or full PII. Rate-limit auth + write endpoints.
- Least privilege everywhere (DB roles, MinIO keys, container user).

## 7. Logging & observability
- **Structured logs** (JSON): include `requestId`, `service`, `level`, `msg`. Propagate a request id per request.
- Log errors with context at the boundary (handler), not at every layer. Don't log-and-rethrow.

## 8. Docker & packaging
- Each service ships a **multi-stage Dockerfile**: small final image, **non-root** user, `EXPOSE` the port, one process per container.
- `backend/docker-compose.yml` runs all 6 services + Postgres/Redis/RabbitMQ/MinIO with healthchecks and `depends_on`. `init-db.sql` runs on first Postgres init; the RS256 **public** key mounts read-only into each service. Run the stack with `docker compose up -d --build`.
- For hot-reload dev, run only the infra in compose and the service via `npm run start:dev` / `go run` — the `.env.example` files default to `localhost`, matching the published infra ports.

## 9. Definition of done (per endpoint/feature)
Input validated · errors mapped to the shape above · authz enforced · happy + error paths covered by a test · structured logs · `.env.example` updated · README endpoint table updated.
