# 🛒 Shopoo - The Local Hub for Second-Hand Electronics

Welcome to **Shopoo**, a hyper-local marketplace platform designed exclusively for buying and selling used electronic devices within Ho Chi Minh City, Vietnam.

Whether you are looking to upgrade your smartphone, find a budget-friendly laptop for study, or simply declutter your old tech gadgets, Shopoo connects you directly with buyers and sellers right in your neighborhood. Built with a focus on simplicity, speed, and trust, our platform cuts through the noise of massive global e-commerce sites to provide a seamless, community-driven trading experience.

### ✨ Key Features
* **Hyper-Local Focus:** Browse and list items exclusively within the districts of Ho Chi Minh City, ensuring quick meetups, easy inspections, and hassle-free transactions.
* **Real-Time Communication:** Connect instantly with users through our built-in live chat to negotiate prices, ask for more details, and arrange safe meetups.
* **Smart Categorization:** Easily find exactly what you're looking for with dedicated categories for Smartphones, Laptops, Tablets, Audio gear, and Tech Accessories.
* **Trust & Transparency:** A transparent user rating and review system helps you identify reputable sellers and trade with absolute confidence.
* **Lightning-Fast Discovery:** Powerful full-text search and advanced filters (by price range, device condition, and specific district) to help you discover the best tech deals in seconds.

## 🏗️ Architecture

Shopoo is built as a **pragmatic polyglot microservices** system — each service uses the language best suited to its job, talking to each other over REST (synchronous) and RabbitMQ events (asynchronous).

| Service | Stack | Responsibility |
|---|---|---|
| **Auth & User** | Go | Registration, login, RS256 JWT, profiles |
| **Media** | Go | Image upload, resize/compress, object storage |
| **Notification** | Go | Push notifications (FCM) — RabbitMQ worker |
| **Listing & Search** | NestJS | Listings CRUD, categories, full-text search & filters |
| **Admin** | NestJS | Moderation queue, dashboard, reports |
| **Chat** | Express + Socket.io | Real-time 1‑to‑1 buyer ↔ seller messaging |

**Shared infrastructure:** PostgreSQL (one schema + least‑privilege role per service), Redis (cache, sessions, presence), RabbitMQ (event bus), MinIO (S3‑compatible object storage). Authentication uses **RS256 JWT** — the Auth service signs with a private key; every other service verifies with the public key only.

### Repositories
| Repo | Description |
|---|---|
| [`backend`](https://github.com/shopoo-vn/backend) | All six microservices + `docker-compose` infrastructure |
| [`web`](https://github.com/shopoo-vn/web) | Admin dashboard — Next.js (App Router) |
| [`mobile`](https://github.com/shopoo-vn/mobile) | Consumer app — React Native (Expo) |

## 🚀 Getting Started

**Prerequisites:** [Docker Desktop](https://www.docker.com/products/docker-desktop/) (with Compose), Node.js 20+. Go 1.22+ is optional — the Go services build inside their containers.

**1. Run the backend (all services + infrastructure):**
```bash
git clone https://github.com/shopoo-vn/backend.git
cd backend
docker compose up -d --build
```
This starts all six services plus PostgreSQL, Redis, RabbitMQ and MinIO. Health checks:
- Auth `:8001` · Media `:8002` · Notification `:8003` · Listing `:8004` · Admin `:8005` · Chat `:8006`
- RabbitMQ UI `:15672` · MinIO console `:9001`

```bash
curl http://localhost:8001/health
```

**2. Web admin & mobile app:** see the README in the [`web`](https://github.com/shopoo-vn/web) and [`mobile`](https://github.com/shopoo-vn/mobile) repositories.

## 📬 Connect

Have a question, found a bug, or want to contribute? Reach out:

* **Email:** [kiethohohoho@gmail.com](mailto:kiethohohoho@gmail.com)
* **Messenger:** [Chat with us](https://www.messenger.com/e2ee/t/6725676397522886)
