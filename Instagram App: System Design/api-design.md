# API Design Matrix: Instagram Subsystem

This document specifies the stateless, cache-friendly REST API contracts for the Instagram application, focused on scale-resilient media ingestion, social graph mutation, and low-latency feed retrieval.

---

## 1. Post Creation & Multipart Media Upload

To handle large media payloads (images up to 10MB, HD videos up to 100MB) without risking connections dropping mid-network, the system utilizes a **Multipart Upload Protocol** matching chunked streaming methods.

### Endpoint: Initiate Multipart Upload Session
* **Path**: `POST /api/v1/posts/upload/initiate`
* **Auth Required**: Yes (Bearer Token)
* **Request Payload**:
  ```json
  {
    "media_type": "video",
    "total_size_bytes": 45120300,
    "mime_type": "video/mp4",
    "caption": "A sunset in San Francisco! #sf #sunset"
  }
  ```
* **Response Payload (201 Created)**:
  ```json
  {
    "upload_id": "upload_sf_9823741893_xyz",
    "chunk_size_bytes": 5242880,
    "total_chunks": 9,
    "upload_urls": [
      {
        "part_number": 1,
        "presigned_url": "https://s3.amazonaws.com/instagram-media-raw/uploads/sf_9823741893_xyz/part-1?AWSAccessKeyId=..."
      },
      {
        "part_number": 2,
        "presigned_url": "https://s3.amazonaws.com/instagram-media-raw/uploads/sf_9823741893_xyz/part-2?AWSAccessKeyId=..."
      }
    ]
  }
  ```

### Endpoint: Complete Upload Session
* **Path**: `POST /api/v1/posts/upload/complete`
* **Auth Required**: Yes
* **Request Payload**:
  ```json
  {
    "upload_id": "upload_sf_9823741893_xyz",
    "parts": [
      { "part_number": 1, "etag": "\"a543571b0c6d59\"" },
      { "part_number": 2, "etag": "\"b743582c0e7f60\"" }
    ]
  }
  ```
* **Response Payload (200 OK)**:
  ```json
  {
    "post_id": "post_1029481029",
    "status": "PROCESSING",
    "media_urls": [
      "https://cdn.instagram.com/posts/2026/06/13/sunset_sf_optimized.mp4"
    ]
  }
  ```

---

## 2. Feed Retrieval & Cursor Pagination

Traditional limit/offset pagination exhibits $O(N)$ database query latency (where $N$ is the offset offset size) and risks showing duplicate posts if new items are inserted during paging. We enforce **Tokenized Cursor Pagination** using encoded base64 timestamps and IDs.

### Endpoint: Fetch Home Timeline Feed
* **Path**: `GET /api/v1/feed`
* **Headers**: `Authorization: Bearer <JWT>`
* **Parameters**:
  * `limit` (int, default: 20, max: 50): Number of records to return.
  * `next_cursor` (string, optional): Base64 encoded state tracking point. E.g., `eyJ0c3BfaW5fc2VjIjoxNzgwNjQyMTg5LCJwb3N0X2lkIjoiMTAyOTQ4MTAyOSJ9`

* **Request Example**:
  `GET /api/v1/feed?limit=20&next_cursor=eyJ0c3BfaW5fc2VjIjoxNzgwNjQyMTg5LCJwb3N0X2lkIjoiMTAyOTQ4MTAyOSJ9`

* **Response Payload (200 OK)**:
  ```json
  {
    "data": [
      {
        "post_id": "post_90283401",
        "author": {
          "user_id": "usr_7728",
          "username": "travel_explorer",
          "avatar_url": "https://cdn.instagram.com/avatars/7728.jpg"
        },
        "caption": "Wandering around Tokyo! #travel",
        "media": [
          {
            "media_id": "med_11202",
            "type": "image",
            "url": "https://cdn.instagram.com/posts/2026/06/tokyo_1.jpg"
          }
        ],
        "created_at": 1780642189,
        "likes_count": 14203,
        "comments_count": 89
      }
    ],
    "paging": {
      "has_more": true,
      "next_cursor": "eyJ0c3BfaW5fc2VjIjoxNzgwNjQyMTkwLCJwb3N0X2lkIjoiOTAyODM0MDEifQ=="
    }
  }
  ```

---

## 3. Social Graph Management (Follow / Unfollow)

These endpoints process relationship mutations which asynchronously update the Neo4j graph structure and flush cache layers.

### Endpoint: Follow User
* **Path**: `POST /api/v1/users/{user_id}/follow`
* **Auth Required**: Yes
* **Response Payload (200 OK)**:
  ```json
  {
    "status": "SUCCESS",
    "relationship": {
      "follower_id": "usr_current_user",
      "followee_id": "usr_target_user",
      "created_at": 1780642190
    }
  }
  ```

### Endpoint: Unfollow User
* **Path**: `POST /api/v1/users/{user_id}/unfollow`
* **Auth Required**: Yes
* **Response Payload (200 OK)**:
  ```json
  {
    "status": "SUCCESS",
    "message": "You are no longer following user_id"
  }
  ```
