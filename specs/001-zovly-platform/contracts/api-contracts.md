# API Contracts for Zovly Hyperlocal eCommerce Platform

## Overview
This document defines the API contracts for the Zovly Hyperlocal eCommerce Platform. All endpoints are versioned under `/api/v1/` and follow RESTful conventions.

## Authentication Endpoints

### Buyer Authentication

**POST /api/v1/auth/buyer/send-otp**
```
Request:
{
  "phone_number": "+92XXXXXXXXXX"
}

Response:
{
  "success": true,
  "message": "OTP sent"
}
```

**POST /api/v1/auth/buyer/verify-otp**
```
Request:
{
  "phone_number": "+92XXXXXXXXXX",
  "firebase_id_token": "string"
}

Response:
{
  "access_token": "string",
  "token_type": "bearer",
  "user": {
    "id": "uuid",
    "phone_number": "+92XXXXXXXXXX",
    "role": "BUYER"
  }
}
```

### Seller Authentication

**POST /api/v1/auth/seller/register**
```
Request (multipart/form-data):
{
  "phone_number": "+92XXXXXXXXXX",
  "business_name": "string",
  "cnic_number": "string",
  "cnic_photo": "file",
  "selfie_photo": "file",
  "shop_front_photo": "file",
  "firebase_id_token": "string",
  "shop_lat": "number",
  "shop_lng": "number",
  "address_text": "string"
}

Response:
{
  "success": true,
  "message": "Registration submitted. Await verification 24-48hrs.",
  "seller_id": "uuid"
}
```

**POST /api/v1/auth/seller/login**
```
Request:
{
  "firebase_id_token": "string"
}

Response:
{
  "access_token": "string",
  "shop": {
    "id": "uuid",
    "name": "string",
    "is_active": boolean
  }
}
```

## Discovery Endpoints

### Shop Discovery

**GET /api/v1/discovery/shops**
```
Query Parameters:
- lat: number (required)
- lng: number (required)
- radius_km: number (default: 2.0, max: 5.0)
- category: string (optional)
- cursor: string (optional)

Response:
{
  "shops": [
    {
      "id": "uuid",
      "name": "string",
      "category": "string",
      "distance_meters": number,
      "cover_image": "string",
      "is_open_now": boolean,
      "product_count": number
    }
  ],
  "next_cursor": "string",
  "total_in_radius": number
}
```

**GET /api/v1/discovery/shops/{shop_id}/products**
```
Query Parameters:
- cursor: string (optional)
- category: string (optional)
- min_price: number (optional)
- max_price: number (optional)
- available_only: boolean (optional)

Response:
{
  "products": [
    {
      "id": "uuid",
      "title": "string",
      "price": number,
      "images": ["string"],
      "is_available": boolean
    }
  ],
  "next_cursor": "string"
}
```

**GET /api/v1/discovery/products/search**
```
Query Parameters:
- q: string (required)
- lat: number (required)
- lng: number (required)
- radius_km: number (default: 2.0, max: 5.0)
- cursor: string (optional)

Response:
{
  "products": [
    {
      "id": "uuid",
      "title": "string",
      "price": number,
      "images": ["string"],
      "is_available": boolean
    }
  ],
  "next_cursor": "string"
}
```

**GET /api/v1/discovery/deals/nearby**
```
Query Parameters:
- lat: number (required)
- lng: number (required)
- radius_km: number (default: 2.0, max: 5.0)

Response:
{
  "deals": [
    {
      "id": "uuid",
      "product_id": "uuid",
      "discount_percent": integer,
      "deal_text": "string",
      "starts_at": "datetime",
      "ends_at": "datetime"
    }
  ]
}
```

## Shop Management Endpoints

### Seller Shop Management

**GET /api/v1/shops/me**
```
Response:
{
  "id": "uuid",
  "name": "string",
  "description": "string",
  "category": "string",
  "geo_location": {
    "lat": number,
    "lng": number
  },
  "address_text": "string",
  "cover_image_url": "string",
  "operating_hours": {"mon": "9:00-21:00", ...},
  "is_active": boolean
}
```

**PUT /api/v1/shops/me**
```
Request:
{
  "name": "string",
  "description": "string",
  "operating_hours": {"mon": "9:00-21:00", ...},
  "cover_image": "file"
}

Response:
{
  "id": "uuid",
  "name": "string",
  "description": "string",
  "category": "string",
  "geo_location": {
    "lat": number,
    "lng": number
  },
  "address_text": "string",
  "cover_image_url": "string",
  "operating_hours": {"mon": "9:00-21:00", ...},
  "is_active": boolean
}
```

**POST /api/v1/shops/me/toggle**
```
Response:
{
  "is_active": boolean
}
```

## Product Management Endpoints

### Seller Product Management

**POST /api/v1/products/**
```
Request (multipart/form-data):
{
  "title": "string",
  "description": "string",
  "price": number,
  "stock_qty": integer,
  "category": "string",
  "images": ["file"]
}

Response:
{
  "id": "uuid",
  "title": "string",
  "description": "string",
  "price": number,
  "stock_qty": integer,
  "category": "string",
  "ai_category": "string",
  "images": ["string"],
  "is_available": boolean,
  "created_at": "datetime"
}
```

**GET /api/v1/products/**
```
Response:
{
  "products": [
    {
      "id": "uuid",
      "title": "string",
      "price": number,
      "images": ["string"],
      "is_available": boolean
    }
  ],
  "next_cursor": "string"
}
```

**GET /api/v1/products/{product_id}**
```
Response:
{
  "id": "uuid",
  "title": "string",
  "description": "string",
  "price": number,
  "min_offer_price": number,
  "stock_qty": integer,
  "category": "string",
  "ai_category": "string",
  "images": ["string"],
  "is_available": boolean,
  "view_count": integer,
  "created_at": "datetime"
}
```

**PUT /api/v1/products/{product_id}**
```
Request:
{
  "title": "string",
  "description": "string",
  "price": number,
  "stock_qty": integer,
  "category": "string",
  "images": ["string"]
}

Response:
{
  "id": "uuid",
  "title": "string",
  "description": "string",
  "price": number,
  "min_offer_price": number,
  "stock_qty": integer,
  "category": "string",
  "ai_category": "string",
  "images": ["string"],
  "is_available": boolean,
  "view_count": integer,
  "created_at": "datetime"
}
```

**DELETE /api/v1/products/{product_id}**
```
Response:
{
  "success": true
}
```

**POST /api/v1/products/{product_id}/toggle-availability**
```
Response:
{
  "is_available": boolean
}
```

**POST /api/v1/products/{product_id}/flash-deal**
```
Request:
{
  "discount_percent": integer,
  "deal_text": "string",
  "duration_hours": integer
}

Response:
{
  "id": "uuid",
  "product_id": "uuid",
  "discount_percent": integer,
  "deal_text": "string",
  "starts_at": "datetime",
  "ends_at": "datetime"
}
```

## Offer Management Endpoints

### Buyer and Seller Offer Management

**POST /api/v1/offers/**
```
Request:
{
  "product_id": "uuid",
  "offered_price": number,
  "buyer_note": "string"
}

Response:
{
  "offer": {
    "id": "uuid",
    "product_id": "uuid",
    "offered_price": number,
    "status": "string",
    "expires_at": "datetime",
    "created_at": "datetime"
  },
  "chat_room_id": "uuid"
}
```

**GET /api/v1/offers/buyer/me**
```
Response:
{
  "offers": [
    {
      "id": "uuid",
      "product_id": "uuid",
      "offered_price": number,
      "status": "string",
      "expires_at": "datetime",
      "created_at": "datetime"
    }
  ],
  "next_cursor": "string"
}
```

**GET /api/v1/offers/seller/me**
```
Response:
{
  "offers": [
    {
      "id": "uuid",
      "buyer_id": "uuid",
      "product_id": "uuid",
      "original_price": number,
      "offered_price": number,
      "status": "string",
      "expires_at": "datetime",
      "created_at": "datetime"
    }
  ],
  "next_cursor": "string"
}
```

**POST /api/v1/offers/{offer_id}/respond**
```
Request:
{
  "action": "ACCEPT|COUNTER|DECLINE",
  "counter_price": number,
  "seller_note": "string"
}

Response:
{
  "success": true
}
```

**POST /api/v1/offers/{offer_id}/buyer-respond**
```
Request:
{
  "action": "ACCEPT|DECLINE"
}

Response:
{
  "success": true
}
```

## Chat Endpoints

### Real-time Anonymous Chat

**GET /api/v1/chat/rooms**
```
Response:
{
  "rooms": [
    {
      "id": "uuid",
      "offer_id": "uuid",
      "product_id": "uuid",
      "shop_id": "uuid",
      "buyer_id": "uuid",
      "buyer_alias": "string",
      "is_active": boolean,
      "created_at": "datetime"
    }
  ]
}
```

**GET /api/v1/chat/rooms/{room_id}/messages**
```
Response:
{
  "messages": [
    {
      "id": "uuid",
      "room_id": "uuid",
      "sender_role": "BUYER|SELLER|SYSTEM",
      "content": "string",
      "is_read": boolean,
      "is_flagged": boolean,
      "created_at": "datetime"
    }
  ],
  "next_cursor": "string"
}
```

**POST /api/v1/chat/rooms/{room_id}/read**
```
Response:
{
  "success": true
}
```

**DELETE /api/v1/chat/rooms/{room_id}**
```
Response:
{
  "success": true
}
```

## Voucher Endpoints

### Pickup Voucher Management

**GET /api/v1/vouchers/buyer/me**
```
Response:
{
  "vouchers": [
    {
      "id": "uuid",
      "offer_id": "uuid",
      "qr_hash": "string",
      "agreed_price": number,
      "status": "string",
      "valid_until": "datetime",
      "created_at": "datetime"
    }
  ]
}
```

**GET /api/v1/vouchers/{voucher_id}**
```
Response:
{
  "id": "uuid",
  "offer_id": "uuid",
  "qr_data": "string",
  "shop_address": "string",
  "product_title": "string",
  "agreed_price": number,
  "valid_until": "datetime",
  "collected_at": "datetime"
}
```

**POST /api/v1/vouchers/{qr_hash}/verify**
```
Request:
{
  "qr_hash": "string"
}

Response:
{
  "success": true,
  "buyer_alias": "string",
  "product_title": "string",
  "agreed_price": number
}
```

**POST /api/v1/vouchers/{voucher_id}/mark-ready**
```
Response:
{
  "success": true
}
```

**POST /api/v1/vouchers/{voucher_id}/cancel**
```
Response:
{
  "success": true
}
```

## Reporting Endpoints

### User Report Management

**POST /api/v1/reports/**
```
Request:
{
  "target_type": "SELLER|PRODUCT|MESSAGE|SHOP",
  "target_id": "uuid",
  "reason": "SCAM|FAKE_PRODUCT|ABUSIVE|WRONG_LOCATION|OTHER",
  "description": "string"
}

Response:
{
  "success": true
}
```

**GET /api/v1/reports/my**
```
Response:
{
  "reports": [
    {
      "id": "uuid",
      "target_type": "SELLER|PRODUCT|MESSAGE|SHOP",
      "target_id": "uuid",
      "reason": "SCAM|FAKE_PRODUCT|ABUSIVE|WRONG_LOCATION|OTHER",
      "description": "string",
      "status": "OPEN|REVIEWED|RESOLVED|DISMISSED",
      "created_at": "datetime"
    }
  ]
}
```

## Admin Endpoints

### Admin Panel Management

**GET /api/v1/admin/sellers/pending**
```
Response:
{
  "sellers": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "business_name": "string",
      "cnic_hash": "string",
      "cnic_photo_url": "string",
      "selfie_photo_url": "string",
      "shop_front_photo_url": "string",
      "verification_status": "PENDING",
      "created_at": "datetime"
    }
  ]
}
```

**GET /api/v1/admin/sellers/{seller_id}**
```
Response:
{
  "id": "uuid",
  "user_id": "uuid",
  "business_name": "string",
  "cnic_hash": "string",
  "cnic_photo_url": "string",
  "selfie_photo_url": "string",
  "shop_front_photo_url": "string",
  "verification_status": "PENDING|VERIFIED|REJECTED|SUSPENDED",
  "rejection_reason": "string",
  "verified_at": "datetime",
  "shop": {
    "id": "uuid",
    "name": "string",
    "geo_location": {
      "lat": number,
      "lng": number
    },
    "address_text": "string",
    "is_active": boolean
  }
}
```

**POST /api/v1/admin/sellers/{seller_id}/verify**
```
Request:
{}

Response:
{
  "success": true
}
```

**POST /api/v1/admin/sellers/{seller_id}/reject**
```
Request:
{
  "reason": "string"
}

Response:
{
  "success": true
}
```

**POST /api/v1/admin/sellers/{seller_id}/suspend**
```
Request:
{}

Response:
{
  "success": true
}
```

**GET /api/v1/admin/reports**
```
Response:
{
  "reports": [
    {
      "id": "uuid",
      "reporter_id": "uuid",
      "target_type": "SELLER|PRODUCT|MESSAGE|SHOP",
      "target_id": "uuid",
      "reason": "SCAM|FAKE_PRODUCT|ABUSIVE|WRONG_LOCATION|OTHER",
      "description": "string",
      "status": "OPEN|REVIEWED|RESOLVED|DISMISSED",
      "created_at": "datetime"
    }
  ]
}
```

**POST /api/v1/admin/reports/{report_id}/resolve**
```
Request:
{
  "action": "RESOLVE|DISMISS"
}

Response:
{
  "success": true
}
```

**GET /api/v1/admin/stats**
```
Response:
{
  "total_users": number,
  "total_shops": number,
  "total_products": number,
  "active_offers": number,
  "today_vouchers": number
}
```

## Response Envelope Format

All API responses follow this envelope format:
```
{
  "success": boolean,
  "data": {},
  "error": "string",
  "meta": {}
}
```

## Error Responses

Standard error response format:
```
{
  "success": false,
  "error": "error message",
  "meta": {}
}
```

## Rate Limiting

- `/api/v1/auth/*/send-otp`: 3 per phone per hour
- `/api/v1/auth/*`: 20 per IP per minute
- `/api/v1/discovery/*`: 60 per token per minute
- `/api/v1/offers/`: 10 POST per token per hour
- `/api/v1/admin/*`: 200 per IP per minute