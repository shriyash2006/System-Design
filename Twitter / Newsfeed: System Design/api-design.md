# API Design: Twitter / Newsfeed System

This document specifies the REST API design contracts for the Twitter / Newsfeed system. The endpoints are optimized to support high write rates for tweets and sub-200ms timeline fetches.

---

## 1. POST `/api/v1/tweets` (Create Tweet)

Publishes a new tweet to the user's timeline.

* **HTTP Method**: `POST`
* **Content-Type**: `application/json`

### Request Headers
| Header | Value | Description |
| :--- | :--- | :--- |
| `Content-Type` | `application/json` | Required. |
| `Authorization` | `Bearer <jwt_token>` | Required. Establishes user session and identity. |

### Request Payload
```json
{
  "content": "Building a recruiter-ready system design playbook! #systemdesign #scale",
  "media_ids": [
    "media_889241578"
  ]
}
```

| Field Name | Data Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `content` | String | **Yes** | The tweet text body. UTF-8 encoded, max 280 characters. |
| `media_ids` | Array of Strings | No | Optional array of pre-uploaded image/video resource identifiers. |

### Response Headers
* `Status: 201 Created`
* `Content-Type: application/json`

### Response Payload
```json
{
  "tweet_id": "tweet_110294729104",
  "user_id": "user_44920194",
  "content": "Building a recruiter-ready system design playbook! #systemdesign #scale",
  "media_urls": [
    "https://s3.amazonaws.com/twitter-media/media_889241578.png"
  ],
  "created_at": "2026-06-06T01:55:00Z",
  "like_count": 0,
  "retweet_count": 0
}
```

---

## 2. GET `/api/v1/timeline/home` (Fetch Home Timeline)

Fetches the calling user's personalized home timeline, which compiles tweets from users they follow.

* **HTTP Method**: `GET`
* **Authorization**: JWT Token required in authorization headers.

### Request Query Parameters
| Parameter | Data Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `limit` | Integer | No | Max number of tweets to return (default: 20, max: 100). |
| `max_id` | String | No | Pagination cursor. Returns tweets older than this ID (exclusive). |

### Response Headers
* `Status: 200 OK`
* `Content-Type: application/json`

### Response Payload
```json
{
  "tweets": [
    {
      "tweet_id": "tweet_110294729104",
      "user_id": "user_44920194",
      "username": "shriyash2006",
      "content": "Building a recruiter-ready system design playbook! #systemdesign #scale",
      "media_urls": [
        "https://s3.amazonaws.com/twitter-media/media_889241578.png"
      ],
      "created_at": "2026-06-06T01:55:00Z",
      "like_count": 142,
      "retweet_count": 12
    }
  ],
  "next_cursor": "tweet_110294729104"
}
```

---

## 3. POST `/api/v1/users/{userId}/follow` (Follow User)

Establishes a follow relationship between the authenticated user and the target user.

* **HTTP Method**: `POST`
* **Path Parameter**: `userId` (The target user to follow)

### Response Headers
* `Status: 204 No Content` (indicating successful execution with empty response body)

---

## 4. POST `/api/v1/tweets/{tweetId}/like` (Like Tweet)

Registers a like on a target tweet.

* **HTTP Method**: `POST`
* **Path Parameter**: `tweetId` (The target tweet to like)

### Response Payload
```json
{
  "tweet_id": "tweet_110294729104",
  "like_count": 143,
  "liked_by_me": true
}
```
