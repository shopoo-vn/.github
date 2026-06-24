# 🛒 Implementation Plan: Dự án 2 - Marketplace Đồ Điện Tử

Dự án này là một hệ thống phân tán hoàn chỉnh, đòi hỏi xử lý thời gian thực (Real-time), dữ liệu do người dùng tạo (UGC) và tìm kiếm/lọc phức tạp. Kiến trúc **Pragmatic Polyglot Microservices** rất tỏa sáng ở dự án này.

---

## 1. Phân bổ Tech Stack cho từng Service

### 🟢 1. Media Service (Xử lý Ảnh)
*   **Tech Stack:** `Golang`
*   **Độ khó:** Dễ / Trung bình
*   **Tại sao:** Upload, resize, compress ảnh sản phẩm. Goroutines của Golang xử lý CPU-bound cực tốt mà không block event loop, phù hợp làm server xử lý file hơn Node.js.

### 🟢 2. Notification Service (Push Notification)
*   **Tech Stack:** `Golang`
*   **Độ khó:** Dễ
*   **Tại sao:** Background Worker lắng nghe RabbitMQ và gọi Firebase Cloud Messaging (FCM). Rất nhẹ, chạy background tốn ít RAM (~15-20MB).

### 🟢 3. Auth & User Service
*   **Tech Stack:** `Golang`
*   **Độ khó:** Dễ
*   **Tại sao:** Các thao tác Login, JWT, lấy Profile đơn giản (EASY).

### 🟡 4. Listing & Search Service (Quản lý Tin đăng & Tìm kiếm)
*   **Tech Stack:** `Node.js (NestJS)`
*   **Độ khó:** Tầm Trung
*   **Tại sao:** Đây là trái tim của hệ thống. Chứa các query lọc sản phẩm (Theo giá, khu vực, danh mục) và Full-Text Search PostgreSQL. NestJS với DTO/Validation và TypeORM giúp tổ chức logic CRUD lớn (MEDIUM) chuẩn xác và dễ maintain hơn Go.

### 🟡 5. Admin Service (Dashboard)
*   **Tech Stack:** `Node.js (NestJS)`
*   **Độ khó:** Tầm Trung
*   **Tại sao:** Gọi sang Listing Service và User Service để tổng hợp số liệu (Aggregation), xử lý báo cáo (Report) và kiểm duyệt. Rất nhiều endpoint API mang tính logic nghiệp vụ, phù hợp với hệ sinh thái Enterprise của NestJS.

### 🔴 6. Chat Service (Nhắn tin Real-time)
*   **Tech Stack:** `Node.js (ExpressJS)`
*   **Độ khó:** Khó
*   **Tại sao:** Tính năng khó nhất (HARD): Quản lý WebSockets, user online/offline, đồng bộ tin nhắn 2 chiều. Dùng ExpressJS + Socket.io vì bạn đã quen thuộc, không bị cản trở bởi các layer trừu tượng của NestJS, và Node.js làm cực kỳ tốt việc xử lý I/O concurrency cho WebSockets.

---

## 2. Kiến trúc Giao tiếp (Inter-service Communication)

*   **Đồng bộ (REST API):** Frontend gọi qua API Gateway (hoặc trực tiếp) tới các services. Admin Service gọi User/Listing service để lấy data.
*   **Bất đồng bộ (RabbitMQ):**
    *   *Event `chat.message.created`*: Khi có tin nhắn từ Chat Service, nó bắn vào MQ, Notification Service nhặt để báo Push Noti tới điện thoại.
    *   *Event `listing.created`*: Listing Service bắn vào MQ, Admin Service nhặt để đưa vào hàng đợi kiểm duyệt thủ công; sau khi duyệt, Admin bắn `listing.approved`/`listing.rejected` để Listing đổi trạng thái và Noti báo người bán.

---

## 3. Cấu trúc Project

```text
marketplace-project/
├── frontend/
│   ├── mobile-app/      (React Native)
│   └── web-admin/       (Next.js)
├── services/
│   ├── auth-service/    (Golang)
│   ├── media-service/   (Golang)
│   ├── noti-service/    (Golang)
│   ├── listing-service/ (NestJS)
│   ├── admin-service/   (NestJS)
│   └── chat-service/    (ExpressJS)
└── docker-compose.yml   (Postgres, Redis, RabbitMQ, MinIO + 6 service)
```

> 📌 **Cấu trúc thực tế đã chốt:** không gộp 1 monorepo. `shopoo/` là folder ô chứa **4 repo độc lập** — `backend/` (toàn bộ services + `infra/`, `keys/`, `docker-compose.yml`), `web/` (Next.js), `mobile/` (React Native), và `shopoo` (workspace: tài liệu kế hoạch + `.claude/skills` + `CLAUDE.md`, gitignore 3 thư mục code). Hạ tầng (Postgres/Redis/RabbitMQ/MinIO) chạy bằng **docker-compose**. Sơ đồ trên chỉ minh hoạ phần phân rã service bên trong repo `backend/`.

---

## ⚠️ User Review Required

Bạn hãy xem xét:
1. Việc phân rã như trên cho dự án Marketplace đã đáp ứng đúng mục tiêu độ khó (EASY=Go, MEDIUM=Nest, HARD=Express) của bạn chưa? → ✅ đã chốt.
2. Mỗi service có database schema độc lập trong PostgreSQL (schema + role riêng). → ✅ đã chốt & triển khai.
3. Dự án Marketplace đã được khởi tạo: backend chạy 6/6 service qua docker-compose; tiếp theo là `web` + `mobile`.
