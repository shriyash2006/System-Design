# API Design: Web Crawler System

This document specifies the internal and service-to-service REST API contracts for managing and monitoring the Web Crawler cluster.

---

## 1. POST `/api/v1/seeds` (Ingest Seed URLs)

Submits a batch of high-authority seed URLs to the crawler to expand or bootstrap the crawl scope.

* **HTTP Method**: `POST`
* **Content-Type**: `application/json`

### Request Headers
| Header | Value | Description |
| :--- | :--- | :--- |
| `Content-Type` | `application/json` | Required. |
| `X-API-Key` | `admin_crawler_secret_key` | Required. Restricts access to administrators. |

### Request Payload
```json
{
  "seeds": [
    {
      "url": "https://en.wikipedia.org/wiki/Main_Page",
      "priority": 10,
      "crawl_interval_hours": 24
    },
    {
      "url": "https://www.nasa.gov",
      "priority": 8,
      "crawl_interval_hours": 48
    }
  ]
}
```

| Field Name | Data Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `seeds` | Array of Objects | **Yes** | A list of seed URLs to inject. |
| `seeds[].url` | String | **Yes** | Valid absolute URL with protocol scheme (`http` or `https`). |
| `seeds[].priority` | Integer | No | Range 1 (Lowest) to 10 (Highest). Influences the initial front queue mapping. |
| `seeds[].crawl_interval_hours` | Integer | No | Custom recurrence interval before the URL is recrawled. |

### Response Headers
* `Status: 202 Accepted`
* `Content-Type: application/json`

### Response Payload
```json
{
  "status": "Accepted",
  "urls_ingested": 2,
  "transaction_id": "tx_crawler_88924018247",
  "timestamp": "2026-06-08T12:15:00Z"
}
```

---

## 2. GET `/api/v1/crawl/status` (Crawl Job Telemetry)

Retrieves real-time progress metrics of the active crawl jobs.

* **HTTP Method**: `GET`
* **Authorization**: Admin API Key required.

### Response Headers
* `Status: 200 OK`
* `Content-Type: application/json`

### Response Payload
```json
{
  "system_status": "RUNNING",
  "active_fetcher_threads": 4500,
  "pages_crawled_count": 89201948,
  "crawl_throughput_pages_per_sec": 386.4,
  "url_frontier_metrics": {
    "front_queues_total_backlog": 12489028,
    "back_queues_active": 45291,
    "back_queues_sleeping": 12048
  },
  "bloom_filters": {
    "url_seen_elements": 90124802,
    "content_seen_elements": 89128390
  }
}
```

---

## 3. GET `/api/v1/dns/resolve` (Internal DNS Verification)

Validates the state of the Fetcher's custom DNS resolver cache.

* **HTTP Method**: `GET`
* **Query Parameter**: `domain` (e.g. `wikipedia.org`)

### Response Payload
```json
{
  "domain": "wikipedia.org",
  "ip_address": "198.35.26.96",
  "resolved_from": "dns_resolver_local_cache",
  "ttl_remaining_seconds": 1842
}
```
