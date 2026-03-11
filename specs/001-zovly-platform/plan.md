# Implementation Plan: Zovly Hyperlocal eCommerce Platform

**Branch**: `001-zovly-platform` | **Date**: 2026-03-12 | **Spec**: specs/001-zovly-platform/spec.md
**Input**: Feature specification from `/specs/001-zovly-platform/spec.md`

## Summary

The Zovly Hyperlocal eCommerce Platform will implement a privacy-preserving marketplace connecting local buyers and sellers. The system will feature location-based discovery within 2km radius, anonymous buyer-seller communication, product negotiation with offers, and pickup-based transactions. Implementation will follow the Zovly Constitution which emphasizes privacy-first architecture, constraint-led development, geo-aware services, and asynchronous AI integration.

## Technical Context

**Language/Version**: Python 3.11+ (FastAPI backend)
**Primary Dependencies**: FastAPI, SQLAlchemy 2.0, PostgreSQL 15 with PostGIS, Redis 7, ARQ, Firebase Admin SDK, Cloudinary, Claude API
**Storage**: PostgreSQL 15 with PostGIS extension for geo-location queries
**Testing**: pytest-asyncio, httpx with ASGITransport, pytest-mock
**Target Platform**: Linux server (Railway.app deployment)
**Project Type**: Web application with mobile frontend (FastAPI backend + Flutter mobile app)
**Performance Goals**: Handle 1000 concurrent users with <200ms response time for API calls
**Constraints**: Never store buyer location permanently, never charge transaction fees in Phase 1, 2km search radius as default
**Scale/Scope**: Expected to serve 10K+ users with 100K+ products, 1M+ transactions

## Constitution Check

**GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.**

1. **Location-Based Proximity Commerce** - ✅ COMPLIES
   - Implementation will use PostGIS for accurate 2km radius geoqueries as specified in the constitution

2. **Privacy-First Architecture** - ✅ COMPLIES
   - Buyer GPS location is captured fresh on every app open but never stored permanently
   - Buyer real phone number is never exposed to sellers through API endpoints
   - All geo filtering happens server-side using PostGIS

3. **Anonymity-Preserving Transactions** - ✅ COMPLIES
   - Buyers shown as `Buyer#[4-digit-random]` in chat sessions
   - Anonymous WebSocket chat system preserves identities
   - Platform retains chat logs for 90 days for dispute resolution

4. **Constraint-Led Development (NON-NEGOTIABLE)** - ✅ COMPLIES
   - Never store buyer location permanently
   - Never charge transaction fees in Phase 1 or 2
   - Never act as payment intermediary
   - Never guarantee delivery or product quality
   - Never expose buyer identity to seller
   - Never allow shops to appear outside their registered area
   - Never allow a seller to operate more than 3 shops per verified identity

5. **Geo-Aware Services** - ✅ COMPLIES
   - All geo filtering happens server-side using PostGIS ST_DWithin()
   - Default search radius is 2.0 km as per constitution

6. **Asynchronous AI Integration** - ✅ COMPLIES
   - All Claude API calls are implemented as async background tasks
   - AI integration includes product categorization, content moderation, and message toxicity checking

## Project Structure

### Documentation (this feature)

```text
specs/001-zovly-platform/
├── plan.md              # This file (/sp.plan command output)
├── research.md          # Phase 0 output (/sp.plan command)
├── data-model.md        # Phase 1 output (/sp.plan command)
├── quickstart.md        # Phase 1 output (/sp.plan command)
├── contracts/           # Phase 1 output (/sp.plan command)
└── tasks.md             # Phase 2 output (/sp.tasks command - NOT created by /sp.plan)
```

### Source Code (repository root)

```text
# Option 2: Web application (when "frontend" + "backend" detected)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

api/
└── [same as backend above]

mobile/
└── [Flutter mobile app structure]
```

**Structure Decision**: This is a web application with a FastAPI backend serving both the mobile app and web admin dashboard. The mobile app (Flutter) and backend (FastAPI) will be separated as shown above, following the constitution's requirement for a FastAPI Python monolith backend with Flutter mobile app.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations found. All constitutional principles are fully addressed by the implementation approach.