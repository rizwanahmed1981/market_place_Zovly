<!--
Sync Impact Report:
- Version change: N/A → 1.0.0 (initial version)
- Added sections: All principles and governance rules derived from user input
- Templates requiring updates: ⚠ pending review of plan-template.md, spec-template.md, tasks-template.md
- Follow-up TODOs: None
-->
# Zovly Constitution

## Core Principles

### I. Location-Based Proximity Commerce
Physical proximity commerce — buyers and sellers who are geographically close transact with zero platform fees, default pickup, and optional delivery at their own risk. This defines the core market approach focusing on South Asian urban/semi-urban markets with 2km search radius as the default.

### II. Privacy-First Architecture
Buyer GPS location is captured fresh on every app open but never stored permanently. Buyer real phone number is NEVER exposed to seller through any API endpoint. All geo filtering happens server-side using PostGIS — client sends raw coordinates only. This ensures user privacy protection by design.

### III. Anonymity-Preserving Transactions
Buyer is always shown to seller as `Buyer#[4-digit-random]` — refreshed per chat session. Anonymous WebSocket chat system ensures user identities remain protected throughout the negotiation process. Platform retains chat logs for 90 days for dispute resolution as disclosed in Terms of Service.

### IV. Constraint-Led Development (NON-NEGOTIABLE)
Strict business rules guide development: never store buyer location permanently, never charge transaction fees in Phase 1 or 2, never act as payment intermediary, never guarantee delivery or product quality, never expose buyer identity to seller. These constraints ensure the platform remains true to its core philosophy.

### V. Geo-Aware Services
ALL geo filtering happens server-side using PostGIS ST_DWithin(). Default search radius is 2.0 km — configurable per-query between 0.5km and 5km max. Sellers outside buyer's radius are completely invisible. PostgreSQL 15 with PostGIS extension is the foundational technology for this capability.

### VI. Asynchronous AI Integration
All Claude API calls are **async background tasks** — they never block the main request flow. AI integration occurs at specific triggers: product upload (auto-categorization and moderation), CNIC verification (pre-screening), chat message (toxicity scoring), and content optimization (descriptions and marketing copy).

## Security Requirements
- CNIC photos stored encrypted at rest in Cloudinary private bucket — never public URL
- Seller CNIC number stored as SHA-256 hash in DB — raw number only visible to admin
- All WebSocket messages sanitized before storage (strip HTML, check length max 1000 chars)
- Admin panel IP-whitelisted — not publicly accessible
- All file uploads validated: type (jpg/png/webp only), size (max 5MB), virus-scanned
- Environment secrets via `.env` — never hardcoded, never in git
- HTTPS enforced everywhere — HTTP redirects to HTTPS at nginx level
- CORS: mobile app origin only in production

## Development Workflow
- Buyer signup: Firebase Phone OTP only — no email, no password
- Seller signup: Phone OTP + CNIC front photo + selfie with CNIC + shop front photo + physical address pin
- Seller account starts as `PENDING_VERIFICATION` — manually reviewed by admin before going LIVE
- Seller re-verification required if: shop location changes by >500m, CNIC mismatch flagged
- All routes versioned: `/api/v1/`
- JWT Bearer token in Authorization header for all protected routes
- Buyer token ≠ Seller token (different scopes in JWT payload)
- All geo endpoints accept: `lat`, `lng`, `radius_km` query params
- Pagination: cursor-based (not offset) for product/shop lists
- WebSocket endpoint: `wss://api.Zovly.com/ws/chat/{room_id}?token={jwt}`
- All responses follow envelope: `{ success, data, error, meta }`
- Rate limiting: 100 req/min per IP, 10 req/min for OTP endpoints

## Governance
The Zovly constitution supersedes all other development practices and architectural decisions. The system architecture follows a FastAPI Python monolith backend with Flutter mobile app, using PostgreSQL 15 with PostGIS extension, Redis 7 for caching/pub-sub, and Claude API for AI integration. All implementation must adhere to the non-negotiable business rules outlined in this constitution, with regular compliance verification required during code reviews and deployments.

**Version**: 1.0.0 | **Ratified**: 2026-03-11 | **Last Amended**: 2026-03-11