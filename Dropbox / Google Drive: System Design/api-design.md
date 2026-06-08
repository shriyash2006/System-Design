# API Design: Distributed Cloud Storage (Dropbox/Google Drive)

This document specifies the REST API design contracts for the Block Service and Metadata Service of the distributed cloud storage system.

---

## 1. POST `/api/v1/blocks/upload` (Block Service Upload)

Uploads an individual compressed and encrypted 4 MB file block.

* **HTTP Method**: `POST`
* **Content-Type**: `application/octet-stream`

### Request Headers
| Header | Value | Description |
| :--- | :--- | :--- |
| `Content-Type` | `application/octet-stream` | Required. Indicates binary block payload. |
| `Authorization` | `Bearer <jwt_token>` | Required. |
| `X-Block-Hash` | `sha256_hash_value` | Required. The SHA-256 content hash of the block, used for deduplication check. |

### Response Headers
* `Status: 201 Created` (or `200 OK` if the block already exists globally due to deduplication)
* `Content-Type: application/json`

### Response Payload
```json
{
  "block_hash": "sha256_hash_value",
  "status": "UPLOADED",
  "bytes_received": 1048576,
  "created_at": "2026-06-08T12:46:00Z"
}
```

---

## 2. POST `/api/v1/metadata/sync` (Metadata Service Sync)

Updates the global file catalog with a new version's block map after a successful block upload sequence.

* **HTTP Method**: `POST`
* **Content-Type**: `application/json`

### Request Payload
```json
{
  "object_id": "obj_99018247",
  "space_id": "space_shared_450",
  "file_name": "quarterly_budget.xlsx",
  "device_id": "dev_macbook_pro_13",
  "previous_version": 4,
  "blocks": [
    { "hash": "sha256_hash_block_1", "position": 0, "size_bytes": 4194304 },
    { "hash": "sha256_hash_block_2", "position": 1, "size_bytes": 4194304 },
    { "hash": "sha256_hash_block_3_modified", "position": 2, "size_bytes": 1048576 }
  ]
}
```

### Response Payload
```json
{
  "object_id": "obj_99018247",
  "new_version": 5,
  "sync_status": "SUCCESS",
  "committed_at": "2026-06-08T12:46:05Z"
}
```

---

## 3. GET `/api/v1/sync/subscribe` (Long Polling Connection)

Opens an HTTP connection that the notification gateway hangs open to push sync notifications to listening client devices.

* **HTTP Method**: `GET`
* **Headers**: `Keep-Alive: timeout=60, max=100`

### Query Parameters
| Parameter | Data Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `device_id` | String | **Yes** | Unique client device ID. |
| `last_seen_tx_id` | String | **Yes** | Transaction watermark to catch up on missed events. |

### Response Headers
* `Status: 200 OK` (Triggered when a new sync event arrives, or `204 No Content` on timeout)
* `Content-Type: application/json`

### Response Payload
```json
{
  "sync_event": {
    "tx_id": "tx_9012480128",
    "space_id": "space_shared_450",
    "object_id": "obj_99018247",
    "action": "UPDATE",
    "version": 5,
    "triggered_by_device": "dev_macbook_pro_13"
  }
}
```
*If a timeout occurs before any events arrive, the server returns an empty `204 No Content` body, and the client immediately re-opens a new request.*
