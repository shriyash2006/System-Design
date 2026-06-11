# API Design: Messaging App System

This document specifies the communication interfaces for the real-time chat application, detailing REST API endpoints, WebSocket connection packets, and UDP presence heartbeats.

---

## 1. WebSocket Protocol & Handshake

WebSocket duplex connections are initiated over standard HTTPS and upgraded to the binary WS protocol.

### Handshake Ingress
* **HTTP Method**: `GET`
* **Path**: `/api/v1/chat/connect`
* **Request Headers**:
  * `Connection: Upgrade`
  * `Upgrade: websocket`
  * `Sec-WebSocket-Version: 13`
  * `Authorization: Bearer <jwt_token>`
  * `X-Device-ID: dev_iphone_15_pro`

---

## 2. WebSocket Packet Contracts

Once established, all communication utilizes JSON frames over the WebSocket channel.

### A. Client-to-Server (SendMessage Packet)
Client sends a text packet to be delivered to a recipient conversation.

```json
{
  "event": "send_message",
  "client_msg_id": "client_uuid_abc123",
  "conversation_id": "conv_990284712",
  "content": "Hey! Did you check out the new design specifications?",
  "media_id": null
}
```

| Field Name | Data Type | Required | Description |
| :--- | :---: | :---: | :--- |
| `event` | String | **Yes** | Payload event type. Must be `send_message`. |
| `client_msg_id` | String | **Yes** | Client-generated temporary UUID to map client-side state before Snowflake ID confirmation. |
| `conversation_id` | String | **Yes** | Target conversation (1-to-1 DM or group channel). |
| `content` | String | **Yes** | UTF-8 encoded text body (max 4000 characters). |
| `media_id` | String | No | Optional pre-uploaded object key. |

### B. Server-to-Client (MessageDelivery Packet)
The gateway pushes incoming packets to active connected users.

```json
{
  "event": "new_message",
  "message_id": "snowflake_179240182470129",
  "conversation_id": "conv_990284712",
  "sender_id": "user_4429018",
  "content": "Hey! Did you check out the new design specifications?",
  "media_url": null,
  "created_at": "2026-06-12T01:22:00Z"
}
```

---

## 3. Presence Heartbeat Packet (UDP / TCP)

Clients verify active state by sending a binary heartbeat ping packet to the Presence Server gateway every 5 seconds.

* **Protocol**: UDP (Preferred for speed and low overhead) or TCP (Fallback).
* **Payload Structure (Binary Struct)**:
```text
+-----------------------+-----------------------+
|  User ID (64-bit int) | Device ID (64-bit int)|
+-----------------------+-----------------------+
|   Sequence (32-bit)   |  Timestamp (64-bit)   |
+-----------------------+-----------------------+
```

---

## 4. POST `/api/v1/media/upload` (Stateless Media Ingest)

Clients upload binary files to S3 before referencing their media identifiers in chat packets.

* **HTTP Method**: `POST`
* **Content-Type**: `multipart/form-data`

### Request Payload
* **Form-Data**: `file: <binary_data>`

### Response Payload
```json
{
  "media_id": "media_whatsapp_9921472",
  "mime_type": "image/jpeg",
  "size_bytes": 102400,
  "uploaded_at": "2026-06-12T01:22:05Z"
}
```
*Clients take this `media_id` and attach it to their WebSocket `send_message` payload.*
