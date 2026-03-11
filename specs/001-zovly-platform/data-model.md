# Data Model for Zovly Hyperlocal eCommerce Platform

## Entities

### 1. User
Represents either a buyer or seller in the system with role-based permissions.

**Fields:**
- id: UUID (PK)
- phone_number: String(20) UNIQUE NOT NULL
- role: Enum('BUYER', 'SELLER', 'ADMIN')
- is_active: Boolean DEFAULT TRUE
- firebase_uid: String UNIQUE
- created_at: DateTime
- updated_at: DateTime

**Validation Rules:**
- phone_number must be unique
- role must be one of BUYER, SELLER, or ADMIN
- Each user can only have one role at a time

### 2. Seller Profile
Contains additional information specific to sellers beyond basic user information.

**Fields:**
- id: UUID (PK)
- user_id: FK → users.id
- business_name: String(200) NOT NULL
- cnic_hash: String(64)  # SHA-256 of raw CNIC
- cnic_photo_url: String  # Cloudinary private URL
- selfie_photo_url: String
- shop_front_photo_url: String
- verification_status: Enum('PENDING','VERIFIED','REJECTED','SUSPENDED')
- rejection_reason: Text NULLABLE
- verified_at: DateTime NULLABLE
- verified_by: UUID FK → users.id NULLABLE

**Relationships:**
- One-to-one with User (via user_id)

**Validation Rules:**
- cnic_hash must be SHA-256 hash
- verification_status must be one of PENDING, VERIFIED, REJECTED, SUSPENDED
- When verification_status is VERIFIED, verified_at must be set

### 3. Shop
A physical location where sellers list products, including geo-location data.

**Fields:**
- id: UUID (PK)
- seller_id: FK → seller_profiles.id
- name: String(200) NOT NULL
- description: Text
- category: String(100)
- geo_location: Geometry(POINT, 4326)  # PostGIS
- address_text: String(500)
- cover_image_url: String
- operating_hours: JSONB  # {"mon": "9:00-21:00", ...}
- is_active: Boolean DEFAULT FALSE  # only true after seller verified
- search_radius_km: Float DEFAULT 2.0
- created_at: DateTime
- updated_at: DateTime

**Relationships:**
- Belongs to Seller Profile (via seller_id)

**Validation Rules:**
- When is_active is TRUE, seller must be verified
- geo_location must be a valid POINT in WGS84 coordinate system
- search_radius_km must be between 0.5 and 5.0 km

### 4. Product
Items listed by sellers for sale, with pricing and inventory details.

**Fields:**
- id: UUID (PK)
- shop_id: FK → shops.id
- title: String(300) NOT NULL
- description: Text
- price: Numeric(12,2) NOT NULL
- min_offer_price: Numeric(12,2)  # auto-calculated: price * 0.10
- stock_qty: Integer DEFAULT 0
- category: String(100)
- ai_category: String(100)  # Claude-assigned
- images: ARRAY(String)  # Cloudinary URLs
- is_available: Boolean DEFAULT TRUE
- is_flagged: Boolean DEFAULT FALSE  # AI flagged for review
- view_count: Integer DEFAULT 0
- created_at: DateTime
- updated_at: DateTime

**Relationships:**
- Belongs to Shop (via shop_id)

**Validation Rules:**
- price must be greater than 0
- stock_qty must be non-negative
- min_offer_price = price * 0.10
- When is_available is FALSE, stock_qty should be 0

### 5. Offer
A proposal made by buyers to sellers for purchasing a product.

**Fields:**
- id: UUID (PK)
- buyer_id: FK → users.id
- product_id: FK → products.id
- original_price: Numeric(12,2)
- offered_price: Numeric(12,2)
- counter_price: Numeric(12,2) NULLABLE
- status: Enum('PENDING','ACCEPTED','COUNTERED','DECLINED','EXPIRED','COMPLETED')
- buyer_note: String(500)
- seller_note: String(500)
- expires_at: DateTime  # offered_at + 24hrs
- created_at: DateTime
- updated_at: DateTime

**Relationships:**
- Belongs to Buyer (via buyer_id)
- Belongs to Product (via product_id)

**Validation Rules:**
- offered_price must be between min_offer_price and original_price
- When status is ACCEPTED, product stock must be decremented
- When status is EXPIRED, must be set after 24 hours
- When status is COMPLETED, must be after ACCEPTED

### 6. Chat Room
Anonymous communication channel between buyers and sellers during negotiation.

**Fields:**
- id: UUID (PK)
- offer_id: FK → offers.id NULLABLE
- product_id: FK → products.id
- shop_id: FK → shops.id
- buyer_id: FK → users.id
- buyer_alias: String(20)  # Buyer#XXXX - generated at room creation
- is_active: Boolean DEFAULT TRUE
- created_at: DateTime

**Relationships:**
- May relate to Offer (via offer_id)
- Belongs to Product (via product_id)
- Belongs to Shop (via shop_id)
- Belongs to Buyer (via buyer_id)

**Validation Rules:**
- buyer_alias must be in format Buyer#XXXX where XXXX is 4-digit random number
- When offer_id is provided, must relate to the same product and buyer

### 7. Message
Individual messages exchanged in chat rooms.

**Fields:**
- id: UUID (PK)
- room_id: FK → chat_rooms.id
- sender_role: Enum('BUYER','SELLER','SYSTEM')
- content: String(1000)
- is_read: Boolean DEFAULT FALSE
- is_flagged: Boolean DEFAULT FALSE
- created_at: DateTime

**Relationships:**
- Belongs to Chat Room (via room_id)

**Validation Rules:**
- content must be between 1 and 1000 characters
- sender_role must be BUYER, SELLER, or SYSTEM

### 8. Pickup Voucher
Digital voucher with QR code that enables physical pickup of purchased items.

**Fields:**
- id: UUID (PK)
- offer_id: FK → offers.id
- qr_hash: String(64) UNIQUE  # SHA-256 of id+secret
- agreed_price: Numeric(12,2)
- status: Enum('PENDING','READY','COLLECTED','CANCELLED','EXPIRED')
- valid_until: DateTime  # created_at + 48hrs
- collected_at: DateTime NULLABLE
- created_at: DateTime

**Relationships:**
- Belongs to Offer (via offer_id)

**Validation Rules:**
- status must be one of PENDING, READY, COLLECTED, CANCELLED, EXPIRED
- When status is COLLECTED, collected_at must be set
- When status is READY, must be after PENDING

### 9. Report
User-generated complaints about sellers, products, or messages.

**Fields:**
- id: UUID (PK)
- reporter_id: FK → users.id
- target_type: Enum('SELLER','PRODUCT','MESSAGE','SHOP')
- target_id: UUID
- reason: Enum('SCAM','FAKE_PRODUCT','ABUSIVE','WRONG_LOCATION','OTHER')
- description: Text NULLABLE
- status: Enum('OPEN','REVIEWED','RESOLVED','DISMISSED')
- created_at: DateTime

**Relationships:**
- Submitted by Reporter (via reporter_id)

**Validation Rules:**
- target_type must be one of SELLER, PRODUCT, MESSAGE, SHOP
- reason must be one of SCAM, FAKE_PRODUCT, ABUSIVE, WRONG_LOCATION, OTHER
- status must be one of OPEN, REVIEWED, RESOLVED, DISMISSED

### 10. Flash Deal
Time-limited promotional offers broadcast to nearby buyers.

**Fields:**
- id: UUID (PK)
- shop_id: FK → shops.id
- product_id: FK → products.id
- discount_percent: Integer  # 1-90
- deal_text: String(300)
- starts_at: DateTime
- ends_at: DateTime
- notified_buyer_count: Integer DEFAULT 0
- is_active: Boolean DEFAULT TRUE

**Relationships:**
- Belongs to Shop (via shop_id)
- Belongs to Product (via product_id)

**Validation Rules:**
- discount_percent must be between 1 and 90
- starts_at must be before ends_at
- When is_active is FALSE, must be after ends_at