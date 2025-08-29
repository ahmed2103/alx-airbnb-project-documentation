# Requirements: ALX Airbnb Project - Backend Features

This document records technical and functional requirements for key backend features of the ALX Airbnb-style project. It focuses on three core subsystems:

1. User Authentication
2. Property Management
3. Booking System

Each section contains: purpose, API endpoints, input / output specifications (JSON schemas), validation rules, error responses, security considerations, and performance criteria.

---

## 1. User Authentication

### Purpose

Provide secure user registration, login, session/token management, password reset, and basic profile retrieval.

### Actors

* Guest (not authenticated)
* Registered User (authenticated)
* Admin

### High-level requirements

* Support email + password registration and login.
* Issue JWT access tokens (short-lived) and refresh tokens (long-lived, stored securely).
* Passwords stored hashed using Argon2id or bcrypt with strong parameters.
* Email verification on signup.
* Rate-limit authentication endpoints to mitigate brute-force attacks.

### Endpoints

#### POST /api/v1/auth/register

* Description: Create a new user and send verification email.
* Auth: None
* Request JSON:

  ```json
  {
    "first_name": "string",
    "last_name": "string",
    "email": "user@example.com",
    "password": "P@ssw0rd!",
    "phone_number": "+201234567890"  
  }
  ```
* Validation rules:

  * `first_name`, `last_name`: required, 1-50 characters, letters, spaces, hyphens allowed.
  * `email`: required, valid email format, max 254 chars, unique.
  * `password`: required, min 10 chars, at least one uppercase, one lowercase, one digit, one special character.
  * `phone_number`: optional, E.164 format if provided.
* Responses:

  * 201 Created: `{ "message": "User registered. Verification email sent." }`
  * 400 Bad Request: validation error details.
  * 409 Conflict: email already exists.
* Security:

  * Password never returned.
  * Create user in DB with `is_verified=false` and an email verification token (expires in 24 hours).
* Performance:

  * P95 registration latency < 300ms under normal load.
  * System must handle 500 registrations/minute burst with backpressure.

#### POST /api/v1/auth/login

* Description: Authenticate user and return access + refresh tokens.
* Auth: None
* Request JSON:

  ```json
  { "email": "user@example.com", "password": "P@ssw0rd!" }
  ```
* Validation rules:

  * `email` and `password` required.
* Responses:

  * 200 OK:

    ```json
    {
      "access_token": "<jwt>",
      "expires_in": 900,
      "refresh_token": "<opaque-uuid>",
      "token_type": "bearer"
    }
    ```
  * 401 Unauthorized: invalid credentials or unverified email.
  * 429 Too Many Requests: rate-limited.
* Security:

  * Access token: JWT signed with RS256, expiry 15 minutes.
  * Refresh token: stored hashed in DB, expiry 30 days.
  * Use `secure`, `httpOnly` cookie for refresh token if web client.
* Performance:

  * P95 login latency < 200ms.

#### POST /api/v1/auth/token/refresh

* Description: Exchange refresh token for new access token.
* Auth: None (refresh token required)
* Request JSON or Cookie: `{ "refresh_token": "<token>" }`
* Responses:

  * 200 OK: new access token and optionally new refresh token.
  * 401 Unauthorized: invalid/expired refresh token.
* Security:

  * Rotate refresh tokens: issue new refresh token on use, revoke previous.

#### POST /api/v1/auth/logout

* Description: Revoke refresh token(s) and terminate session.
* Auth: Bearer access token
* Request JSON: optional `{ "all_devices": true }`
* Responses:

  * 200 OK: confirmation message.

#### GET /api/v1/users/me

* Description: Return authenticated user's profile (non-sensitive fields).
* Auth: Bearer access token
* Response JSON:

  ```json
  {
    "id": "uuid",
    "first_name": "string",
    "last_name": "string",
    "email": "user@example.com",
    "phone_number": "+201234567890",
    "is_verified": true,
    "role": "guest",
    "created_at": "2025-08-29T12:00:00Z"
  }
  ```

### Error codes (common)

* 400 Bad Request — validation error
* 401 Unauthorized — invalid credentials or token
* 403 Forbidden — action not allowed
* 404 Not Found — resource missing
* 429 Too Many Requests — rate limited
* 500 Internal Server Error — unexpected

---

## 2. Property Management

### Purpose

Create/read/update/delete properties, manage images, host verification, and search/filter properties.

### Actors

* Guest (can search/listings)
* Host (create and manage their properties)
* Admin

### High-level requirements

* CRUD operations for properties.
* Image upload via signed URL or multipart upload; images stored in S3-compatible storage.
* Properties have availability calendars and metadata (amenities, price, rules).
* Search API supports pagination, sorting, filtering (location, price range, property type, amenities, number of guests).
* Soft-delete properties to preserve historical bookings.

### Data model (excerpt)

* Property:

  * `id` UUID
  * `host_id` UUID (FK Users)
  * `title` string (max 140)
  * `description` text
  * `address` object `{street, city, state, country, lat, lng, postal_code}`
  * `price_per_night` decimal
  * `currency` string (ISO4217)
  * `max_guests` integer
  * `amenities` array of strings
  * `images` array of URLs
  * `is_active` boolean
  * `created_at`, `updated_at`

### Endpoints

#### POST /api/v1/properties

* Description: Host creates a property listing.
* Auth: Bearer token (role: host or admin)
* Request JSON (example):

  ```json
  {
    "title": "Cozy Nile-facing studio",
    "description": "Near downtown...",
    "address": {"street":"...","city":"Cairo","state":"Cairo","country":"EG","lat":30.0444,"lng":31.2357,"postal_code":"11511"},
    "price_per_night": 35.00,
    "currency": "USD",
    "max_guests": 2,
    "amenities": ["wifi","kitchen","air_conditioning"]
  }
  ```
* Validation rules:

  * `title`: required, 5-140 chars.
  * `price_per_night`: required, >= 0.5
  * `currency`: valid ISO4217 code.
  * `max_guests`: required, 1-50.
  * `lat`/`lng`: valid float ranges.
* Responses:

  * 201 Created: returns created property object.
  * 400, 401, 403 as applicable.
* Performance:

  * P95 create latency < 300ms.

#### GET /api/v1/properties/{property\_id}

* Description: Fetch property details.
* Auth: Public
* Response: property object plus host public profile (name, rating).
* Caching: Cache GET responses for 60 seconds; invalidate on update.

#### PUT /api/v1/properties/{property\_id}

* Description: Update property (host only).
* Auth: Bearer token; host must own property or be admin.
* Validation: same as create; partial updates allowed via PATCH.

#### DELETE /api/v1/properties/{property\_id}

* Description: Soft-delete.
* Auth: host or admin.
* Response: 204 No Content.

#### POST /api/v1/properties/{property\_id}/images

* Description: Request an upload URL (signed).
* Auth: host
* Request: `{ "filename": "image.jpg", "content_type": "image/jpeg" }`
* Response: `{ "upload_url": "https://s3...", "file_url": "https://cdn.../image.jpg" }`
* Validation: Max image size 10MB; allowed types: jpeg, png, webp.
* Security: Signed URLs expire in 5 minutes.

#### GET /api/v1/properties/search

* Description: Search and filter properties.
* Query parameters:

  * `q` (text search), `city`, `country`, `lat`, `lng`, `radius_km`,
  * `min_price`, `max_price`, `currency`, `min_guests`, `amenities` (comma separated),
  * `start_date`, `end_date` (for availability filter),
  * `sort` (`price_asc`, `price_desc`, `rating`, `distance`),
  * `page`, `page_size` (pagination)
* Response: paginated list with `total_count`, `page`, `page_size`, `items`.
* Performance:

  * Search P95 latency < 250ms for basic filters; complex queries may be slower.
  * Support 1000 search requests/minute sustained.

---

## 3. Booking System

### Purpose

Allow guests to request and manage bookings, check availability, handle payments integration (abstracted), and manage cancellations.

### Actors

* Guest
* Host
* Admin

### High-level requirements

* Check availability before creating booking.
* Prevent double-booking using optimistic locking or calendar reservations.
* Save booking state transitions (requested → confirmed → checked-in → completed → cancelled).
* Integrate with payment gateway (tokenized payments) or support 'pay later' flow.

### Data model (excerpt)

* Booking:

  * `id` UUID
  * `property_id` UUID
  * `guest_id` UUID
  * `host_id` UUID
  * `start_date` date
  * `end_date` date
  * `total_price` decimal
  * `currency` string
  * `status` enum (requested, confirmed, cancelled, completed, refunded)
  * `created_at`, `updated_at`

### Endpoints

#### POST /api/v1/bookings

* Description: Create a booking request (guest).
* Auth: Bearer token (guest)
* Request JSON:

  ```json
  {
    "property_id": "uuid",
    "start_date": "2025-09-10",
    "end_date": "2025-09-15",
    "guests": 2,
    "payment_method_id": "pm_..."  
  }
  ```
* Validation rules:

  * `property_id`: required, must exist and be active.
  * `start_date` and `end_date`: required, `start_date` < `end_date`, not in the past.
  * Minimum stay rules enforced by property if present.
  * `guests` <= `property.max_guests`.
* Workflow & Concurrency:

  * Verify availability atomically (use DB transaction + row-level locks or calendar service).
  * Reserve dates for short window (e.g., 10 minutes) while payment completes.
* Responses:

  * 201 Created: booking object with `status: requested` and `expires_at` if awaiting payment.
  * 409 Conflict: dates not available.
* Payments:

  * If payment required at booking, request a payment intent from payment provider and proceed.
  * On payment success, set booking `status: confirmed`.

#### GET /api/v1/bookings/{booking\_id}

* Description: Fetch booking details (guest/host/admin authorized).
* Auth: Bearer token

#### PATCH /api/v1/bookings/{booking\_id}/cancel

* Description: Cancel booking (guest or host under rules).
* Auth: Bearer token
* Rules: Cancellation policy determines refund eligibility.
* Response: 200 OK with updated booking status and refund details if any.

#### GET /api/v1/properties/{property\_id}/availability

* Description: Return availability calendar for the property for given range.
* Query: `start_date`, `end_date` (max 365 days window)
* Response: array of date ranges or an optimized bitmap representation of availability.
* Performance:

  * Calendar requests P95 < 150ms for ranges < 90 days.

### Performance & Scale

* Booking creation throughput target: 200 bookings/minute sustained during peak.
* Strong consistency for availability checks; eventual consistency acceptable for analytics/caches.
* Use background workers for post-booking tasks (notifications, calendar sync, analytics).

---

## Cross-cutting requirements

### Authentication & Authorization

* Role-based access control: guest, host, admin.
* Use JWT for stateless auth; refresh tokens for long-lived sessions.

### Input validation & sanitization

* Strictly validate all inputs server-side.
* Sanitize text fields to prevent XSS in any admin or host UI that renders HTML.

### Rate limiting & Throttling

* Auth endpoints: 10 requests/min per IP (configurable).
* Search endpoints: 120 requests/min per IP or API key.

### Observability

* Expose metrics (Prometheus): request latency, error rates, DB pool usage, queue length.
* Structured logs (JSON) with correlation ID per request.
* Distributed tracing (optional) using W3C Trace Context.

### Backups & Data Retention

* Daily DB backups, point-in-time recovery for 7 days.
* Audit logs retained for 90 days.

### SLA & Performance targets (summary)

* API availability target: 99.9%.
* P95 latency: auth < 300ms, property read < 250ms, search < 250ms, booking creation < 400ms.

---

## Appendix: Example error JSON

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed for field 'email'",
    "details": { "email": "invalid format" }
  }
}
```

---

End of document.
