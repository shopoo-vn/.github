---
name: nestjs-service-patterns
description: Best practices for the NestJS backend services in shopoo (listing-service, admin-service). Use when writing, reviewing, or refactoring NestJS/TypeScript backend code — covers module structure, DTOs + class-validator, TypeORM entities & migrations, ConfigModule env validation, JWT guards (RS256 public key), exception filters matching the project error shape, RabbitMQ integration, pagination, and testing. Pair with marketplace-conventions.
---

# NestJS Service Patterns

> Used for the "MEDIUM" services: `listing-service`, `admin-service`. NestJS earns its keep here via DI, DTO validation, and module boundaries for large CRUD/business logic.

## Module structure
```
src/
  main.ts                 bootstrap: global pipes/filters, /health, shutdown hooks
  app.module.ts           imports ConfigModule + feature modules
  common/                 filters, interceptors, guards, dto (pagination), decorators
  config/                 typed config + Joi/zod env schema
  database/               TypeORM datasource + migrations
  <feature>/              feature module: controller, service, dto/, entities/
```
- One **feature module per domain** (`listing`, `category`, `search`). Controllers are thin; **business logic lives in services**; data access via injected repositories.
- Export a service from its module only when another module needs it.

## DTOs & validation
- Every request body/query is a **DTO class** with `class-validator` decorators. Enable a global `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })` in `main.ts`.
- Use `class-transformer` for type coercion (query strings → numbers, etc.). Never trust `any`.
- Response shaping: serialize entities (omit secrets) — `@Exclude()`/`ClassSerializerInterceptor` or explicit response DTOs.

## TypeORM
- Entities map to the service's schema (set `schema` in datasource or entity). `snake_case` columns (`namingStrategy` or explicit `name:`), UUID PKs, `@CreateDateColumn`/`@UpdateDateColumn`.
- **`synchronize: false` always.** Use generated migrations (`typeorm migration:generate`) checked into `database/migrations`, run on deploy/startup.
- Use the repository pattern via `@InjectRepository`. Build filtered queries with the QueryBuilder; parameterize everything. For full-text search use a `tsvector` column + GIN index and `to_tsquery`.

## Cross-cutting (match marketplace-conventions)
- **Errors:** a global `ExceptionFilter` that renders `{ "error": "<message>" }` and the right status. Throw Nest `HttpException` subclasses (`NotFoundException`, `ConflictException`, …) from services.
- **Auth:** `JwtAuthGuard` verifying **RS256 with the public key** (`passport-jwt` with `algorithms:['RS256']`, `publicKey` from `keys/jwt_public.pem`), plus a `RolesGuard` reading `role` from claims for admin endpoints. Get the user via a `@CurrentUser()` param decorator.
- **Config:** `ConfigModule.forRoot({ isGlobal:true, validationSchema })` — app refuses to boot on bad env. Inject `ConfigService`, never `process.env` in features.
- **Pagination:** shared `PaginationQueryDto { page, limit }` and a `Paginated<T> { items, page, limit, total }` response.

## RabbitMQ
- Wrap a single connection/channel in a `MessagingModule` (amqplib or `@golevelup/nestjs-rabbitmq`). Publish to `marketplace.events` with the standard envelope **after** the DB transaction commits. Consumers are idempotent and ack on success.

## main.ts essentials
Global `ValidationPipe` + exception filter · `app.enableShutdownHooks()` · CORS allow-list · `GET /health` · listen on `LISTING_HTTP_PORT`/`ADMIN_HTTP_PORT`.

## Testing
- Unit-test services with a mocked repository (`jest`). e2e with `supertest` against a test module. Cover validation failures + authz, not just happy paths.

## Don't
Don't put logic in controllers · don't enable `synchronize` · don't return raw entities with secret fields · don't read `process.env` outside config · don't let one service query another's schema.
