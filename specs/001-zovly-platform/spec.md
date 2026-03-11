# Feature Specification: Zovly Hyperlocal eCommerce Platform

**Feature Branch**: `1-zovly-platform`
**Created**: 2026-03-11
**Status**: Draft
**Input**: User description: "# sp.specification — Zovly Hyperlocal eCommerce Platform"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Buyer Discovery and Purchase Flow (Priority: P1)

A buyer using the mobile app discovers nearby shops, views products, makes offers, and completes purchases through the anonymous chat system.

**Why this priority**: This represents the core value proposition of the platform - allowing users to find and purchase goods from nearby sellers in a privacy-preserving manner.

**Independent Test**: Can be fully tested by simulating a buyer finding shops within 2km radius, viewing products, making an offer, and completing the purchase flow.

**Acceptance Scenarios**:
1. **Given** buyer has location enabled, **When** they open the discovery screen, **Then** they see shops within 2km radius sorted by distance
2. **Given** buyer is viewing a shop's products, **When** they select a product and make an offer, **Then** they receive an anonymous chat room with the seller
3. **Given** buyer has made an offer, **When** seller accepts it, **Then** buyer receives a pickup voucher with QR code

---

### User Story 2 - Seller Registration and Verification (Priority: P2)

A seller registers with the platform by providing business details and documents, then goes through a verification process.

**Why this priority**: This enables the core functionality of having sellers on the platform and maintaining trust through verification.

**Independent Test**: Can be tested by simulating a seller registration with all required documents, verifying the account, and ensuring they can create products.

**Acceptance Scenarios**:
1. **Given** seller is registering, **When** they submit required documents, **Then** their account enters pending verification state
2. **Given** seller's account is pending verification, **When** admin approves it, **Then** seller can create products and shops
3. **Given** seller has verified account, **When** they create a shop, **Then** the shop appears in discovery results

---

### User Story 3 - Admin Verification and Oversight (Priority: P3)

An administrator manages seller accounts, reviews reports, and monitors platform health.

**Why this priority**: Ensures platform integrity and allows oversight of seller activities and user reports.

**Independent Test**: Can be tested by simulating an admin approving a seller, rejecting one with a reason, and viewing reports.

**Acceptance Scenarios**:
1. **Given** admin accesses verification dashboard, **When** they view pending sellers, **Then** they can approve or reject accounts
2. **Given** admin reviews reports, **When** they find a seller with 3 open reports, **Then** the seller is suspended
3. **Given** admin needs to monitor platform, **When** they view stats dashboard, **Then** they see key metrics like total users, shops, and products

---

### Edge Cases

- What happens when buyer's location is not available or inaccurate?
- How does system handle seller submitting multiple verification requests?
- What happens if a seller's shop location changes significantly?
- How does system handle expired offers or vouchers?
- What happens if a product becomes unavailable during an active offer?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST enable buyers to discover nearby shops within a 2km radius using their GPS location
- **FR-002**: System MUST allow sellers to register with business details and verification documents
- **FR-003**: System MUST enable anonymous chat between buyers and sellers during negotiation
- **FR-004**: System MUST support product listing, viewing, and offer creation by buyers
- **FR-005**: System MUST generate and manage pickup vouchers with QR codes for transactions
- **FR-006**: System MUST support flash deals broadcast to nearby buyers
- **FR-007**: System MUST provide admin dashboard for seller verification and report management
- **FR-008**: System MUST protect buyer privacy by not exposing real phone numbers or locations
- **FR-009**: System MUST enforce transaction rules where sellers can only offer pickup at their physical location
- **FR-010**: System MUST integrate with Claude API for automated content moderation and categorization

### Key Entities *(include if feature involves data)*

- **User**: Represents either a buyer or seller in the system with role-based permissions
- **Shop**: A physical location where sellers list products, including geo-location data
- **Product**: Items listed by sellers for sale, with pricing and inventory details
- **Offer**: A proposal made by buyers to sellers for purchasing a product
- **Chat Room**: Anonymous communication channel between buyers and sellers during negotiation
- **Pickup Voucher**: Digital voucher with QR code that enables physical pickup of purchased items
- **Report**: User-generated complaints about sellers, products, or messages
- **Flash Deal**: Time-limited promotional offers broadcast to nearby buyers

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Buyers can discover and interact with nearby shops within 2km radius in under 3 seconds
- **SC-002**: Seller registration and verification process completes in under 10 minutes
- **SC-003**: 95% of buyers can complete an offer flow (make offer, receive response, complete purchase) within 15 minutes
- **SC-004**: System maintains 99.9% uptime during peak usage hours
- **SC-005**: 90% of verified sellers can create products and make them available within first 24 hours
- **SC-006**: Admin dashboard loads and responds to queries in under 2 seconds
- **SC-007**: Buyer privacy is maintained with 100% of real phone numbers hidden from sellers
- **SC-008**: System detects and flags inappropriate content with 90% accuracy through AI moderation