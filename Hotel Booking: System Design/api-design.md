# API Design Matrix: Hotel Booking Subsystem

This document specifies the REST API contracts for the Hotel Booking Subsystem. These endpoints manage property registration, faceted search requests, and transaction-safe room bookings.

---

## 1. Hotel & Room Catalog Management (Admin Ingestion Path)

Authorized property administrators update hotel listings and room configurations.

### Endpoint: Create Hotel Profile
* **Path**: `POST /api/v1/hotels`
* **Content-Type**: `application/json`
* **Auth Required**: Yes (Bearer JWT Admin)
* **Request Payload**:
  ```json
  {
    "name": "Grand Palace Hotel",
    "destination": "San Francisco, CA",
    "description": "Luxurious stay in the heart of downtown.",
    "amenities": ["wifi", "pool", "gym", "spa"],
    "latitude": 37.7749,
    "longitude": -122.4194
  }
  ```
* **Response Payload (201 Created)**:
  ```json
  {
    "hotel_id": "htl_00982348",
    "status": "DRAFT",
    "created_at": 1780642230
  }
  ```

### Endpoint: Add Room Type config
* **Path**: `POST /api/v1/hotels/{hotel_id}/room-types`
* **Auth Required**: Yes (Admin)
* **Request Payload**:
  ```json
  {
    "name": "Deluxe King Room",
    "capacity": 2,
    "total_inventory": 45,
    "default_price": 249.99
  }
  ```
* **Response Payload (201 Created)**:
  ```json
  {
    "room_type_id": "rt_882934",
    "hotel_id": "htl_00982348",
    "total_rooms": 45
  }
  ```

---

## 2. Discovery & Lodging Search (Search Path)

Allows users to search for lodging matching check-in/checkout intervals.

### Endpoint: Search Lodgings
* **Path**: `GET /api/v1/search`
* **Auth Required**: No (Public)
* **Query Parameters**:
  * `destination` (string, required): Search query (e.g., "San Francisco").
  * `check_in` (string, required): Format `YYYY-MM-DD` (e.g., `2026-07-01`).
  * `check_out` (string, required): Format `YYYY-MM-DD` (e.g., `2026-07-08`).
  * `guests` (int, default: 1): Number of visitors.
  * `limit` (int, default: 20): Paged results limit.
* **Response Payload (200 OK)**:
  ```json
  {
    "data": [
      {
        "hotel_id": "htl_00982348",
        "name": "Grand Palace Hotel",
        "destination": "San Francisco, CA",
        "room_types": [
          {
            "room_type_id": "rt_882934",
            "name": "Deluxe King Room",
            "total_stay_price": 1749.93,
            "available_count": 8
          }
        ]
      }
    ]
  }
  ```

---

## 3. Concurrency-Locked Booking (Reservation Path)

Locks down a room. Employs an **Idempotency Key** inside headers to protect against network retry duplicate charges.

### Endpoint: Book Reservation
* **Path**: `POST /api/v1/reservations`
* **Headers**: 
  * `Authorization: Bearer <JWT>`
  * `X-Idempotency-Key: <UUIDv4>` (e.g., `9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d`)
* **Request Payload**:
  ```json
  {
    "room_type_id": "rt_882934",
    "check_in": "2026-07-01",
    "check_out": "2026-07-08",
    "guest_details": {
      "primary_guest_name": "Alice Johnson",
      "additional_guests": 1
    }
  }
  ```
* **Response Payload (201 Created)**:
  ```json
  {
    "reservation_id": "res_77192304",
    "status": "PENDING_PAYMENT",
    "expires_at_timestamp": 1780642830,
    "total_charge": 1749.93
  }
  ```
