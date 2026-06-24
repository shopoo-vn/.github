# 📱 Implementation Plan — FRONTEND (Marketplace Đồ Điện Tử)

> Plan này bám theo `implementation_plan_marketplace.md`. Có **2 ứng dụng**:
> - **Mobile App** (React Native) — app người dùng cuối: duyệt/đăng tin, chat, nhận push.
> - **Web Admin** (Next.js) — dashboard cho admin: kiểm duyệt, quản lý user, báo cáo.
>
> Cả hai gọi backend qua REST + JWT; Mobile thêm WebSocket (Socket.io) cho chat và FCM cho push.
>
> 📌 **Cấu trúc:** mỗi app là **1 repo riêng** trong workspace `shopoo/` → `shopoo/web/` (Next.js) và `shopoo/mobile/` (React Native), tách biệt với `shopoo/backend/`.

---

## 0. Tổng quan & Tech chung

| App | Framework | State / Data | Realtime | Auth |
|---|---|---|---|---|
| Mobile | React Native (Expo) | React Query + Zustand | Socket.io client | JWT (SecureStore) |
| Web Admin | Next.js (App Router) | React Query | — | JWT (httpOnly cookie/session) |

**Shared concerns (cả 2 app):**
- **API client** chung: axios instance + interceptor gắn `Authorization: Bearer`, tự refresh khi 401.
- **Types**: định nghĩa TypeScript types theo contract backend (`User`, `Listing`, `Conversation`, `Message`…). Cân nhắc sinh từ OpenAPI.
- **.env** cho base URL theo môi trường (dev/prod).
- **Design system**: có sẵn skill `design-taste-frontend` trong `.claude/` — dùng để giữ UI nhất quán, tránh giao diện "template".

---

## 1. 📱 MOBILE APP (React Native + Expo)

### 1.1 Lựa chọn kỹ thuật
- **Expo (managed)** cho nhanh; cần native module FCM → dùng Expo + `expo-notifications` hoặc bare nếu cần `@react-native-firebase`.
- **Navigation**: React Navigation (Stack + Bottom Tabs).
- **Data fetching**: TanStack Query (cache, retry, pagination/infinite scroll cho list).
- **Global state nhẹ**: Zustand (auth user, socket connection, unread count).
- **Form**: React Hook Form + Zod.
- **Image**: `expo-image-picker` để chọn ảnh, upload multipart lên Media Service.

### 1.2 Điều hướng & màn hình

```text
RootNavigator
├── AuthStack (chưa login)
│   ├── Login
│   ├── Register
│   └── ForgotPassword
└── MainTabs (đã login)
    ├── HomeTab        → ListingList → ListingDetail
    │                                  └→ Chat (với người bán)
    ├── SearchTab      → SearchFilter → Results
    ├── PostTab        → CreateListing (multi-step: ảnh → thông tin → giá → đăng)
    ├── ChatTab        → ConversationList → ChatRoom
    └── ProfileTab     → MyProfile / MyListings / Settings / DeviceToken
```

### 1.3 Tính năng theo module

**A. Auth**
- Login/Register → lưu access token (memory/Zustand) + refresh token (`expo-secure-store`).
- Interceptor refresh token khi 401; logout xóa token.

**B. Browse & Search (Listing Service)**
- Home: list tin mới nhất, infinite scroll.
- Search: ô tìm kiếm full-text + bộ lọc (giá min/max, danh mục, khu vực, tình trạng), sort.
- ListingDetail: ảnh carousel, mô tả, giá, info người bán, nút **"Chat với người bán"** và **"Gọi"**.

**C. Create Listing (Listing + Media Service)**
- Multi-step form: chọn nhiều ảnh → upload lên Media Service (hiện progress) → nhận `media_ids` → submit `POST /listings`.
- Validate bằng Zod; xử lý lỗi upload/đăng.

**D. Chat real-time (Chat Service — Socket.io)**
- Kết nối socket sau login (token qua handshake), reconnect khi mất mạng.
- ConversationList: danh sách hội thoại + last message + badge chưa đọc.
- ChatRoom: tin nhắn 2 chiều, `typing`, `message:read`, optimistic UI khi gửi.
- Đồng bộ lịch sử qua REST (`GET /messages?before=`), realtime qua WS.

**E. Push Notification (Noti Service — FCM)**
- Xin quyền, lấy FCM token → `POST /devices`.
- Nhận push tin nhắn mới / tin được duyệt; tap vào → deep link tới ChatRoom hoặc ListingDetail.

### 1.4 Cấu trúc thư mục (mobile)
```text
mobile/                  (repo riêng — shopoo/mobile/)
├── src/
│   ├── api/            (client.ts, listings.ts, auth.ts, chat.ts)
│   ├── navigation/
│   ├── screens/        (auth/, home/, search/, post/, chat/, profile/)
│   ├── components/     (ListingCard, FilterSheet, MessageBubble, …)
│   ├── store/          (auth.ts, socket.ts — Zustand)
│   ├── hooks/          (useListings, useChat, useAuth)
│   └── types/
└── app.json / .env
```

---

## 2. 💻 WEB ADMIN (Next.js — Admin Service)

### 2.1 Lựa chọn kỹ thuật
- **Next.js App Router** + TypeScript.
- **Auth**: chỉ role `admin`; lưu token httpOnly cookie (an toàn hơn localStorage) hoặc NextAuth credentials.
- **Data**: React Query + server actions/route handlers proxy sang Admin Service.
- **UI**: component library (shadcn/ui hoặc Ant Design — hợp dashboard) + biểu đồ (Recharts).
- Áp dụng skill `design-taste-frontend` để dashboard gọn, chuyên nghiệp.

### 2.2 Sơ đồ trang
```text
/login
/(dashboard)
├── /                      → Overview: số tin, user, tin chờ duyệt, biểu đồ
├── /moderation            → Hàng đợi kiểm duyệt (approve/reject + lý do)
├── /listings              → Toàn bộ tin (search/filter/đổi trạng thái)
├── /users                 → Danh sách user, ban/unban
└── /reports               → Tin/người dùng bị report
```

### 2.3 Tính năng theo trang
- **Dashboard**: gọi `GET /admin/dashboard` → cards + charts (tin theo ngày, user mới, tỉ lệ duyệt).
- **Moderation queue**: bảng tin `pending`, xem chi tiết (ảnh + mô tả), nút Approve/Reject (nhập lý do) → gọi Admin Service (backend publish `listing.approved/rejected`).
- **Users**: bảng + tìm kiếm, ban/unban.
- **Reports**: xử lý report, ẩn tin / cảnh báo user.
- Auth guard: redirect `/login` nếu không phải admin; layout có sidebar + topbar.

### 2.4 Cấu trúc thư mục (web-admin)
```text
web/                     (repo riêng — shopoo/web/)
├── app/
│   ├── (auth)/login/
│   └── (dashboard)/{page, moderation, listings, users, reports}/
├── components/        (DataTable, StatCard, Chart, ModerationDialog)
├── lib/               (api client, auth, queryClient)
└── types/
```

---

## 3. Lộ trình triển khai (Milestones)

> Frontend nên đi **sau hoặc song song** với backend tương ứng. Có thể dùng mock (MSW) khi backend chưa xong.

**FE-M0 — Setup chung:** khởi tạo 2 app, cấu hình TypeScript, API client + interceptor, types theo contract, `.env`.

**FE-M1 — Auth (cả 2 app):** Login/Register mobile, Login admin. → cần backend M1 (Auth).

**FE-M2 — Listing browse + detail (mobile):** Home/Search/Filter/Detail. → cần backend M2 (Listing).

**FE-M3 — Create listing + upload ảnh (mobile):** multi-step + Media. → cần backend M2+M3.

**FE-M4 — Web Admin dashboard + moderation:** → cần backend M4 (Admin).

**FE-M5 — Chat real-time (mobile):** Socket.io client, ConversationList, ChatRoom. → cần backend M5 (Chat).

**FE-M6 — Push notification (mobile):** FCM token + handle push + deep link. → cần backend M6 (Noti).

**FE-M7 — Hoàn thiện:** loading/empty/error states, skeleton, pull-to-refresh, theming, polish UI.

---

## 4. Việc cần chốt trước khi code

1. **Mobile**: Expo managed (nhanh) hay bare (full native cho FCM)? → Đề xuất Expo + `expo-notifications`; chuyển bare nếu vướng FCM.
2. **State chat/unread**: Zustand store toàn cục + React Query cho history — OK chứ?
3. **Web Admin UI kit**: shadcn/ui (custom, đẹp) hay Ant Design (sẵn component dashboard)? → Đề xuất shadcn/ui kết hợp skill design-taste.
4. **Mock backend**: dùng MSW để FE chạy trước khi BE xong, hay chờ BE từng milestone?
5. **i18n**: app tiếng Việt thuần hay cần đa ngôn ngữ?
