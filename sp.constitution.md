# sp.constitution — HyperLocal Commerce Platform

## PROJECT IDENTITY

**Project Name:** Zovly (or placeholder: `Zovly_APP`)
**Type:** Hyperlocal eCommerce mobile + backend platform
**Primary Market:** South Asian urban/semi-urban markets (Pakistan-first)
**Core Philosophy:** Physical proximity commerce — buyers and sellers who are geographically close transact with zero platform fees, default pickup, and optional delivery at their own risk.

---

## SYSTEM ARCHITECTURE OVERVIEW

```
Zovly/
├── backend/                  # FastAPI Python monolith (Phase 1)
│   ├── app/
│   │   ├── api/              # Route handlers (versioned: /api/v1/)
│   │   ├── core/             # Config, security, dependencies
│   │   ├── models/           # SQLAlchemy ORM models
│   │   ├── schemas/          # Pydantic request/response schemas
│   │   ├── services/         # Business logic layer
│   │   ├── websockets/       # Real-time chat handlers
│   │   └── workers/          # Background tasks (Celery or ARQ)
│   ├── migrations/           # Alembic migrations
│   ├── tests/
│   └── requirements.txt
│
├── mobile/                   # Flutter (iOS + Android)
│   ├── lib/
│   │   ├── core/             # Theme, constants, router
│   │   ├── features/         # Feature-first folder structure
│   │   │   ├── auth/
│   │   │   ├── discovery/    # Map + shop list
│   │   │   ├── shop/         # Shop profile + products
│   │   │   ├── negotiation/  # Offer/counter-offer flow
│   │   │   ├── chat/         # Anonymous WebSocket chat
│   │   │   └── pickup/       # Voucher + QR system
│   │   └── shared/           # Widgets, utils, API client
│   └── pubspec.yaml
│
├── admin/                    # Simple React web dashboard
│   ├── src/
│   │   ├── pages/
│   │   │   ├── sellers/      # Verification queue
│   │   │   ├── reports/      # Flagged content
│   │   │   └── analytics/
│   └── package.json
│
└── infra/
    ├── docker-compose.yml
    ├── nginx.conf
    └── .env.example
```

---

## TECHNOLOGY STACK — LOCKED DECISIONS

| Layer | Technology | Reason |
|---|---|---|
| Backend Language | Python 3.11+ | Developer's expertise |
| Web Framework | FastAPI | Async, fast, auto-docs |
| ORM | SQLAlchemy 2.0 (async) | Python standard |
| Database | PostgreSQL 15 + PostGIS extension | Geo queries (2km radius) |
| Cache / Pub-Sub | Redis 7 | Real-time chat relay, sessions |
| Real-time | WebSockets (FastAPI native) | Anonymous chat, notifications |
| Auth - Buyers | Firebase Auth (Phone OTP) | Simplest OTP flow |
| Auth - Sellers | Custom JWT + manual verification | CNIC + shop photo review |
| File Storage | Cloudinary | Product/shop photos, CNIC uploads |
| Push Notifications | Firebase Cloud Messaging (FCM) | Cross-platform |
| Mobile | Flutter 3.x (Dart) | Single codebase iOS + Android |
| Admin Panel | React + Vite + TailwindCSS | Lightweight |
| Background Jobs | ARQ (async Redis Queue) | Python async native |
| AI Integration | Claude API (claude-sonnet-4-20250514) | Content moderation, auto-categorization |
| Hosting | Railway.app (backend) + Cloudflare (CDN) | Cost-effective |
| Geo Library | PostGIS ST_DWithin() | Accurate 2km radius queries |

---

## MCP SERVERS REQUIRED

Claude CLI must have these MCP servers installed and configured in the project:

```json
// .mcp.json (project root)
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/Zovly"]
    },
    "redis": {
      "command": "npx",
      "args": ["-y", "mcp-server-redis"],
      "env": {
        "REDIS_URL": "${REDIS_URL}"
      }
    },
    "cloudinary": {
      "command": "npx",
      "args": ["-y", "mcp-cloudinary"],
      "env": {
        "CLOUDINARY_URL": "${CLOUDINARY_URL}"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

**Install commands:**
```bash
# Core MCP servers
npx @modelcontextprotocol/server-postgres
npx @modelcontextprotocol/server-filesystem
npx mcp-server-redis

# Optional but recommended
npx @modelcontextprotocol/server-github     # version control
npx mcp-server-cloudinary                   # file/image management
```

---

## DOMAIN MODEL — CORE ENTITIES

```
User (abstract base)
├── Buyer         → phone_number, anonymous_alias, location (transient GPS)
└── Seller        → business_name, cnic_number, shop_location (FIXED Point), 
                    verification_status, shop_photo, cnic_photo

Shop              → seller_id, name, category, geo_location (PostGIS POINT), 
                    radius_km=2, is_active, operating_hours

Product           → shop_id, title, description, price, stock_qty, 
                    category, images[], is_available, created_at

Offer             → buyer_id, product_id, offered_price, status 
                    [PENDING|ACCEPTED|COUNTERED|DECLINED|EXPIRED], 
                    counter_price, expires_at

ChatRoom          → offer_id OR product_id, buyer_alias, seller_id, 
                    is_active, created_at

Message           → room_id, sender_role [BUYER|SELLER], content, 
                    timestamp, is_read

PickupVoucher     → offer_id, qr_code_hash, status [PENDING|READY|COLLECTED|CANCELLED], 
                    valid_until

Report            → reporter_id, target_type, target_id, reason, status
```

---

## BUSINESS RULES — NON-NEGOTIABLE

### Location Rules
- Buyer GPS location is captured fresh on every app open — never stored permanently
- Seller shop location is set ONCE during registration and changes require re-verification
- ALL geo filtering happens server-side using PostGIS — client sends raw coordinates only
- Default search radius: **2.0 km** — configurable per-query between 0.5km and 5km max
- Sellers outside buyer's radius are completely invisible — no pagination trick to bypass this

### Authentication Rules
- Buyer signup: Firebase Phone OTP only — no email, no password
- Seller signup: Phone OTP + CNIC front photo + selfie with CNIC + shop front photo + physical address pin
- Seller account starts as `PENDING_VERIFICATION` — manually reviewed by admin before going LIVE
- Seller re-verification required if: shop location changes by >500m, CNIC mismatch flagged

### Anonymity Rules
- Buyer is always shown to seller as `Buyer#[4-digit-random]` — refreshed per chat session
- Buyer real phone number NEVER exposed to seller through any API endpoint
- Platform retains chat logs for 90 days for dispute resolution — disclosed in ToS

### Negotiation Rules
- Offer price must be between 10% and 99% of listed price (no insulting offers, no over-bids)
- Offer expires in 24 hours if no seller response
- Seller can counter once — buyer must then Accept or Decline (no infinite loop)
- Accepted offer locks product stock for 2 hours for pickup voucher redemption
- Seller can broadcast a "flash discount" to all buyers currently within 2km radius

### Transaction Rules
- Platform default = **PICKUP ONLY** from physical shop location
- If buyer/seller mutually agree to delivery via chat — platform ToS explicitly states zero liability
- No payment gateway in MVP — cash on pickup
- Pickup voucher = QR code generated on offer acceptance, valid 48 hours

### Content Rules
- All product photos auto-scanned via Claude API for inappropriate content before going live
- Products auto-categorized by Claude API on upload (saves seller time)
- Sellers can list max 500 products per shop in MVP
- 3 unresolved reports against a seller = automatic account suspension pending review

---

## API DESIGN PRINCIPLES

- All routes versioned: `/api/v1/`
- JWT Bearer token in Authorization header for all protected routes
- Buyer token ≠ Seller token (different scopes in JWT payload)
- All geo endpoints accept: `lat`, `lng`, `radius_km` query params
- Pagination: cursor-based (not offset) for product/shop lists
- WebSocket endpoint: `wss://api.Zovly.com/ws/chat/{room_id}?token={jwt}`
- All responses follow envelope: `{ success, data, error, meta }`
- Rate limiting: 100 req/min per IP, 10 req/min for OTP endpoints

---

## SECURITY CONSTITUTION

- CNIC photos stored encrypted at rest in Cloudinary private bucket — never public URL
- Seller CNIC number stored as SHA-256 hash in DB — raw number only visible to admin
- All WebSocket messages sanitized before storage (strip HTML, check length max 1000 chars)
- Admin panel IP-whitelisted — not publicly accessible
- All file uploads validated: type (jpg/png/webp only), size (max 5MB), virus-scanned
- Environment secrets via `.env` — never hardcoded, never in git
- HTTPS enforced everywhere — HTTP redirects to HTTPS at nginx level
- CORS: mobile app origin only in production

---

## AI INTEGRATION POINTS (Claude API)

| Trigger | Input | Output | Purpose |
|---|---|---|---|
| Product upload | Photo + title | Category suggestion, inappropriate flag | Auto-categorize, moderate |
| New seller CNIC | CNIC photo | Legibility check, flag if fake-looking | Pre-screen before human review |
| Chat message | Message text | Toxicity score | Auto-flag abusive messages |
| Seller description | Raw text | Clean formatted description | Help low-literacy sellers |
| Flash deal | Seller inputs | Optimized deal text for buyers | Marketing copy |

All Claude API calls are **async background tasks** — they never block the main request flow.

---

## PHASE ROADMAP

```
Phase 1 — MVP (Month 1-3)
  ✓ Buyer/Seller auth
  ✓ Seller shop + product CRUD
  ✓ Location-based discovery (map + list)
  ✓ Basic anonymous chat
  ✓ Offer/negotiation flow
  ✓ Pickup voucher + QR
  ✓ Admin verification dashboard

Phase 2 — Growth (Month 4-5)
  ✓ Flash deals / broadcast offers
  ✓ Buyer wishlist + price drop alerts
  ✓ Seller analytics dashboard
  ✓ AI product categorization live
  ✓ Report/dispute system

Phase 3 — Scale (Month 6+)
  ✓ Multi-language (Urdu support)
  ✓ Seller subscription tiers (featured listings)
  ✓ API rate limit tiers
  ✓ Horizontal scaling + read replicas
```

---

## WHAT THIS PLATFORM WILL NEVER DO (Constraints)

- Never store buyer location permanently
- Never charge transaction fees in Phase 1 or Phase 2
- Never act as payment intermediary
- Never guarantee delivery or product quality
- Never expose buyer identity to seller
- Never allow shops to appear outside their registered area
- Never allow a seller to operate more than 3 shops per verified identity
