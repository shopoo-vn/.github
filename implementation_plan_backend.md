# ⚙️ Implementation Plan — BACKEND (Marketplace Đồ Điện Tử)

> Plan này bám theo `implementation_plan_marketplace.md`. Kiến trúc: **Pragmatic Polyglot Microservices** (Go cho service EASY, NestJS cho MEDIUM, ExpressJS cho HARD). DB-per-service trên cùng 1 PostgreSQL instance, giao tiếp đồng bộ qua REST và bất đồng bộ qua RabbitMQ.

---

## 0. Tổng quan kiến trúc

```text
                       ┌─────────────────────────────┐
   Mobile / Web Admin  │        API Gateway          │  (REST, JWT validate)
        (REST/WS)  ───▶ │   (NGINX hoặc Go gateway)   │
                       └──────────────┬──────────────┘
        ┌──────────────┬──────────────┼──────────────┬───────────────┐
        ▼              ▼              ▼              ▼               ▼
  ┌──────────┐  ┌───────────┐  ┌────────────┐  ┌───────────┐  ┌──────────┐
  │  Auth &  │  │  Media    │  │  Listing & │  │  Admin    │  │   Chat   │
  │  User    │  │  Service  │  │  Search    │  │  Service  │  │  Service │
  │  (Go)    │  │  (Go)     │  │  (NestJS)  │  │  (NestJS) │  │ (Express)│
  └────┬─────┘  └─────┬─────┘  └─────┬──────┘  └─────┬─────┘  └────┬─────┘
       │              │              │               │              │
       └──────────────┴──────────────┴───────────────┴──────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                      ▼
        ┌──────────┐          ┌────────────┐          ┌───────────┐
        │PostgreSQL│          │   Redis    │          │ RabbitMQ  │
        │(schema/  │          │(cache,     │          │ (events)  │
        │ service) │          │ session,   │          └─────┬─────┘
        └──────────┘          │ presence)  │                │
                              └────────────┘                ▼
                                                     ┌──────────────┐
                                                     │ Noti Service │
                                                     │    (Go)      │──▶ FCM
                                                     └──────────────┘
```

| Service | Stack | Độ khó | DB schema | Port (dev) |
|---|---|---|---|---|
| Auth & User | Go (net/http hoặc Gin) | EASY | `auth` | 8001 |
| Media | Go | EASY/MEDIUM | `media` (metadata) + object storage | 8002 |
| Notification | Go (worker) | EASY | `noti` | 8003 |
| Listing & Search | NestJS + TypeORM | MEDIUM | `listing` | 8004 |
| Admin | NestJS | MEDIUM | `admin` (read-mostly) | 8005 |
| Chat | ExpressJS + Socket.io | HARD | `chat` | 8006 |

---

## 1. Shared Infrastructure (docker-compose)

- **PostgreSQL 16** — 1 container, mỗi service 1 schema riêng (`auth`, `listing`, `chat`, …). Mỗi service có user DB riêng chỉ quyền trên schema của nó → tách biệt logic, dễ tách instance sau này.
- **Redis 7** — cache kết quả search, lưu refresh-token/session, lưu trạng thái online/offline cho Chat (presence), rate-limit.
- **RabbitMQ 3 (management)** — exchange `marketplace.events` kiểu `topic`. Mỗi consumer 1 queue riêng bind theo routing key.
- **MinIO** (tùy chọn) — object storage S3-compatible cho ảnh, để Media Service không lưu file vào local disk. Nếu muốn đơn giản giai đoạn đầu: lưu local + serve static.

**Việc cần làm:** ✅ đã làm
1. `docker-compose.yml` chạy Postgres + Redis + RabbitMQ + MinIO (+ cả 6 service) với volume persist + healthcheck.
2. `infra/postgres/init-db.sql` tạo các schema + DB user/role (Postgres chạy tự động ở lần init đầu).
3. Mỗi service có `.env.example` riêng (DB url, Redis url, RabbitMQ url, JWT public key path).

---

## 2. Đặc tả chi tiết từng Service

### 🟢 2.1 Auth & User Service (Go) — EASY
**Trách nhiệm:** đăng ký, đăng nhập, refresh token, CRUD profile, là **nguồn chân lý (source of truth)** về user.

**Endpoints (REST):**
- `POST /auth/register`
- `POST /auth/login` → trả access JWT (15') + refresh token (lưu Redis)
- `POST /auth/refresh`
- `POST /auth/logout`
- `GET  /users/me` (auth)
- `PATCH /users/me`
- `GET  /users/:id` (internal, cho service khác lấy info người bán)

**DB (`auth`):** `users(id, email, phone, password_hash, display_name, avatar_url, status, created_at)`

**Events publish:** `user.registered`, `user.updated`.
**Phụ thuộc:** Redis (refresh token). Phát hành & ký JWT bằng **RS256**; các service khác chỉ verify bằng **public key**.

---

### 🟢 2.2 Media Service (Go) — EASY/MEDIUM
**Trách nhiệm:** nhận upload ảnh sản phẩm, resize/compress nhiều kích thước (thumbnail/medium/full), trả về URL.

**Endpoints:**
- `POST /media/upload` (multipart, auth) → resize bằng goroutines, lưu MinIO/local, trả `{ id, urls: {thumb, medium, full} }`
- `GET  /media/:id` (hoặc serve qua static/CDN)
- `DELETE /media/:id` (auth, chủ sở hữu)

**DB (`media`):** `media(id, owner_id, original_name, sizes_json, created_at)`
**Kỹ thuật:** dùng `imaging`/`bimg`, worker pool goroutine giới hạn concurrency để không OOM. Validate mime/size.
**Phụ thuộc:** Object storage (MinIO/local).

---

### 🟢 2.3 Notification Service (Go) — EASY
**Trách nhiệm:** background worker, **không có REST public** (hoặc chỉ vài endpoint quản lý device token). Lắng nghe RabbitMQ → gọi FCM push.

**Endpoints (nhỏ):**
- `POST /devices` (auth) — đăng ký FCM device token
- `DELETE /devices/:token`

**Events consume:**
- `chat.message.created` → push "Bạn có tin nhắn mới"
- `listing.approved` / `listing.rejected` → push cho người bán
**DB (`noti`):** `device_tokens(user_id, token, platform)`, `notifications(id, user_id, type, payload, read, created_at)`
**Phụ thuộc:** RabbitMQ (consumer), Firebase Admin SDK (FCM), gọi Auth Service để biết user.

---

### 🟡 2.4 Listing & Search Service (NestJS) — MEDIUM — **TRÁI TIM HỆ THỐNG**
**Trách nhiệm:** CRUD tin đăng, danh mục, lọc nâng cao (giá/khu vực/danh mục/tình trạng) và Full-Text Search PostgreSQL.

**Endpoints:**
- `POST   /listings` (auth) — tạo tin (kèm media ids)
- `PATCH  /listings/:id` (auth, owner)
- `DELETE /listings/:id`
- `GET    /listings/:id`
- `GET    /listings` — query: `q, category, minPrice, maxPrice, location, condition, sort, page, limit`
- `GET    /categories`

**DB (`listing`):**
- `categories(id, name, slug, parent_id)`
- `listings(id, seller_id, title, description, price, category_id, location, condition, status, media_ids[], search_vector tsvector, created_at)`
- GIN index trên `search_vector`; index trên `(category_id, price, location, status)`.
- Trigger cập nhật `search_vector` từ title+description.

**Events publish:** `listing.created` (→ Admin đưa vào hàng đợi kiểm duyệt), `listing.updated`, `listing.deleted`.
**Events consume:** `listing.approved`/`rejected` từ Admin → đổi `status`.
**Phụ thuộc:** Redis (cache trang search nóng), gọi Auth Service lấy info seller (hoặc cache).
**Điểm nhấn NestJS:** dùng DTO + `class-validator`, TypeORM repository, module hóa rõ ràng.

---

### 🟡 2.5 Admin Service (NestJS) — MEDIUM
**Trách nhiệm:** dashboard tổng hợp, kiểm duyệt tin đăng, quản lý user, báo cáo.

**Endpoints (đều yêu cầu role=admin):**
- `GET   /admin/dashboard` — số liệu tổng hợp (gọi Listing + Auth + aggregate)
- `GET   /admin/moderation/queue` — danh sách tin chờ duyệt
- `POST  /admin/moderation/:listingId/approve`
- `POST  /admin/moderation/:listingId/reject`
- `GET   /admin/users` / `PATCH /admin/users/:id/ban`
- `GET   /admin/reports` — tin bị report

**DB (`admin`):** `moderation_items(listing_id, status, reviewed_by, reason, created_at)`, `reports(id, target_type, target_id, reporter_id, reason, status)`. Phần lớn là read-mostly + aggregate.

**Events consume:** `listing.created` → thêm vào `moderation_items` (pending).
**Events publish:** `listing.approved`, `listing.rejected`.
**Phụ thuộc:** gọi REST sang Listing & Auth để aggregate; cân nhắc cache số liệu dashboard ở Redis.

---

### 🔴 2.6 Chat Service (ExpressJS + Socket.io) — HARD
**Trách nhiệm:** nhắn tin real-time 1-1 giữa người mua & người bán, presence online/offline, đồng bộ 2 chiều, lịch sử tin nhắn.

**REST (lịch sử / fallback):**
- `GET /chat/conversations` (auth)
- `GET /chat/conversations/:id/messages?before=&limit=`
- `POST /chat/conversations` — tạo/lấy conversation theo (buyer, seller, listing)

**WebSocket (Socket.io) events:**
- client→server: `message:send`, `typing`, `message:read`
- server→client: `message:new`, `presence:update`, `message:delivered`
- Auth handshake bằng JWT trong `socket.handshake.auth.token`.

**DB (`chat`):**
- `conversations(id, listing_id, buyer_id, seller_id, last_message_at)`
- `messages(id, conversation_id, sender_id, body, created_at, read_at)`
- index `(conversation_id, created_at)` cho pagination.

**Redis:** lưu map `userId → socketId(s)` cho presence + định tuyến tin nhắn khi scale nhiều instance (Socket.io Redis adapter).
**Events publish:** `chat.message.created` → Notification Service push FCM khi người nhận offline.
**Điểm khó:** xử lý nhiều socket / 1 user, reconnect, message ordering, delivery/read receipt, scale ngang bằng Redis adapter.

---

## 3. Event Catalog (RabbitMQ — exchange `marketplace.events`, topic)

| Routing key | Producer | Consumer | Mục đích |
|---|---|---|---|
| `user.registered` | Auth | (tùy chọn) | seed/cache |
| `listing.created` | Listing | Admin | đưa vào hàng đợi kiểm duyệt |
| `listing.approved` | Admin | Listing, Noti | publish tin + push cho seller |
| `listing.rejected` | Admin | Listing, Noti | ẩn tin + báo seller |
| `chat.message.created` | Chat | Noti | push tin nhắn mới |

> Quy ước: payload JSON có `eventId`, `occurredAt`, `data`. Consumer idempotent (lưu `eventId` đã xử lý).

---

## 4. Auth & Bảo mật xuyên service

- **JWT RS256** ✅ (đã chốt): Auth Service giữ **private key** để ký token; các service khác chỉ cần **public key** để verify (không chia sẻ secret). Cặp khóa sinh bằng OpenSSL, để ở `keys/` (gitignore) và mount vào container.
- API Gateway hoặc từng service tự verify token qua middleware chung.
- Role trong claim (`user` / `admin`) để Admin Service gate.
- Rate-limit cơ bản ở Gateway/Redis.

---

## 5. Lộ trình triển khai (Milestones)

**M0 — Nền tảng (1 buổi):** docker-compose (PG/Redis/RabbitMQ/MinIO + 6 service), init schema, repo monorepo, `.env`, script chạy từng service.

**M1 — Auth & User (Go):** register/login/JWT/profile + middleware verify token dùng chung. → **Mốc chặn nhất, làm trước.**

**M2 — Listing & Search (NestJS):** CRUD + filter + full-text + publish `listing.created`. (Có thể seed category.)

**M3 — Media (Go):** upload/resize, tích hợp vào luồng tạo listing.

**M4 — Admin (NestJS):** moderation queue consume `listing.created`, approve/reject publish event; dashboard aggregate.

**M5 — Chat (Express/Socket.io):** REST history + WS real-time + presence Redis + publish `chat.message.created`.

**M6 — Notification (Go):** consume `chat.message.created` & `listing.*`, push FCM, device token.

**M7 — Hoàn thiện:** API Gateway, healthcheck, logging chung, seed data, tài liệu API (OpenAPI/Postman).

> Thứ tự ưu tiên theo phụ thuộc: **Auth → Listing → Media → Admin → Chat → Noti**. Mỗi milestone phải chạy & test độc lập được trước khi sang cái sau.

---

## 6. Cấu trúc thư mục (Backend trong monorepo)

```text
marketplace-project/
├── services/
│   ├── auth-service/        (Go)    — cmd/, internal/{handler,service,repo}, migrations/
│   ├── media-service/       (Go)
│   ├── noti-service/        (Go)
│   ├── listing-service/     (NestJS)— src/{listing,category,search,common}, dto/, entities/
│   ├── admin-service/       (NestJS)
│   └── chat-service/        (Express)— src/{routes,sockets,services,repo}
├── infra/postgres/init-db.sql   — schema + role mỗi service
├── scripts/gen-keys.*           — sinh cặp khóa RS256
├── keys/                        — RS256 keypair (gitignored)
└── docker-compose.yml           — 6 service + Postgres/Redis/RabbitMQ/MinIO
```

---

## 7. Việc cần chốt trước khi code

1. **API Gateway**: dùng NGINX (cấu hình route đơn giản) hay viết gateway nhỏ bằng Go? → Đề xuất: NGINX cho nhanh, hoặc gọi trực tiếp service trong giai đoạn dev. *(chưa chốt)*
2. ✅ **JWT**: **RS256** — Auth ký bằng private key, service khác verify bằng public key.
3. ✅ **Object storage**: **MinIO** ngay từ đầu (S3-compatible, trong docker-compose).
4. **Migration tool**: Go services dùng SQL embed + chạy lúc khởi động (giai đoạn đầu), có thể nâng lên `golang-migrate` sau; NestJS dùng TypeORM migrations. *(chưa chốt hẳn)*
