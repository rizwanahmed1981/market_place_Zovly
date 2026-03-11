# Zovly — HyperLocal Commerce Platform

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-009688.svg)](https://fastapi.tiangolo.com/)
[![Flutter](https://img.shields.io/badge/Flutter-3.x-02569B.svg)](https://flutter.dev/)
[![PostgreSQL + PostGIS](https://img.shields.io/badge/PostgreSQL-15+-336791.svg)](https://postgresql.org/)

A **hyperlocal eCommerce platform** connecting buyers and sellers within a 2km radius. Built for South Asian urban/semi-urban markets with a **pickup-first, zero platform fees** model.

---

## 🚀 Core Features

### For Buyers
- **Location-based discovery** — Find shops within 0.5–5km radius using GPS
- **Anonymous negotiation** — Make offers on products without revealing identity
- **Real-time chat** — WebSocket-based anonymous chat with sellers
- **Pickup vouchers** — QR code system for seamless item collection
- **Flash deals** — Get notified about nearby discounts

### For Sellers
- **Shop management** — Create and manage your digital storefront
- **Product catalog** — List up to 500 products with AI-powered categorization
- **Offer system** — Accept, counter, or decline buyer offers
- **Verification dashboard** — CNIC + shop photo verification for trust
- **Flash deal broadcasting** — Send promotional offers to nearby buyers

### For Admins
- **Seller verification** — Manual review queue for new sellers
- **Report management** — Handle flagged content and disputes
- **Analytics dashboard** — Platform-wide metrics and insights

---

## 🏗️ Architecture

```
┌─────────────────┐         ┌─────────────────┐
│   Flutter App   │         │  React Admin    │
│  (iOS + Android)│         │   Dashboard     │
└────────┬────────┘         └────────┬────────┘
         │                           │
         └───────────┬───────────────┘
                     │
         ┌───────────▼───────────┐
         │   FastAPI Backend     │
         │   (Python 3.11+)      │
         └───────────┬───────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐   ┌─────▼─────┐   ┌────▼────┐
│PostgreSQL│   │   Redis   │   │Cloudinary│
│ + PostGIS│   │  (Cache)  │   │ (Storage)│
└─────────┘   └───────────┘   └─────────┘
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|------------|
| **Backend** | Python 3.11+, FastAPI, SQLAlchemy (async), ARQ |
| **Database** | PostgreSQL 15 + PostGIS extension |
| **Cache / Queue** | Redis 7 |
| **Mobile** | Flutter 3.x (Dart) |
| **Admin Panel** | React + Vite + TailwindCSS |
| **Auth** | Firebase Auth (Phone OTP) |
| **File Storage** | Cloudinary |
| **Real-time** | WebSockets (FastAPI native) |
| **AI** | Claude API (content moderation, auto-categorization) |
| **Hosting** | Railway.app + Cloudflare CDN |

---

## 📦 Project Structure

```
ecom-platform/
├── backend/                 # FastAPI monolith
│   ├── app/
│   │   ├── api/            # Versioned routes (/api/v1/)
│   │   ├── core/           # Config, security, dependencies
│   │   ├── models/         # SQLAlchemy ORM models
│   │   ├── schemas/        # Pydantic schemas
│   │   ├── services/       # Business logic
│   │   ├── websockets/     # Real-time chat handlers
│   │   └── workers/        # ARQ background tasks
│   ├── migrations/         # Alembic migrations
│   ├── tests/
│   └── docker-compose.yml
│
├── mobile/                  # Flutter app
│   ├── lib/
│   │   ├── core/           # Theme, router, constants
│   │   ├── features/       # Feature-first structure
│   │   └── shared/         # Widgets, utils, API client
│   └── pubspec.yaml
│
├── admin/                   # React dashboard
│   ├── src/
│   │   ├── pages/          # Seller verification, reports
│   │   └── components/
│   └── package.json
│
├── infra/                   # Deployment configs
│   └── docker-compose.yml
│
└── refrence_docs/          # Project documentation
```

---

## 🔑 Key Business Rules

### Location & Discovery
- **Default search radius:** 2.0 km (configurable: 0.5–5km)
- **Buyer location:** Captured fresh on every app open — never stored permanently
- **Seller location:** Fixed during registration, changes require re-verification
- **Geo-filtering:** Server-side PostGIS queries using `ST_DWithin()`

### Anonymity & Privacy
- Buyers appear as `Buyer#XXXX` (4-digit random) — refreshed per chat session
- Buyer phone numbers **never** exposed to sellers
- Chat logs retained for 90 days for dispute resolution

### Negotiation Flow
- Offer price must be **10%–99%** of listed price
- Offers expire in **24 hours** if no response
- Seller can counter **once** — buyer must then accept or decline
- Accepted offer locks product stock for **2 hours**

### Transaction Model
- **Pickup-only** by default (cash on pickup)
- No payment gateway in MVP
- QR voucher valid for **48 hours**
- Platform charges **zero fees** (Phase 1 & 2)

---

## 🚀 Quick Start

### Prerequisites
- Python 3.11+
- Docker & Docker Compose
- Node.js 18+ (for admin panel)
- Flutter 3.x (for mobile app)

### Backend Setup

```bash
cd backend

# Copy environment variables
cp .env.example .env
# Edit .env with your API keys

# Start services with Docker
docker-compose up -d

# Run migrations
alembic upgrade head

# Start development server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

API docs available at: `http://localhost:8000/docs`

### Mobile App Setup

```bash
cd mobile

# Install dependencies
flutter pub get

# Run on device/emulator
flutter run
```

### Admin Dashboard Setup

```bash
cd admin

# Install dependencies
npm install

# Start development server
npm run dev
```

---

## 📝 Environment Variables

Required environment variables (see `.env.example`):

```bash
# App
APP_ENV=development
SECRET_KEY=your-secret-key
ALLOWED_ORIGINS=http://localhost:3000

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/Zovly
DATABASE_URL_SYNC=postgresql://user:pass@localhost:5432/Zovly

# Redis
REDIS_URL=redis://localhost:6379/0

# Firebase
FIREBASE_PROJECT_ID=your-project-id
FIREBASE_SERVICE_ACCOUNT_JSON={}

# Cloudinary
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret

# Anthropic (AI)
ANTHROPIC_API_KEY=your-api-key

# Admin
ADMIN_SECRET_KEY=your-admin-secret
ADMIN_ALLOWED_IPS=127.0.0.1
```

---

## 🧪 Testing

```bash
cd backend

# Run all tests
pytest

# Run with coverage
pytest --cov=app

# Run specific module tests
pytest tests/test_auth.py -v
```

---

## 📄 API Endpoints

### Authentication
- `POST /api/v1/auth/buyer/send-otp` — Send OTP to buyer phone
- `POST /api/v1/auth/buyer/verify-otp` — Verify OTP and get JWT
- `POST /api/v1/auth/seller/register` — Register new seller
- `POST /api/v1/auth/seller/login` — Seller login

### Discovery
- `GET /api/v1/discovery/shops` — List shops within radius
- `GET /api/v1/discovery/shops/{id}/products` — Get shop products
- `GET /api/v1/discovery/products/search` — Search products by text
- `GET /api/v1/discovery/deals/nearby` — Get active flash deals

### Offers & Negotiation
- `POST /api/v1/offers` — Make an offer on a product
- `POST /api/v1/offers/{id}/respond` — Seller responds to offer
- `GET /api/v1/offers/buyer/me` — Buyer's offer history
- `GET /api/v1/offers/seller/me` — Seller's incoming offers

### Chat (WebSocket)
- `WS /ws/chat/{room_id}?token={jwt}` — Real-time chat connection

### Pickup Vouchers
- `GET /api/v1/vouchers/buyer/me` — Buyer's vouchers
- `GET /api/v1/vouchers/{id}` — Voucher details with QR data
- `POST /api/v1/vouchers/{qr_hash}/verify` — Seller verifies QR

### Admin (IP-restricted)
- `GET /api/v1/admin/sellers/pending` — Sellers awaiting verification
- `POST /api/v1/admin/sellers/{id}/verify` — Approve seller
- `GET /api/v1/admin/reports` — Open reports
- `GET /api/v1/admin/stats` — Dashboard statistics

Full API documentation: `/docs` (Swagger UI)

---

## 🗺️ Roadmap

### Phase 1 — MVP (Month 1-3)
- ✅ Buyer/Seller authentication
- ✅ Seller shop + product CRUD
- ✅ Location-based discovery
- ✅ Anonymous chat system
- ✅ Offer/negotiation flow
- ✅ Pickup voucher + QR system
- ✅ Admin verification dashboard

### Phase 2 — Growth (Month 4-5)
- ⏳ Flash deals / broadcast offers
- ⏳ Buyer wishlist + price drop alerts
- ⏳ Seller analytics dashboard
- ⏳ AI product categorization
- ⏳ Report/dispute system

### Phase 3 — Scale (Month 6+)
- ⏳ Multi-language support (Urdu)
- ⏳ Seller subscription tiers
- ⏳ API rate limit tiers
- ⏳ Horizontal scaling + read replicas

---

## 🔒 Security Features

- **JWT authentication** with role-based scopes (Buyer/Seller/Admin)
- **Rate limiting** (Redis-backed, per-IP and per-endpoint)
- **CORS enforcement** — mobile app origins only in production
- **HTTPS enforced** — HTTP redirects to HTTPS at nginx level
- **Encrypted storage** — CNIC photos encrypted at rest
- **IP whitelisting** — admin panel not publicly accessible
- **Input validation** — file type, size, virus scanning
- **WebSocket sanitization** — HTML stripping, length limits

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📜 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

## 📞 Support

For issues, questions, or contributions, please open a GitHub issue.

---

**Built with ❤️ for South Asian local commerce**
