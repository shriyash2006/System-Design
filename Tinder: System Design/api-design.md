# API Design Matrix: Tinder Matchmaking Subsystem

This document specifies the REST and stateful WebSocket API contracts for Tinder, optimized for low-latency profile registration, geo-scoped discovery deck loading, and real-time mutual matchmaking events.

---

## 1. Profile Creation & Media Ingestion

Users create profiles including bios, gender classifications, age parameters, and media assets.

### Endpoint: Create / Update Profile
* **Path**: `POST /api/v1/profile`
* **Content-Type**: `multipart/form-data`
* **Auth Required**: Yes (Bearer JWT)
* **Request Payload**:
  * `bio` (string): Text biography.
  * `age` (int): User age.
  * `gender` (string): `male` | `female` | `other`.
  * `preference_gender` (string): `male` | `female` | `all`.
  * `preference_max_distance_miles` (int, default: 50, max: 100): Search radius filter.
  * `photos` (binary file arrays): Media upload.
* **Response Payload (200 OK)**:
  ```json
  {
    "user_id": "usr_9982348",
    "status": "ACTIVE",
    "avatar_url": "https://cdn.tinder.com/profiles/usr_9982348/photo1.jpg"
  }
  ```

---

## 2. Recommendation & Discovery Engine

Retrieves the card deck of potential matches within geographic and preference ranges, excluding profiles the caller has already swiped.

### Endpoint: Get Discovery Card Deck
* **Path**: `GET /api/v1/discovery/recommendations`
* **Auth Required**: Yes
* **Query Parameters**:
  * `latitude` (double, required): Current latitude coordinate.
  * `longitude` (double, required): Current longitude coordinate.
  * `limit` (int, default: 20, max: 50): Card count returned.
* **Response Payload (200 OK)**:
  ```json
  {
    "data": [
      {
        "user_id": "usr_776102",
        "username": "Sophia",
        "age": 25,
        "bio": "Software developer who loves hiking and coffee.",
        "distance_miles": 4.2,
        "photos": [
          "https://cdn.tinder.com/profiles/usr_776102/photo1.jpg"
        ]
      }
    ]
  }
  ```

---

## 3. Location Update & Passport Mode

Updates the user's coordinate system. Enabling Passport Mode sets custom virtual coordinates, routing the user to a new geographic Geo Shard.

### Endpoint: Update Coordinates
* **Path**: `PUT /api/v1/location`
* **Auth Required**: Yes
* **Request Payload**:
  ```json
  {
    "latitude": 40.7128,
    "longitude": -74.0060,
    "passport_mode_active": true,
    "destination_name": "New York City"
  }
  ```
* **Response Payload (200 OK)**:
  ```json
  {
    "status": "MIGRATED",
    "new_s2_cell_id": "89c25a3",
    "message": "User discovery deck successfully routed to New York City Geo Shard."
  }
  ```

---

## 4. WebSocket Swiping & Match Loop

Because swiping events require real-time execution, active users establish a persistent WebSocket connection through the Tinder API Gateway.

### Client WebSocket Frame: Swipe Action
* **Event**: `swipe`
* **Payload**:
  ```json
  {
    "event_type": "swipe_right",
    "swipee_id": "usr_776102",
    "timestamp": 1780642220
  }
  ```

### Server WebSocket Frame: Match Notification
* **Event**: `match_alert`
* **Payload**:
  ```json
  {
    "match_id": "mtch_77182903",
    "matched_user": {
      "user_id": "usr_776102",
      "username": "Sophia",
      "avatar_url": "https://cdn.tinder.com/profiles/usr_776102/photo1.jpg"
    },
    "chat_room_id": "room_xyz_abc_77182903",
    "timestamp": 1780642221
  }
  ```
