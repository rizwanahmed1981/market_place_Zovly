---
id: 1
title: zovly-platform-spec
stage: spec
date: 2026-03-11
surface: agent
model: claude-sonnet-4-5-20250929
feature: 1-zovly-platform
branch: main
user: ecomw
command: /sp.specify
labels: ["spec", "platform", "ecommerce"]
links:
  spec: specs/1-zovly-platform/spec.md
  ticket: null
  adr: null
  pr: null
files:
 - specs/1-zovly-platform/spec.md
 - specs/1-zovly-platform/checklists/requirements.md
tests:
---
## Prompt

```
# sp.specification — Zovly Hyperlocal eCommerce Platform

## INSTRUCTION TO CLAUDE CLI

You are building Zovly — a hyperlocal eCommerce platform in Python (FastAPI backend) + Flutter (mobile). The constitution defines architecture, tech stack, business rules, and MCP servers. This specification defines every feature as a buildable task. Build one module at a time. Do not skip steps. Do not invent features not listed here.

---

## MODULE 1 — PROJECT BOOTSTRAP

### Task 1.1 — Backend Initialization
Create the FastAPI project with this exact structure:
```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app factory, middleware, CORS, router registration
│   ├── core/
│   │   ├── config.py            # Pydantic Settings from .env
│   │   ├── database.py          # Async SQLAlchemy engine + session factory
│   │   ├── redis.py             # Redis connection pool
│   │   ├── security.py          # JWT encode/decode, password hash (not needed for OTP but for admin)
│   │   └── dependencies.py      # FastAPI Depends: get_db, get_current_buyer, get_current_seller
│   ├── api/
│   │   └── v1/
│   │       ├── router.py        # Aggregates all v1 routes
│   │       ├── auth.py
│   │       ├── shops.py
│   │       ├── products.py
│   │       ├── discovery.py
│   │       ├── offers.py
│   │       ├── chat.py
│   │       ├── vouchers.py
│   │       └── admin.py
│   ├── models/                  # SQLAlchemy models (one file per entity)
│   ├── schemas/                 # Pydantic schemas (one file per entity)
│   ├── services/                # Business logic (one file per domain)
│   ├── websockets/
│   │   └── chat_handler.py
│   └── workers/
│       └── tasks.py             # ARQ background tasks
├── migrations/
├── tests/
├── .env.example
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

**requirements.txt must include:**
```
fastapi==0.111.0
uvicorn[standard]==0.29.0
sqlalchemy[asyncio]==2.0.30
asyncpg==0.29.0
alembic==1.13.1
pydantic-settings==2.2.1
redis[hiredis]==5.0.4
arq==0.25.0
python-jose[cryptography]==3.3.0
python-multipart==0.0.9
httpx==0.27.0
firebase-admin==6.5.0
cloudinary==1.40.0
anthropic==0.26.0
geoalchemy2==0.15.1
shapely==2.0.4
Pillow==10.3.0
qrcode==7.4.2
websockets==12.0
```

### Task 1.2 — Environment Configuration
`.env.example` must expose ALL required keys:
```
# App
APP_ENV=development
SECRET_KEY=
ALLOWED_ORIGINS=http://localhost:3000

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/Zovly
DATABASE_URL_SYNC=postgresql://user:pass@localhost:5432/Zovly

# Redis
REDIS_URL=redis://localhost:6379/0

# Firebase
FIREBASE_PROJECT_ID=
FIREBASE_SERVICE_ACCOUNT_JSON=

# Cloudinary
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

# Anthropic
ANTHROPIC_API_KEY=

# Admin
ADMIN_SECRET_KEY=
ADMIN_ALLOWED_IPS=127.0.0.1
```

### Task 1.3 — Docker Compose
`docker-compose.yml` must spin up: `postgres` (with PostGIS), `redis`, `backend` (FastAPI), `worker` (ARQ). Include health checks. PostGIS image: `postgis/postgis:15-3.4`.

---

## MODULE 2 — DATABASE SCHEMA

### Task 2.1 — Create All SQLAlchemy Models

**models/user.py**
```
Table: users
- id: UUID PK
- phone_number: String(20) UNIQUE NOT NULL
- role: Enum('BUYER', 'SELLER', 'ADMIN')
- is_active: Boolean DEFAULT TRUE
- firebase_uid: String UNIQUE
- created_at, updated_at: DateTime
```

**models/seller_profile.py**
```
Table: seller_profiles
- id: UUID PK
- user_id: FK → users
- business_name: String(200) NOT NULL
- cnic_hash: String(64)              # SHA-256 of raw CNIC
- cnic_photo_url: String             # Cloudinary private URL
- selfie_photo_url: String
- shop_front_photo_url: String
- verification_status: Enum('PENDING','VERIFIED','REJECTED','SUSPENDED')
- rejection_reason: Text NULLABLE
- verified_at: DateTime NULLABLE
- verified_by: UUID FK → users NULLABLE
```

**models/shop.py**
```
Table: shops
- id: UUID PK
- seller_id: FK → seller_profiles
- name: String(200) NOT NULL
- description: Text
- category: String(100)
- geo_location: Geometry(POINT, 4326)  # PostGIS
- address_text: String(500)
- cover_image_url: String
- operating_hours: JSONB             # {"mon": "9:00-21:00", ...}
- is_active: Boolean DEFAULT FALSE   # only true after seller verified
- search_radius_km: Float DEFAULT 2.0
- created_at, updated_at
```

**models/product.py**
```
Table: products
- id: UUID PK
- shop_id: FK → shops
- title: String(300) NOT NULL
- description: Text
- price: Numeric(12,2) NOT NULL
- min_offer_price: Numeric(12,2)     # auto-calculated: price * 0.10
- stock_qty: Integer DEFAULT 0
- category: String(100)
- ai_category: String(100)           # Claude-assigned
- images: ARRAY(String)              # Cloudinary URLs
- is_available: Boolean DEFAULT TRUE
- is_flagged: Boolean DEFAULT FALSE  # AI flagged for review
- view_count: Integer DEFAULT 0
- created_at, updated_at
```

**models/offer.py**
```
Table: offers
- id: UUID PK
- buyer_id: FK → users
- product_id: FK → products
- original_price: Numeric(12,2)
- offered_price: Numeric(12,2)
- counter_price: Numeric(12,2) NULLABLE
- status: Enum('PENDING','ACCEPTED','COUNTERED','DECLINED','EXPIRED','COMPLETED')
- buyer_note: String(500)
- seller_note: String(500)
- expires_at: DateTime               # offered_at + 24hrs
- created_at, updated_at
```

**models/chat.py**
```
Table: chat_rooms
- id: UUID PK
- offer_id: FK → offers NULLABLE
- product_id: FK → products
- shop_id: FK → shops
- buyer_id: FK → users
- buyer_alias: String(20)            # Buyer#XXXX - generated at room creation
- is_active: Boolean DEFAULT TRUE
- created_at

Table: messages
- id: UUID PK
- room_id: FK → chat_rooms
- sender_role: Enum('BUYER','SELLER','SYSTEM')
- content: String(1000)
- is_read: Boolean DEFAULT FALSE
- is_flagged: Boolean DEFAULT FALSE
- created_at
```

**models/voucher.py**
```
Table: pickup_vouchers
- id: UUID PK
- offer_id: FK → offers
- qr_hash: String(64) UNIQUE        # SHA-256 of id+secret
- agreed_price: Numeric(12,2)
- status: Enum('PENDING','READY','COLLECTED','CANCELLED','EXPIRED')
- valid_until: DateTime              # created_at + 48hrs
- collected_at: DateTime NULLABLE
- created_at
```

**models/report.py**
```
Table: reports
- id: UUID PK
- reporter_id: FK → users
- target_type: Enum('SELLER','PRODUCT','MESSAGE','SHOP')
- target_id: UUID
- reason: Enum('SCAM','FAKE_PRODUCT','ABUSIVE','WRONG_LOCATION','OTHER')
- description: Text NULLABLE
- status: Enum('OPEN','REVIEWED','RESOLVED','DISMISSED')
- created_at
```

**models/flash_deal.py**
```
Table: flash_deals
- id: UUID PK
- shop_id: FK → shops
- product_id: FK → products
- discount_percent: Integer          # 1-90
- deal_text: String(300)
- starts_at: DateTime
- ends_at: DateTime
- notified_buyer_count: Integer DEFAULT 0
- is_active: Boolean DEFAULT TRUE
```

### Task 2.2 — Alembic Setup
Initialize Alembic with async support. Create initial migration from all models. Include PostGIS extension creation in first migration: `CREATE EXTENSION IF NOT EXISTS postgis;`

### Task 2.3 — PostGIS Index
Create spatial index on `shops.geo_location` and `shops.search_radius_km` for fast radius queries.

---

## MODULE 3 — AUTHENTICATION

### Task 3.1 — Buyer Auth (`/api/v1/auth/buyer/`)

**POST `/send-otp`**
- Body: `{ phone_number: "+92XXXXXXXXXX" }`
- Action: Call Firebase Admin SDK `auth.create_user()` or trigger SMS OTP
- Rate limit: 3 requests per phone per hour (Redis counter)
- Response: `{ success: true, message: "OTP sent" }`

**POST `/verify-otp`**
- Body: `{ phone_number, firebase_id_token }`
- Action: Verify token with Firebase Admin → get/create user in DB with role=BUYER
- Response: `{ access_token, token_type: "bearer", user: { id, phone_number, role } }`

### Task 3.2 — Seller Auth (`/api/v1/auth/seller/`)

**POST `/register`**
- Body: multipart/form-data
  ```
  phone_number, business_name, cnic_number,
  cnic_photo (file), selfie_photo (file), shop_front_photo (file),
  firebase_id_token, shop_lat, shop_lng, address_text
  ```
- Actions in order:
  1. Verify Firebase ID token
  2. Check CNIC hash not already registered
  3. Upload all 3 photos to Cloudinary private bucket
  4. Create user (role=SELLER) + seller_profile (status=PENDING) + shop record
  5. Queue ARQ task: `notify_admin_new_seller_verification`
- Response: `{ success: true, message: "Registration submitted. Await verification 24-48hrs.", seller_id }`

**POST `/login`**
- Body: `{ firebase_id_token }`
- Action: Verify token → find SELLER user → check verification_status
- If PENDING: return 403 with `{ status: "PENDING_VERIFICATION" }`
- If REJECTED: return 403 with `{ status: "REJECTED", reason }`
- If VERIFIED: return JWT
- Response: `{ access_token, shop: { id, name, is_active } }`

### Task 3.3 — JWT Structure
```python
# Buyer token payload
{ "sub": user_id, "role": "BUYER", "exp": now+7days }

# Seller token payload
{ "sub": user_id, "role": "SELLER", "shop_id": shop_id, "exp": now+30days }

# Admin token payload
{ "sub": admin_id, "role": "ADMIN", "exp": now+8hrs }
```

---

## MODULE 4 — DISCOVERY (Core Feature)

### Task 4.1 — Shop Discovery (`/api/v1/discovery/`)

**GET `/shops`**
- Query params: `lat`, `lng`, `radius_km` (default 2.0, max 5.0), `category` (optional), `cursor` (pagination)
- Auth: Buyer JWT required
- PostGIS query:
  ```sql
  SELECT shops.*,
    ST_Distance(geo_location::geography, ST_MakePoint(:lng, :lat)::geography) as distance_meters
  FROM shops
  JOIN seller_profiles ON shops.seller_id = seller_profiles.id
  WHERE seller_profiles.verification_status = 'VERIFIED'
    AND shops.is_active = TRUE
    AND ST_DWithin(geo_location::geography, ST_MakePoint(:lng, :lat)::geography, :radius_meters)
  ORDER BY distance_meters ASC
  LIMIT 20
  ```
- Response: `{ shops: [...], next_cursor, total_in_radius }`
- Each shop includes: id, name, category, distance_meters, cover_image, is_open_now, product_count

**GET `/shops/{shop_id}/products`**
- Auth: Buyer JWT
- Query: `cursor`, `category`, `min_price`, `max_price`, `available_only`
- Response: paginated products list

**GET `/products/search`**
- Query: `q` (text search), `lat`, `lng`, `radius_km`, `cursor`
- Searches product titles/descriptions within radius shops only
- Uses PostgreSQL full-text search `to_tsvector`

**GET `/deals/nearby`**
- Query: `lat`, `lng`, `radius_km`
- Returns active flash_deals from shops within radius
- Ordered by discount_percent DESC

---

## MODULE 5 — SHOP & PRODUCT MANAGEMENT (Seller)

### Task 5.1 — Shop Management (`/api/v1/shops/`)
All routes require Seller JWT.

**GET `/me`** — Get own shop details

**PUT `/me`** — Update shop: name, description, operating_hours, cover_image
- If location change > 500m → set is_active=FALSE, queue re-verification task

**POST `/me/toggle`** — Toggle shop is_active (open/closed for the day)

### Task 5.2 — Product CRUD (`/api/v1/products/`)
All routes require Seller JWT. Products belong to seller's shop only.

**POST `/`** — Create product
- Body: multipart with images (max 5 files, 5MB each), title, description, price, stock_qty, category
- Actions:
  1. Validate: price > 0, stock_qty >= 0
  2. Upload images to Cloudinary public bucket
  3. Set min_offer_price = price * 0.10
  4. Save product with is_available=TRUE
  5. Queue ARQ task: `ai_process_product(product_id)` → Claude API for category + moderation
- Response: product object

**GET `/`** — List seller's own products (all, including unavailable)

**GET `/{product_id}`** — Single product detail

**PUT `/{product_id}`** — Update product fields (partial update, PATCH behavior)
- Price change recalculates min_offer_price

**DELETE `/{product_id}`** — Soft delete (set is_available=FALSE, keep for order history)

**POST `/{product_id}/toggle-availability`** — Toggle is_available

**POST `/{product_id}/flash-deal`** — Create flash deal
- Body: `{ discount_percent, deal_text, duration_hours }` (duration max 24h)
- Queue ARQ task: `broadcast_flash_deal(deal_id)` → FCM to nearby buyers

---

## MODULE 6 — NEGOTIATION / OFFER SYSTEM

### Task 6.1 — Offer Flow (`/api/v1/offers/`)

**POST `/`** — Buyer makes offer
- Auth: Buyer JWT
- Body: `{ product_id, offered_price, buyer_note }`
- Validations:
  - Product must be is_available=TRUE
  - offered_price must be between product.min_offer_price and product.price
  - Buyer cannot have existing PENDING/COUNTERED offer on same product
  - Seller shop must be within buyer's current radius (pass lat/lng)
- Actions:
  1. Create offer (status=PENDING, expires_at=now+24hrs)
  2. Auto-create chat_room linked to offer (generate buyer_alias=`Buyer#XXXX`)
  3. Send FCM push to seller: "New offer on [product_name]"
  4. Return: `{ offer, chat_room_id }`

**GET `/buyer/me`** — Buyer's offer history (paginated)

**GET `/seller/me`** — Seller's incoming offers (paginated, filter by status)

**POST `/{offer_id}/respond`** — Seller responds to offer
- Auth: Seller JWT (must own the product)
- Body: `{ action: "ACCEPT"|"COUNTER"|"DECLINE", counter_price?, seller_note? }`
- If ACCEPT:
  - Set offer.status=ACCEPTED
  - Lock product stock: decrement stock_qty by 1
  - Create pickup_voucher (status=PENDING, valid_until=now+48hrs)
  - Send FCM to buyer: "Offer accepted! Pickup within 48hrs"
- If COUNTER:
  - counter_price must be between offered_price and original_price
  - Set offer.status=COUNTERED
  - Send FCM to buyer with counter price
- If DECLINE:
  - Set offer.status=DECLINED
  - Release any stock lock
  - Send FCM to buyer

**POST `/{offer_id}/buyer-respond`** — Buyer responds to counter
- Auth: Buyer JWT
- Body: `{ action: "ACCEPT"|"DECLINE" }`
- If ACCEPT: same as seller ACCEPT above

---

## MODULE 7 — REAL-TIME ANONYMOUS CHAT

### Task 7.1 — WebSocket Chat Handler

**WebSocket endpoint:** `GET /ws/chat/{room_id}`
- Query param: `?token={jwt}` (WebSocket can't send headers)
- On connect:
  1. Validate JWT
  2. Verify user is buyer OR seller of this chat_room
  3. Add connection to Redis pub/sub channel: `chat:{room_id}`
  4. Send last 50 messages as connection handshake
- On message receive:
  1. Validate: content max 1000 chars, not empty
  2. Sanitize: strip HTML tags
  3. Save to DB (messages table)
  4. Publish to Redis channel: `PUBLISH chat:{room_id} {message_json}`
  5. Queue ARQ task: `moderate_message(message_id)` → Claude toxicity check
  6. All subscribers in channel receive message
- On disconnect: remove from channel, update last_seen

**Message JSON structure:**
```json
{
  "id": "uuid",
  "room_id": "uuid",
  "sender_role": "BUYER|SELLER",
  "sender_display": "Buyer#4821 OR shop_name",
  "content": "text",
  "timestamp": "ISO8601",
  "is_read": false
}
```

### Task 7.2 — Chat REST endpoints (`/api/v1/chat/`)

**GET `/rooms`** — List user's chat rooms (buyer: their rooms, seller: their shop's rooms)

**GET `/rooms/{room_id}/messages`** — Paginated message history (cursor-based, newest first)

**POST `/rooms/{room_id}/read`** — Mark messages as read

**DELETE `/rooms/{room_id}`** — Close chat room (both parties can close; sets is_active=FALSE)

---

## MODULE 8 — PICKUP VOUCHER SYSTEM

### Task 8.1 — Voucher endpoints (`/api/v1/vouchers/`)

**GET `/buyer/me`** — Buyer's vouchers (filter by status)

**GET `/{voucher_id}`** — Voucher detail + QR code data
- Response includes: `qr_data` (string to render as QR), shop address, product title, agreed_price, valid_until

**POST `/{qr_hash}/verify`** — Seller scans and verifies voucher
- Auth: Seller JWT
- Action:
  1. Find voucher by qr_hash
  2. Validate: status=READY or PENDING, not expired, belongs to seller's shop
  3. Set status=COLLECTED, collected_at=now
  4. Set offer.status=COMPLETED
  5. Increment product sold_count
- Response: `{ success: true, buyer_alias, product_title, agreed_price }`

**POST `/{voucher_id}/mark-ready`** — Seller marks item as ready for pickup
- Sets voucher.status=READY
- Sends FCM to buyer: "Your item is ready for pickup at [shop_name]"

**POST `/{voucher_id}/cancel`** — Cancel voucher (buyer or seller)
- Set status=CANCELLED
- Restore product stock_qty +1
- Set offer.status back to DECLINED or create CANCELLED status

---

## MODULE 9 — FLASH DEALS & NOTIFICATIONS

### Task 9.1 — Flash Deal Broadcasting (ARQ Worker Task)
```
task: broadcast_flash_deal(deal_id)
Steps:
1. Load flash_deal with shop.geo_location
2. Query all buyer FCM tokens where last_known_radius intersects shop location
   (NOTE: buyers don't store location — use opt-in "notify me about nearby deals" preference)
3. Send FCM multicast to max 1000 tokens per batch
4. Update flash_deal.notified_buyer_count
```

**Buyer opt-in for deal notifications:**
- Buyer can toggle: `notify_nearby_deals: bool` in their profile
- If TRUE: when buyer opens app, their FCM token + current location saved to Redis with 2hr TTL
- Flash deal worker queries Redis for active buyer tokens within radius

### Task 9.2 — Notification Types (FCM)
All FCM payloads include `notification_type` field for client routing:
```
NEW_OFFER          → Seller: buyer made offer
OFFER_ACCEPTED     → Buyer: seller accepted
OFFER_COUNTERED    → Buyer: seller countered with price
OFFER_DECLINED     → Buyer: offer declined
OFFER_EXPIRED      → Buyer: 24hr offer expired
VOUCHER_READY      → Buyer: item ready for pickup
VOUCHER_EXPIRING   → Buyer: voucher expires in 6hrs (scheduled task)
FLASH_DEAL         → Buyer: nearby deal available
VERIFICATION_DONE  → Seller: account verified/rejected
```

---

## MODULE 10 — ADMIN PANEL BACKEND

### Task 10.1 — Admin Routes (`/api/v1/admin/`)
IP-restricted middleware. Admin JWT required.

**GET `/sellers/pending`** — List sellers awaiting verification
**GET `/sellers/{seller_id}`** — Full seller detail with photos
**POST `/sellers/{seller_id}/verify`** — Approve seller
- Body: `{}`
- Sets seller_profile.verification_status=VERIFIED
- Sets shop.is_active=TRUE
- Sends FCM to seller: account verified

**POST `/sellers/{seller_id}/reject`** — Reject seller
- Body: `{ reason: string }`
- Sets status=REJECTED with reason
- Sends FCM to seller with reason

**POST `/sellers/{seller_id}/suspend`** — Suspend active seller

**GET `/reports`** — List open reports (filter by target_type, status)
**POST `/reports/{report_id}/resolve`** — Resolve report with action

**GET `/stats`** — Dashboard: total users, shops, products, active offers, today's vouchers

---

## MODULE 11 — AI BACKGROUND TASKS (Claude API)

### Task 11.1 — ARQ Worker Tasks (`workers/tasks.py`)

**`ai_process_product(product_id)`**
```
1. Load product (title, description, images[0])
2. Call Claude API:
   System: "You are a product categorization assistant for a local marketplace."
   User: "Product: {title}. Description: {description}.
          Task 1: Assign ONE category from: [Electronics, Clothing, Food,
          Furniture, Books, Beauty, Sports, Home, Toys, Other]
          Task 2: Is this product image/description inappropriate? (yes/no)
          Task 3: Rate description quality 1-5, rewrite if score < 3.
          Respond in JSON: {category, is_inappropriate, description_rewrite}"
3. Update product: ai_category, is_flagged (if inappropriate),
   description (if rewrite provided)
```

**`moderate_message(message_id)`**
```
1. Load message content
2. Call Claude API with simple toxicity check prompt
3. If toxic score > 0.8: set message.is_flagged=TRUE,
   notify admin via system message in room
```

**`notify_admin_new_seller_verification(seller_id)`**
```
1. Load seller details
2. Send email/FCM to admin with seller review link
3. Log notification sent
```

---

## MODULE 12 — REPORTING & TRUST SYSTEM

### Task 12.1 — Report endpoints (`/api/v1/reports/`)

**POST `/`** — Submit report (Auth: any authenticated user)
- Body: `{ target_type, target_id, reason, description? }`
- After save: check if target seller now has >= 3 OPEN reports
- If yes: queue `auto_suspend_check(seller_id)` worker task

**GET `/my`** — Requester's submitted reports

### Task 12.2 — Auto-suspension Worker Task
```
task: auto_suspend_check(seller_id)
1. Count OPEN reports against seller in last 30 days
2. If >= 3: set seller.verification_status=SUSPENDED
3. Notify admin for review
4. Send FCM to seller: "Account suspended pending review"
```

---

## MODULE 13 — FLUTTER MOBILE APP STRUCTURE

### Task 13.1 — App Architecture
Use: Flutter 3.x + Riverpod (state) + GoRouter (navigation) + Dio (HTTP) + web_socket_channel + google_maps_flutter + firebase_messaging + flutter_local_notifications

```
mobile/lib/
├── main.dart
├── core/
│   ├── router.dart           # GoRouter config with auth guards
│   ├── theme.dart            # App color scheme, typography
│   ├── constants.dart        # API base URL, map defaults
│   └── di.dart               # Riverpod provider scope
├── features/
│   ├── auth/
│   │   ├── screens/          # BuyerLoginScreen, SellerRegisterScreen
│   │   ├── providers/        # authStateProvider, userProvider
│   │   └── services/         # FirebaseAuthService
│   ├── discovery/
│   │   ├── screens/          # MapDiscoveryScreen, ShopListScreen
│   │   ├── widgets/          # ShopCard, MapMarker, RadiusSlider
│   │   └── providers/        # nearbyShopsProvider (location-aware)
│   ├── shop/
│   │   ├── screens/          # ShopDetailScreen, ProductDetailScreen
│   │   │                     # SellerDashboardScreen, ProductManageScreen
│   │   ├── widgets/          # ProductCard, ProductForm, ImagePicker
│   │   └── providers/
│   ├── negotiation/
│   │   ├── screens/          # MakeOfferScreen, OfferListScreen
│   │   │                     # OfferDetailScreen (seller view)
│   │   └── providers/
│   ├── chat/
│   │   ├── screens/          # ChatRoomScreen, ChatListScreen
│   │   ├── services/         # WebSocketChatService (singleton)
│   │   └── providers/
│   ├── pickup/
│   │   ├── screens/          # VoucherScreen (QR display), QRScannerScreen (seller)
│   │   └── providers/
│   └── profile/
│       ├── screens/          # BuyerProfileScreen, SellerProfileScreen
│       └── providers/
└── shared/
    ├── api/
    │   ├── api_client.dart   # Dio instance with auth interceptor
    │   └── endpoints.dart    # All API endpoint strings
    ├── models/               # Dart data classes (freezed)
    └── widgets/              # LoadingSpinner, ErrorState, EmptyState, etc.
```

### Task 13.2 — Key Mobile Features to Implement

**Location Flow:**
- On app launch: request location permission → get GPS → store in memory (NOT persistent)
- Pass lat/lng as query params on every discovery API call
- Show radius slider (0.5 to 5km) on discovery screen

**Map Screen:**
- Google Maps with shop markers within radius
- Marker tap → bottom sheet with shop summary + "Visit Shop" button
- My location button
- Radius circle overlay on map

**Seller Registration Flow:**
- Multi-step form: Phone OTP → Business Info → Document Upload → Shop Location Pin
- Location pin: open map, tap to place pin, confirm address
- Image uploads: camera or gallery picker for all 3 photos
- "Awaiting verification" screen after submit

**Anonymous Chat:**
- WebSocket connection managed as singleton service
- Reconnect logic with exponential backoff
- Show `Buyer#XXXX` as sender name in all messages
- Typing indicator via WebSocket event

**QR Voucher:**
- Use `qr_flutter` package to render QR from `qr_data` string
- Full-screen QR for easy scanning
- Seller uses camera to scan buyer's QR via `mobile_scanner` package

---

## MODULE 14 — ADMIN WEB DASHBOARD

### Task 14.1 — React Admin Dashboard Structure
```
admin/src/
├── pages/
│   ├── Login.jsx             # Admin secret key login
│   ├── Dashboard.jsx         # Stats overview
│   ├── SellerVerification/
│   │   ├── Queue.jsx         # List of PENDING sellers
│   │   └── Review.jsx        # Full seller review with photo viewer
│   ├── Reports/
│   │   ├── ReportList.jsx
│   │   └── ReportDetail.jsx
│   └── Sellers/
│       └── SellerList.jsx    # All sellers with status filter
├── components/
│   ├── PhotoViewer.jsx       # Full-screen CNIC/selfie viewer
│   └── StatusBadge.jsx
└── api/
    └── adminApi.js
```

---

## MODULE 15 — SECURITY & MIDDLEWARE

### Task 15.1 — Middleware Stack (FastAPI)

Implement in `main.py` in this order:
1. `TrustedHostMiddleware` — only allow configured hosts
2. `CORSMiddleware` — ALLOWED_ORIGINS from env
3. Custom `RateLimitMiddleware` — Redis-backed, per IP, per endpoint override
4. Custom `AdminIPWhitelistMiddleware` — block `/api/v1/admin/*` for non-whitelisted IPs
5. `GZipMiddleware` — compress responses > 1000 bytes

### Task 15.2 — Rate Limits
```
/api/v1/auth/*/send-otp    → 3 per phone per hour
/api/v1/auth/*             → 20 per IP per minute
/api/v1/discovery/*        → 60 per token per minute
/api/v1/offers/            → 10 POST per token per hour (prevent spam offers)
/ws/chat/*                 → 1 connection per token at a time
/api/v1/admin/*            → 200 per IP per minute
```

---

## BUILD ORDER FOR CLAUDE CLI

Execute modules in this exact sequence to avoid dependency issues:

```
1.  Module 1  → Project bootstrap, docker, requirements
2.  Module 2  → Database models + migrations
3.  Module 3  → Authentication (backend)
4.  Module 10 → Admin routes (needed to verify sellers before testing)
5.  Module 4  → Discovery
6.  Module 5  → Shop & Product management
7.  Module 6  → Offer/Negotiation system
8.  Module 7  → WebSocket chat
9.  Module 8  → Pickup vouchers
10. Module 9  → Flash deals & notifications
11. Module 11 → AI background tasks
12. Module 12 → Reports
13. Module 15 → Security middleware
14. Module 13 → Flutter mobile app
15. Module 14 → Admin web dashboard
```

---

## TESTING REQUIREMENTS

Each module must include:
- `tests/test_{module}.py` with pytest-asyncio
- At minimum: happy path test + auth failure test + validation failure test
- Use `httpx.AsyncClient` with `ASGITransport` for API tests
- Use `pytest-mock` to mock Firebase, Cloudinary, Anthropic API calls
- Fixtures in `tests/conftest.py`: async DB session, test buyer token, test seller token

---

## DEPLOYMENT CHECKLIST (Railway.app)

```
Services to deploy:
1. postgres  → Railway PostgreSQL plugin (enable PostGIS via init SQL)
2. redis     → Railway Redis plugin
3. backend   → Dockerfile, expose port 8000
4. worker    → Same Dockerfile, CMD override to run ARQ worker

Environment variables: Set all .env.example keys in Railway dashboard
Domain: Configure custom domain with SSL (Railway auto-provisions)
```
## Response snapshot

The feature specification for the Zovly Hyperlocal eCommerce Platform has been successfully created. This includes:

1. A comprehensive specification file at specs/1-zovly-platform/spec.md
2. A quality checklist at specs/1-zovly-platform/checklists/requirements.md
3. All required sections completed including:
   - User scenarios and testing
   - Functional requirements
   - Key entities
   - Success criteria
   - Edge cases

The specification captures all the core functionality described in the original feature description, including:
- Buyer discovery and purchase flows
- Seller registration and verification
- Admin oversight capabilities
- Privacy-preserving anonymous chat
- Location-based discovery with 2km radius
- Product offering and negotiation system
- Pickup voucher management
- AI-powered moderation and categorization

All requirements are testable and measurable, with technology-agnostic success criteria.
## Outcome

- ✅ Impact: Complete feature specification for Zovly Hyperlocal eCommerce Platform
- 🧪 Tests: Quality checklist validation passed
- 📁 Files: Created specs/1-zovly-platform/spec.md and specs/1-zovly-platform/checklists/requirements.md
- 🔁 Next prompts: Ready for planning with /sp.plan
- 🧠 Reflection: The specification covers all core requirements from the user's detailed specification

## Evaluation notes (flywheel)

- Failure modes observed: None
- Graders run and results (PASS/FAIL): All validations passed
- Prompt variant (if applicable): N/A
- Next experiment (smallest change to try): Begin planning with /sp.plan