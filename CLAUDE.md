# CLAUDE.md — shopoo Marketplace workspace

Workspace = **3 independent repos** under `shopoo/`: `backend/` (Go + NestJS + Express microservices), `web/` (Next.js admin), `mobile/` (React Native). Planning docs live at the root (`implementation_plan_*.md`).

## Always follow the skill set
When writing or reviewing code, **load the matching skill(s) first** and code to them:

| You are touching… | Load skill |
|---|---|
| anything in this project (cross-cutting) | **`marketplace-conventions`** (always) |
| `backend/services/{auth,media,noti}-service` (Go) | `golang-service-patterns` |
| `backend/services/{listing,admin}-service` (NestJS) | `nestjs-service-patterns` |
| `backend/services/chat-service` (Express+Socket.io) | `express-socketio-realtime` |
| `web/` (Next.js) | `nextjs-web-patterns` (+ `design-taste-frontend` for visuals) |
| `mobile/` (React Native) | `react-native-expo-patterns` |

`marketplace-conventions` wins on any cross-service concern (API shape, error shape, events, auth, env, DB naming).

## Locked decisions (don't re-litigate)
- **Auth:** RS256 JWT — Auth signs with private key, others verify with `keys/jwt_public.pem`. Access ~15m, opaque refresh in Redis (rotated).
- **DB:** one Postgres, **schema + least-privilege role per service**; no cross-schema reads.
- **Async:** RabbitMQ topic exchange `marketplace.events`, standard envelope `{eventId,type,occurredAt,data}`, idempotent consumers.
- **Object storage:** MinIO (S3-compatible).
- **Error shape everywhere:** `{"error":"message"}`. **Health:** `GET /health`.

## Milestone order (status)
Auth ✅ → Listing ✅ → Media (Go) ✅code → Admin ✅ → Chat ✅ → Notification (Go) ✅code → **web + mobile ← next** (frontends). Backend is code-complete: NestJS/Express services build-verified; Go services (media, noti) need `go mod tidy && go build` to verify (Go not installed here). Each milestone must run & be testable on its own.

## Build / run — Docker (best practice for this microservices stack)
Full runbook: `backend/README.md`.
- Whole stack: from `backend/` → `docker compose up -d --build` (all 6 services + Postgres/Redis/RabbitMQ/MinIO). `init-db.sql` runs on first Postgres init; keys mount read-only from `keys/`.
- Hot-reload one service: `docker compose up -d postgres redis rabbitmq minio`, then `cd backend/services/<svc>` → `npm run start:dev` (Node) or `go run ./cmd/server` (Go).

## Environment note
Docker Desktop works on this box (it just needed a restart to pick up virtualization). Go services build **inside their containers** (Dockerfiles run `go mod tidy`), so a local Go toolchain isn't required to run the stack; Node services are also `npm run build`-verified. Re-check toolchain availability before claiming something was run.

## House rules
Conventional Commits. Don't commit `.env` or `keys/*.pem`. Update the service README endpoint table and `.env.example` as part of each feature (see `marketplace-conventions` §9 Definition of Done).
