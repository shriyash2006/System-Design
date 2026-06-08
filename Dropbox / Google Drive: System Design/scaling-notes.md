# Scaling Strategy: Block Chunking, Client-Side Sync, and Deduplication

Scaling a distributed cloud storage platform to support 500 million Daily Active Users (DAU) and 270 Petabytes of 10-year storage requires optimizing network transfers, indexing block histories, and using space-saving storage tiers.

---

## 1. File Chunking & Block-Level Synchronization

Uploading a large file entirely (e.g. 500 MB) on every small edit is highly inefficient. We implement **Block Chunking** to optimize transfers:

* **Fixed Block Splitting**: Files are split into blocks of a fixed size, commonly **4 MB**.
* **Differential Upload**: When a user modifies a file, only the modified blocks are compressed (Huffman coding), encrypted (AES-256), and uploaded to the Block Service (Amazon S3).
* **Metadata Assembly**: The Metadata Service updates the database to point to the new block hashes at their respective index positions.

---

## 2. Client-Side Subsystem Architecture

The client application runs background services to handle the workload before sending data to the server:

```
                  [ Designated Local Folder ]
                               │
                       (Monitor Scan)
                               │
                               ▼
                        [ Blockifier ]
                  (Splits file into 4MB blocks)
                               │
                               ▼
                         [ SQLite DB ]  ◄──► [ Synchronizer ]
                 (Local block map & hashes)     (Transfers blocks/metadata)
```

1. **Monitor**: Constantly monitors the local sync folder for file system events (create, modify, delete).
2. **Blockifier**:
   * Splits files into 4 MB chunks on write.
   * Reassembles chunks into full files on download.
3. **Client SQLite Database**: A lightweight database engine running locally on the device to record the hash, version, and sync status of every block. This prevents redundant hashes computation on startup.
4. **Synchronizer**: Handles parallel uploads/downloads of blocks to/from the cloud.

---

## 3. Multi-Device Sync & Notification Pipeline

When Client A uploads a file block map, Client B must update its local copy:

1. **Upload Phase**: Client A's synchronizer uploads the updated blocks to the Block Service (S3) and sends the block metadata map to the Metadata Service.
2. **Kafka Event Broker**: The Metadata Service writes a sync event (e.g., `Update File X to Version 5`) to **Apache Kafka**.
3. **Queue Distribution**: Kafka consumers push events into dedicated user notification queues.
4. **Long Polling Notification**:
   * Client B maintains a persistent connection via **Long Polling** to the Notification Service.
   * When the event lands in its queue, the Notification Service responds.
   * If no event arrives within 50 seconds, the server returns an empty `204 No Content` response, and Client B re-opens a new request.
5. **Download Phase**: Client B parses the sync notification, queries the Metadata Service for the version 5 block map, downloads only the missing 4 MB blocks, and reassembles the file.

---

## 4. Global Block Deduplication

To optimize the 270 PB storage footprint, the Block Service implements **Global Deduplication**:

* When a client attempts to upload block `sha256_hash_XYZ`, the Block Service queries the global metadata index.
* If that exact block hash is already stored (from another file or another user), the Block Service returns a success code immediately and skips the raw S3 upload.
* The Metadata Service simply creates a new pointer linking the user's file version to the existing S3 block.
