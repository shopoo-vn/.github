---
name: express-socketio-realtime
description: Best practices for the real-time chat-service in shopoo (Express + Socket.io + TypeScript). Use when writing or reviewing the chat service or any WebSocket code — covers project structure, sharing JWT auth between HTTP and WS, socket handshake auth (RS256 public key), rooms/presence in Redis, the Redis adapter for horizontal scale, payload validation, ack/error patterns, message persistence + RabbitMQ publish, and graceful shutdown. Pair with marketplace-conventions.
---

# Express + Socket.io Realtime Patterns

> The "HARD" service: 1-1 buyer↔seller chat, presence, delivery/read receipts. Express handles REST history; Socket.io handles live messaging. Node's strength is I/O concurrency — keep the event loop free.

## Structure
```
src/
  config/            typed env config
  app.ts             express app (REST: conversations, history)
  server.ts          http.Server + Socket.io + graceful shutdown
  http/              REST routes + controllers + validation
  sockets/           io setup, auth middleware, event handlers (message, presence, typing)
  services/          chat logic (persist, fan-out, publish events)
  repo/              pg queries (conversations, messages)
  lib/               redis, rabbitmq, jwt-verify
```

## Auth (shared between REST and WS)
- REST: Bearer-token middleware verifying **RS256 with the public key**.
- WS: `io.use()` handshake middleware reading `socket.handshake.auth.token`; verify the same way; attach `socket.data.userId`/`role`. Reject (`next(err)`) on invalid token — never let an unauthenticated socket join a room.

## Rooms, presence, scale
- Each conversation is a room (`conv:<id>`). On `message:send`, authorize the sender is a participant, **persist to Postgres first**, then `io.to(room).emit('message:new', msg)`.
- Presence: store `online:<userId> → Set<socketId>` in Redis; update on connect/disconnect; emit `presence:update` to relevant peers. A user may have multiple sockets — track all, only mark offline when the last disconnects.
- Horizontal scale: use **`@socket.io/redis-adapter`** so emits reach sockets on other instances. Never keep cross-request state in process memory.

## Event contract
- Names are namespaced verbs: client→server `message:send`, `typing`, `message:read`; server→client `message:new`, `message:delivered`, `presence:update`.
- **Validate every inbound payload** (zod) before acting. Use **ack callbacks** for request/response (`socket.emit('message:send', dto, (ack) => ...)`) and emit an `error` event with the project error shape on failure.
- Be idempotent: clients retry on reconnect — dedupe by a client-supplied `clientMsgId`.

## Persistence + events
- Messages table indexed by `(conversation_id, created_at)` for history pagination (`GET /conversations/:id/messages?before=&limit=`).
- After persisting a message, publish `chat.message.created` to `marketplace.events` (standard envelope) so Notification Service can push to offline recipients.

## Event loop hygiene & lifecycle
- Never run CPU-heavy or sync-blocking work in a handler; offload or stream. No synchronous fs/crypto in the hot path.
- **Graceful shutdown:** stop accepting connections, `io.close()`, drain, close redis/pg/amqp, then exit. Handle SIGTERM.
- Use TypeScript everywhere; type the socket event maps (`Server<ClientToServer, ServerToClient>`).

## Don't
Don't trust client-sent senderId (use `socket.data.userId`) · don't emit before persisting · don't store presence/rooms in memory when scaling · don't block the event loop · don't skip payload validation.
