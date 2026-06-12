# API Design Matrix: Proximity Service

This document defines the stateless, high-availability API endpoints for the Proximity Service subsystem. The service supports two-dimensional coordinate search requests and decoupled high-volume review writing.

---

## 1. Attraction Ingestion (Creation Path)

Allows authorized business owners or system managers to register a new physical point of interest (Attraction) complete with coordinates and media assets.

### Endpoint: Create Attraction
* **Path**: `POST /api/v1/attractions`
* **Content-Type**: `multipart/form-data`
* **Auth Required**: Yes
* **Form Parameters**:
  * `name` (string, required): Name of the business/attraction.
  * `latitude` (double, required): Precise latitude coordinates (e.g., `37.7749`).
  * `longitude` (double, required): Precise longitude coordinates (e.g., `-122.4194`).
  * `category` (string, required): Business category classification (e.g., `restaurant`, `museum`).
  * `address` (string, optional): Full textual physical street address.
  * `photos` (binary files, optional): Array of business photo files.
* **Response Payload (201 Created)**:
  ```json
  {
    "attraction_id": "attr_9082341",
    "name": "Sotto Mare Oysteria",
    "latitude": 37.7749,
    "longitude": -122.4194,
    "geohash": "9q8yyz",
    "media_urls": [
      "https://cdn.proximity.com/attractions/attr_9082341/cover.jpg"
    ],
    "created_at": 1780642195
  }
  ```

---

## 2. Geospatial Proximity Search (Read Path)

Clients fetch nearby attractions filtered by radius, category tags, and search keywords.

### Endpoint: Search Attractions Nearby
* **Path**: `GET /api/v1/attractions/search`
* **Auth Required**: No (Public)
* **Query Parameters**:
  * `latitude` (double, required): User's current latitude coordinate.
  * `longitude` (double, required): User's current longitude coordinate.
  * `radius_meters` (int, default: 5000, max: 50000): Geospatial search boundary.
  * `query` (string, optional): Full-text search term (e.g., "sushi").
  * `category` (string, optional): Restricts results to category tag (e.g., "restaurants").
  * `limit` (int, default: 20, max: 100): Paging size limits.
  * `next_cursor` (string, optional): Pagination token encoding geohash and offset.

* **Request Example**:
  `GET /api/v1/attractions/search?latitude=37.7749&longitude=-122.4194&radius_meters=2000&query=sushi&limit=20`

* **Response Payload (200 OK)**:
  ```json
  {
    "data": [
      {
        "attraction_id": "attr_77361",
        "name": "Sushi Zone San Francisco",
        "latitude": 37.7758,
        "longitude": -122.4205,
        "distance_meters": 137.4,
        "average_stars": 4.7,
        "reviews_count": 1402,
        "geohash": "9q8yyz7u",
        "categories": ["restaurants", "japanese", "sushi"],
        "media_urls": [
          "https://cdn.proximity.com/attractions/attr_77361/thumb.jpg"
        ]
      }
    ],
    "paging": {
      "has_more": true,
      "next_cursor": "eyJnaCI6IjlxOHl5ejd1Iiwib2ZmIjoxfQ=="
    }
  }
  ```

---

## 3. High-Throughput Review Submission (Review Path)

Allows users to submit reviews. Since reviews are high-volume, these writes bypass the PostGIS transactional relational database and stream to an append-optimized NoSQL database.

### Endpoint: Submit Review
* **Path**: `POST /api/v1/attractions/reviews`
* **Auth Required**: Yes
* **Request Payload**:
  ```json
  {
    "attraction_id": "attr_77361",
    "star_rating": 5,
    "content": "Outstanding food and very fast service! Highly recommend the spicy tuna roll."
  }
  ```
* **Response Payload (202 Accepted)**:
  ```json
  {
    "status": "ACCEPTED",
    "review_id": "rev_082348123_abc",
    "message": "Your review has been submitted and is processing."
  }
  ```
