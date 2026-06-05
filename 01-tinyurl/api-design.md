# API Design: Distributed URL Shortener (TinyURL)

This document details the public and internal REST API contracts for the TinyURL URL Shortener service. The API endpoints are designed to be idempotent, stateless, and optimized for ultra-low latency redirection lookups.

---

## 1. POST `/api/v1/url` (Token Generation)

This endpoint takes a long, raw destination URL and returns a unique, 7-character Base-62 encoded short link.

* **HTTP Method**: `POST`
* **Content Type**: `application/json`
* **Authentication**: Token-based authentication via headers.

### Request Headers
| Header | Value | Description |
| :--- | :--- | :--- |
| `Content-Type` | `application/json` | Required. |
| `X-RateLimit-Token` | `client_key_abcdef12345` | Required. Used to identify the client, authorize write quota, and track consumption rate limits. |

### Request Payload
```json
{
  "long_url": "https://sports.yahoo.com/news/live-updates-nba-finals-game-seven-001245782.html",
  "custom_alias": "nba-finals-g7",
  "ttl_seconds": 315360000
}
```

| Field Name | Data Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `long_url` | String | **Yes** | The original, full-length URL to be shortened. Must contain a valid protocol (`http` or `https`). |
| `custom_alias` | String | No | Optional alphanumeric user-provided string (e.g. `nba-finals-g7`). If provided, it overrides sequence key generation. |
| `ttl_seconds` | Integer | No | Optional time-to-live parameter. Indicates how long the link remains valid before automatic garbage collection. Defaults to 10 years (315,360,000s). |

### Response Headers
* `Status: 201 Created`
* `Content-Type: application/json`

### Response Payload
```json
{
  "short_url": "https://tiny.url/nba-finals-g7",
  "short_url_token": "nba-finals-g7",
  "long_url": "https://sports.yahoo.com/news/live-updates-nba-finals-game-seven-001245782.html",
  "created_at": "2026-06-05T06:45:00Z",
  "expires_at": "2036-06-02T06:45:00Z"
}
```

---

## 2. GET `/{short_url_token}` (Redirection Lookup)

This endpoint resolves a shortened token and performs a permanent HTTP redirection to the original long destination URL.

* **HTTP Method**: `GET`
* **Content Type**: None (returns redirection headers only)

### Request Path Parameter
| Parameter | Data Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `short_url_token` | String | **Yes** | The 7-character Base-62 token representing the shortened key (e.g. `aB87zK2`). |

### Response Headers
* `Status: 301 Moved Permanently` (or `302 Found` based on analytics configuration)
* `Location: https://sports.yahoo.com/news/live-updates-nba-finals-game-seven-001245782.html`
* `Cache-Control: public, max-age=86400, stale-while-revalidate=3600`

### Response Payload
* **Empty Body**: The body is left intentionally empty to minimize network payload sizing. Redirections are parsed and executed natively at the browser socket level by examining the `Location` header.

---

## 3. Redirect Status Code Evaluation: 301 vs. 302

Selecting the appropriate HTTP status code is a critical architectural decision:

### HTTP 301 Moved Permanently (Default Selection)
* **Mechanism**: Instructs browsers and search engines that the requested resource has permanently moved. The browser caches the mapping locally in its own DNS/redirect cache database.
* **Benefit**: Consecutive requests from the same client bypass our API gateway and load balancers entirely, loading directly from local memory. This dramatically reduces system read volume and read latency.
* **Drawback**: Makes tracking click metrics and real-time user-agent analytics extremely difficult for cached links, as subsequent redirects don't hit our servers.

### HTTP 302 Found (Temporary Redirect)
* **Mechanism**: Instructs browsers that the resource has moved temporarily. The browser does not cache the mapping, forcing all subsequent hits to route back to our load balancer.
* **Benefit**: Allows complete, high-fidelity click-tracking, telemetry logging, and geolocation monitoring on every single user request.
* **Drawback**: Substantially increases traffic volumes, requiring larger Redis/Memcached cache nodes and database lookup performance to sustain the read load.
