# Distributed File Storage System

> **A scalable backend file storage platform supporting authentication, chunked uploads, resumable transfers, concurrent uploads, caching, background processing, and secure access.**

[![Java](https://img.shields.io/badge/Java-17+-orange?style=flat-square&logo=openjdk)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen?style=flat-square&logo=springboot)](https://spring.io/projects/spring-boot)
[![MySQL](https://img.shields.io/badge/MySQL-8.x-blue?style=flat-square&logo=mysql)](https://www.mysql.com/)
[![Redis](https://img.shields.io/badge/Redis-7.x-red?style=flat-square&logo=redis)](https://redis.io/)
[![Docker](https://img.shields.io/badge/Docker-Compose-blue?style=flat-square&logo=docker)](https://docs.docker.com/compose/)
[![License](https://img.shields.io/badge/License-MIT-lightgrey?style=flat-square)](LICENSE)

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Motivation](#motivation)
3. [Architecture](#architecture)
4. [Technology Stack](#technology-stack)
5. [Folder Structure](#folder-structure)
6. [Database Design](#database-design)
7. [Core Features](#core-features)
8. [API Reference](#api-reference)
9. [Upload Workflow](#upload-workflow)
10. [Download Workflow](#download-workflow)
11. [Concurrency Model](#concurrency-model)
12. [Caching Strategy](#caching-strategy)
13. [Security Design](#security-design)
14. [ACID Compliance](#acid-compliance)
15. [CS Fundamentals Demonstrated](#cs-fundamentals-demonstrated)
16. [Getting Started](#getting-started)
17. [Environment Configuration](#environment-configuration)
18. [Future Roadmap](#future-roadmap)
19. [Resume Impact](#resume-impact)

---

## Project Overview

The **Distributed File Storage System** is a production-inspired backend platform built with Java and Spring Boot. It goes well beyond basic CRUD: the system handles **large file uploads via chunking**, **resumable interrupted transfers**, **parallel concurrent uploads**, **JWT-based authentication**, **Redis metadata caching**, and **asynchronous background processing** for chunk merging and cleanup.

This project is intentionally built to demonstrate **engineering depth** — not the width of features. Every design decision maps to a core Computer Science concept: OS-level concurrency, DBMS transactions, system design trade-offs, and network protocol handling.

---

## Motivation

Most college projects demonstrate basic web development. This project is built to a different standard — it simulates real problems solved inside cloud storage infrastructure teams at companies like Google, Dropbox, and AWS.

The driving question was: **What would it take to build the backend of a file storage service from scratch, without relying on cloud SDKs, that still demonstrates production-grade patterns?**

The answer involved:

- Designing a metadata-storage separation strategy (only metadata in MySQL; raw bytes on disk)
- Implementing chunked upload with merge workers to handle files larger than memory
- Building a Redis caching layer to offload hot-path database reads
- Writing a resumable upload mechanism so interrupted transfers can continue without restarting
- Securing every endpoint with JWT and integrating Spring Security filters at the right layers

This project is the product of studying **Operating Systems, DBMS, Computer Networks, OOP, and System Design** and then applying those concepts inside a single coherent system.

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                     React Native Client                    │
└───────────────────────────┬────────────────────────────────┘
                            │ HTTPS / REST
                            ▼
┌────────────────────────────────────────────────────────────┐
│              Spring Boot REST API Layer                    │
│   AuthController  │  FileController  │  ChunkController   │
└───────────────────────────┬────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│              Spring Security — JWT Filter Chain            │
│     JwtAuthenticationFilter  →  SecurityFilterChain       │
└───────────────────────────┬────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│                     Service Layer                          │
│   AuthService │ FileService │ ChunkService │ CacheService  │
└──────┬──────────────┬──────────────┬────────────────┬──────┘
       │              │              │                │
       ▼              ▼              ▼                ▼
 ┌──────────┐  ┌──────────┐  ┌──────────┐   ┌──────────────┐
 │  MySQL   │  │  Redis   │  │  Local   │   │  Background  │
 │ Metadata │  │  Cache   │  │ Storage  │   │   Workers    │
 │  (JPA)   │  │  Layer   │  │ (Chunks) │   │(Async/Exec.) │
 └──────────┘  └──────────┘  └──────────┘   └──────────────┘
```

### Layer Responsibilities

| Layer | Responsibility |
|---|---|
| **REST Controllers** | Accept HTTP requests, validate input DTOs, delegate to service layer, return structured responses |
| **JWT Filter** | Intercept every request, extract and validate JWT token, inject authenticated principal into security context |
| **Service Layer** | Business logic, orchestration of DB + cache + storage, transaction boundaries |
| **Repository Layer** | Spring Data JPA interfaces; no raw SQL unless performance-critical |
| **MySQL** | Stores users, file metadata, chunk records, upload state — never raw bytes |
| **Redis** | Caches file metadata and upload state; reduces hot-path DB reads; supports TTL-based expiry |
| **Local Storage** | Stores raw chunk files on disk under structured paths; chunks merged by background workers |
| **Background Workers** | Async ExecutorService threads that merge chunks, clean up temp files, retry failed operations |

---

## Technology Stack

### Backend

| Technology | Purpose |
|---|---|
| Java 17+ | Primary language — strong typing, concurrency APIs, mature ecosystem |
| Spring Boot 3.x | Application framework — auto-configuration, embedded Tomcat, production-ready |
| Spring Security | Security filter chain, JWT integration, access control |
| Spring Data JPA | ORM abstraction over Hibernate; entity mapping, JPQL queries |
| Maven | Dependency management and build lifecycle |

### Data Layer

| Technology | Purpose |
|---|---|
| MySQL 8.x | Relational metadata store — ACID transactions, foreign key integrity |
| Redis 7.x | In-memory cache — metadata hot-path, upload state, TTL management |

### Infrastructure

| Technology | Purpose |
|---|---|
| Docker | Containerization of the Spring Boot application |
| Docker Compose | Orchestrates MySQL + Redis + App containers locally |

### Security

| Technology | Purpose |
|---|---|
| JWT (JSON Web Tokens) | Stateless authentication; signed tokens issued on login |
| BCrypt | Adaptive password hashing; resistant to brute-force attacks |

---

## Folder Structure

```
src/
└── main/
    └── java/
        └── com.example.storage/
            │
            ├── config/               # Spring beans, Redis config, Async executor config
            │   ├── RedisConfig.java
            │   ├── AsyncConfig.java
            │   └── SecurityConfig.java
            │
            ├── controller/           # REST endpoints — thin layer, delegates to services
            │   ├── AuthController.java
            │   ├── FileController.java
            │   └── ChunkController.java
            │
            ├── dto/                  # Request/Response objects — decoupled from entities
            │   ├── LoginRequest.java
            │   ├── RegisterRequest.java
            │   ├── FileMetadataResponse.java
            │   └── ChunkUploadRequest.java
            │
            ├── entity/               # JPA entities — map to MySQL tables
            │   ├── User.java
            │   ├── FileMetadata.java
            │   ├── Chunk.java
            │   └── UploadState.java
            │
            ├── exception/            # Global exception handling
            │   ├── GlobalExceptionHandler.java
            │   ├── FileNotFoundException.java
            │   └── UnauthorizedAccessException.java
            │
            ├── repository/           # Spring Data JPA interfaces
            │   ├── UserRepository.java
            │   ├── FileRepository.java
            │   ├── ChunkRepository.java
            │   └── UploadStateRepository.java
            │
            ├── security/             # JWT utilities and filter
            │   ├── JwtUtil.java
            │   └── JwtAuthenticationFilter.java
            │
            ├── service/              # Business logic and orchestration
            │   ├── AuthService.java
            │   ├── FileService.java
            │   ├── ChunkService.java
            │   ├── CacheService.java
            │   └── BackgroundWorkerService.java
            │
            └── util/                 # Shared utility classes
                ├── FileStorageUtil.java
                └── HashUtil.java

resources/
└── application.properties            # All environment configuration
```

### Why This Structure

This layout follows **Clean Architecture** principles:

- **Controllers** only know about DTOs and service interfaces — they never touch repositories or storage directly.
- **Services** contain all business rules and orchestrate across the persistence, cache, and storage layers.
- **Entities** are only used inside the persistence boundary — DTOs prevent entity leakage into the API layer.
- **Config classes** centralize infrastructure wiring, keeping business code free of framework boilerplate.
- **Exception handling** is centralized in `GlobalExceptionHandler` using `@ControllerAdvice`, so error formatting is consistent across all endpoints.

---

## Database Design

### Schema Overview

```
┌─────────────┐       ┌──────────────────┐       ┌───────────┐
│    users    │──1:N──│  file_metadata   │──1:N──│  chunks   │
└─────────────┘       └──────────────────┘       └───────────┘
      │                        │
      │                        │1:1
      │                ┌───────────────┐
      │                │ upload_state  │
      └────────────────┘
```

### Table Definitions

#### `users`

```sql
CREATE TABLE users (
    id           BIGINT       PRIMARY KEY AUTO_INCREMENT,
    username     VARCHAR(100) NOT NULL UNIQUE,
    email        VARCHAR(255) NOT NULL UNIQUE,
    password     VARCHAR(255) NOT NULL,              -- BCrypt hash
    created_at   TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_email (email)
);
```

#### `file_metadata`

```sql
CREATE TABLE file_metadata (
    id            BIGINT        PRIMARY KEY AUTO_INCREMENT,
    user_id       BIGINT        NOT NULL,
    file_name     VARCHAR(255)  NOT NULL,
    file_size     BIGINT        NOT NULL,             -- bytes
    content_type  VARCHAR(100),
    storage_path  VARCHAR(512),                       -- path on disk (after merge)
    checksum      VARCHAR(64),                        -- SHA-256 for integrity
    status        ENUM('UPLOADING','COMPLETE','FAILED') DEFAULT 'UPLOADING',
    created_at    TIMESTAMP     DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status)
);
```

#### `chunks`

```sql
CREATE TABLE chunks (
    id            BIGINT    PRIMARY KEY AUTO_INCREMENT,
    file_id       BIGINT    NOT NULL,
    chunk_index   INT       NOT NULL,
    chunk_path    VARCHAR(512),
    size          BIGINT,
    uploaded_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (file_id) REFERENCES file_metadata(id) ON DELETE CASCADE,
    UNIQUE KEY uq_file_chunk (file_id, chunk_index),
    INDEX idx_file_id (file_id)
);
```

#### `upload_state`

```sql
CREATE TABLE upload_state (
    id              BIGINT    PRIMARY KEY AUTO_INCREMENT,
    file_id         BIGINT    NOT NULL UNIQUE,
    total_chunks    INT       NOT NULL,
    received_chunks INT       DEFAULT 0,
    last_updated    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (file_id) REFERENCES file_metadata(id) ON DELETE CASCADE
);
```

### Design Decisions

**Metadata–Storage Separation:** Raw file bytes are never stored in MySQL. MySQL stores only structured metadata. This is the same pattern used in systems like HDFS (NameNode for metadata, DataNode for bytes). It keeps the database lean, allows independent scaling of storage, and keeps queries fast.

**Normalization:** The schema is in Third Normal Form (3NF). Upload state is separated from file metadata to avoid update anomalies when chunk counts change during an ongoing upload.

**Indexing:** Indexes on `user_id` in `file_metadata` and `file_id` in `chunks` are critical — list-files and chunk-lookup queries are the hottest read paths, and without indexes these would be full table scans.

**Checksum column:** SHA-256 stored post-merge allows integrity verification on download without re-reading the entire file.

---

## Core Features

### Authentication

- User registration with BCrypt-hashed passwords
- Login endpoint issues a signed JWT token (configurable expiry)
- Every protected endpoint passes through `JwtAuthenticationFilter`, which validates the token and injects the user principal into Spring Security's context
- Stateless design — no server-side session storage

### File Upload

- Accepts `multipart/form-data` requests
- Validates file type and size before accepting
- Stores file metadata in MySQL immediately (status: `UPLOADING`)
- Writes raw bytes to local storage under a structured path: `uploads/{userId}/{fileId}/`
- For small files: direct single-shot write, status updated to `COMPLETE`
- For large files: triggers chunked upload pipeline

### File Download

- Authorization check — only the owning user (or authorized roles in future) can download
- Redis cache checked first for metadata
- On cache hit: file path resolved immediately, stream initiated
- On cache miss: metadata fetched from MySQL, cached in Redis with TTL, then file streamed
- Uses `StreamingResponseBody` for memory-efficient streaming (no full file load into heap)

### Chunked Upload

Large files are split client-side into fixed-size chunks (e.g., 5 MB each). Each chunk is independently uploaded:

```
POST /api/files/{fileId}/chunks/{chunkIndex}
Content-Type: multipart/form-data
Body: chunk binary data
```

Each chunk is:
1. Written to disk at `uploads/{userId}/{fileId}/chunks/{chunkIndex}`
2. Recorded in the `chunks` table
3. `upload_state.received_chunks` incremented atomically

When `received_chunks == total_chunks`, a background merge job is triggered.

### Resumable Upload

On connection failure or client restart:

```
GET /api/files/{fileId}/upload-state
```

Returns the list of already-received chunk indices. The client re-uploads only missing chunks. This avoids full re-upload of large files and is the same mechanism used by the TUS resumable upload protocol.

### Concurrent Uploads

Multiple users (or even the same user uploading multiple files) are handled in parallel. Thread-safety is ensured at multiple levels:

- Each upload writes to an isolated directory path (keyed on `userId/fileId`)
- `upload_state.received_chunks` updates are protected by `@Transactional` with pessimistic locking
- The background worker pool is managed by a configured `ThreadPoolTaskExecutor`

### Redis Caching

| Cache Key Pattern | Value | TTL |
|---|---|---|
| `file:meta:{fileId}` | Serialized `FileMetadataResponse` | 10 minutes |
| `upload:state:{fileId}` | Serialized `UploadState` | 30 minutes |
| `user:files:{userId}` | List of file IDs | 5 minutes |

Cache invalidation follows the **write-through** pattern: on every metadata update, the corresponding Redis key is updated simultaneously inside the same `@Transactional` service method.

### Background Workers

An `ExecutorService`-backed worker pool handles:

- **Chunk Merge Job:** Triggered when all chunks are received. Opens chunks in order, concatenates into final file, updates `storage_path` and status in MySQL, then purges chunk files.
- **Cleanup Job:** Scheduled with `@Scheduled`, runs periodically to delete orphaned chunk directories from stale or failed uploads.
- **Retry Job:** Checks for files stuck in `UPLOADING` status beyond a threshold and re-triggers merge or flags them as `FAILED`.

---

## API Reference

### Authentication Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/auth/register` | Register a new user | No |
| `POST` | `/api/auth/login` | Authenticate and receive JWT | No |

### File Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/files/upload` | Upload a complete file | Yes |
| `GET` | `/api/files` | List all files for authenticated user | Yes |
| `GET` | `/api/files/{fileId}` | Get file metadata | Yes |
| `GET` | `/api/files/{fileId}/download` | Download file content | Yes |
| `PATCH` | `/api/files/{fileId}/rename` | Rename a file | Yes |
| `DELETE` | `/api/files/{fileId}` | Delete file and metadata | Yes |

### Chunked Upload Endpoints

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/files/chunked/init` | Initialise a chunked upload session | Yes |
| `POST` | `/api/files/{fileId}/chunks/{chunkIndex}` | Upload a single chunk | Yes |
| `GET` | `/api/files/{fileId}/upload-state` | Get received chunk indices (resume support) | Yes |
| `POST` | `/api/files/{fileId}/complete` | Manually trigger merge (optional) | Yes |

### Example: Login Response

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "expiresIn": 3600,
  "userId": 42,
  "username": "tushar"
}
```

### Example: File Metadata Response

```json
{
  "fileId": 101,
  "fileName": "report.pdf",
  "fileSize": 10485760,
  "contentType": "application/pdf",
  "status": "COMPLETE",
  "checksum": "a3f5d...",
  "uploadedAt": "2025-08-15T10:30:00Z"
}
```

---

## Upload Workflow

```
User Login
    │
    ▼
POST /api/auth/login  →  JWT Token Issued
    │
    ▼
Client selects file
    │
    ├──[Small File]─────────────────────────────────────────────┐
    │                                                           │
    ▼                                                           ▼
POST /api/files/upload                         POST /api/files/chunked/init
(multipart/form-data)                          (total_chunks, file_name, size)
    │                                                           │
    ▼                                                           ▼
Metadata inserted in MySQL                     FileMetadata created (status: UPLOADING)
(status: UPLOADING)                            UploadState record created
    │                                                           │
    ▼                                                           ▼
File written to uploads/{uid}/{fid}/           Client loops:
    │                                          POST /chunks/{index}  (per chunk)
    ▼                                                           │
MySQL updated (status: COMPLETE)                               ▼
    │                                          Chunk written to disk
    ▼                                          Chunk record inserted in MySQL
Redis cache updated                            upload_state.received_chunks++
    │                                                           │
    ▼                                          When all chunks received:
Success Response                                               │
                                                               ▼
                                               BackgroundWorkerService.mergechunks()
                                                               │
                                                               ▼
                                               Chunks concatenated → final file
                                               storage_path set, status: COMPLETE
                                                               │
                                                               ▼
                                               Temp chunks purged
                                               Redis cache updated
```

---

## Download Workflow

```
GET /api/files/{fileId}/download
    │
    ▼
JwtAuthenticationFilter validates token
    │
    ▼
Authorization check: does this user own fileId?
    │
    ├── No  → 403 Forbidden
    │
    ▼
Redis lookup: file:meta:{fileId}
    │
    ├── Cache HIT  →  Resolve storage_path from cached metadata
    │                         │
    │                         ▼
    │                 Stream file bytes to client
    │
    └── Cache MISS →  Query MySQL for FileMetadata
                              │
                              ▼
                      Store in Redis (TTL: 10 min)
                              │
                              ▼
                      Stream file bytes to client
                      (StreamingResponseBody — no heap buffering)
```

---

## Concurrency Model

Concurrency is handled at three levels:

### Thread Pool Configuration

A dedicated `ThreadPoolTaskExecutor` is configured in `AsyncConfig`:

```java
@Bean(name = "fileProcessingExecutor")
public Executor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(10);
    executor.setQueueCapacity(50);
    executor.setThreadNamePrefix("StorageWorker-");
    executor.initialize();
    return executor;
}
```

### CompletableFuture for Async Merge

```java
@Async("fileProcessingExecutor")
public CompletableFuture<Void> mergeChunks(Long fileId) {
    // merge logic
    return CompletableFuture.completedFuture(null);
}
```

### Isolation via Path Namespacing

Since every upload writes to `uploads/{userId}/{fileId}/`, concurrent uploads by the same user or different users never share a directory. No locking is needed at the storage layer.

### Transactional Counter Updates

`upload_state.received_chunks` is incremented using a `@Transactional` service call with optimistic locking (`@Version` on the entity), preventing lost updates when two chunk uploads for the same file arrive simultaneously.

---

## Caching Strategy

Redis is used as a **read-through, write-through cache** for the metadata hot path.

### Read Path

```
Request → Redis lookup
    ├── HIT  → Return cached value (no DB hit)
    └── MISS → Read from MySQL → Store in Redis → Return value
```

### Write Path

Every mutation to `FileMetadata` or `UploadState` updates Redis within the same `@Transactional` block:

```java
@Transactional
public void updateFileName(Long fileId, String newName) {
    FileMetadata meta = fileRepository.findById(fileId).orElseThrow();
    meta.setFileName(newName);
    fileRepository.save(meta);
    cacheService.evict("file:meta:" + fileId);   // invalidate stale cache
}
```

### TTL Rationale

- File metadata TTL: **10 minutes** — infrequently mutated, safe to cache aggressively
- Upload state TTL: **30 minutes** — active during ongoing uploads; auto-expires after completion
- User file list TTL: **5 minutes** — shorter because list changes on every new upload

---

## Security Design

### JWT Token Flow

```
POST /api/auth/login
    │
    ▼
Credentials validated against MySQL (BCrypt compare)
    │
    ▼
JWT generated: { sub: userId, iat: now, exp: now+3600 }
Signed with HS256 + secret key
    │
    ▼
Returned to client

Subsequent Requests:
Authorization: Bearer <token>
    │
    ▼
JwtAuthenticationFilter extracts token
    │
    ▼
Signature verified, expiry checked
    │
    ▼
UsernamePasswordAuthenticationToken injected into SecurityContext
```

### Security Layers

| Layer | Mechanism |
|---|---|
| Password storage | BCrypt (adaptive rounds, salted) |
| API authentication | JWT, verified on every request |
| Endpoint protection | Spring Security `HttpSecurity` rules |
| Ownership enforcement | Service layer checks `file.userId == authenticatedUserId` |
| Error masking | `GlobalExceptionHandler` never exposes stack traces to clients |

---

## ACID Compliance

All critical operations use Spring's `@Transactional` annotation to guarantee ACID properties at the database level.

### Atomicity

When a file upload completes, metadata update and storage path assignment happen in a single transaction. If the storage write fails, the transaction rolls back — the DB never reflects a file that doesn't exist on disk.

### Consistency

Foreign key constraints (e.g., `chunks.file_id → file_metadata.id`) are enforced at the DB level. JPA validation annotations enforce constraints at the application level before any query is issued.

### Isolation

Concurrent chunk uploads for the same file use optimistic locking (`@Version`) to detect and retry conflicting counter increments. This avoids phantom reads on `received_chunks`.

### Durability

MySQL with InnoDB engine guarantees that committed transactions survive restarts. Redis is configured with AOF (append-only file) persistence for upload state durability.

---

## CS Fundamentals Demonstrated

### Operating Systems
- **Concurrency:** Multiple uploads handled in parallel via `ExecutorService` thread pools
- **Synchronization:** Optimistic locking prevents lost updates on shared upload counters
- **Producer-Consumer:** Chunk upload is the producer; the merge worker is the consumer, decoupled via async dispatch
- **Background threads:** `@Scheduled` cleanup jobs run independently of request threads

### Computer Networks
- **HTTP/HTTPS:** REST API over HTTP with proper status codes (200, 201, 400, 401, 403, 404, 500)
- **Multipart transfers:** `multipart/form-data` for binary chunk upload
- **Streaming:** `StreamingResponseBody` avoids loading entire files into memory during download
- **Stateless protocol:** JWT enables stateless authentication over HTTP

### DBMS
- **CRUD:** All four operations implemented across four tables
- **ACID:** Enforced via `@Transactional` and InnoDB
- **Normalization:** Schema in 3NF
- **Indexing:** Selective indexes on hot-path foreign keys
- **Transactions:** Rollback on partial failures

### Object-Oriented Programming
- **Layered architecture:** Controller → Service → Repository separation of concerns
- **Dependency Injection:** Spring IoC container manages all bean lifecycles
- **Abstraction:** Service interfaces decouple controllers from implementations
- **Encapsulation:** Entities never leak into the API layer (DTOs used instead)

### System Design
- **Caching:** Redis layer with TTL and write-through invalidation
- **Chunking:** File splitting for large-file handling (mirrors S3 multipart upload design)
- **Resumability:** State tracking enables partial upload continuation
- **Scalability patterns:** Stateless auth, metadata-storage separation, async workers
- **Fault tolerance:** Retry jobs, cleanup workers, graceful error responses

---

## Getting Started

### Prerequisites

- Java 17+
- Docker and Docker Compose
- Maven 3.8+

### Clone and Run

```bash
git clone https://github.com/yourusername/distributed-file-storage.git
cd distributed-file-storage

# Start MySQL and Redis via Docker Compose
docker-compose up -d

# Build and run the Spring Boot app
mvn clean install
mvn spring-boot:run
```

The server starts on `http://localhost:8080`.

### Docker Compose Services

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: filestorage
    ports:
      - "3306:3306"

  redis:
    image: redis:7.0
    ports:
      - "6379:6379"
    command: ["redis-server", "--appendonly", "yes"]

  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - mysql
      - redis
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/filestorage
      SPRING_REDIS_HOST: redis
```

---

## Environment Configuration

```properties
# application.properties

# Server
server.port=8080

# MySQL
spring.datasource.url=jdbc:mysql://localhost:3306/filestorage
spring.datasource.username=root
spring.datasource.password=root
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect

# Redis
spring.redis.host=localhost
spring.redis.port=6379

# JWT
jwt.secret=your-256-bit-secret
jwt.expiration=3600000

# Storage
storage.base-path=./uploads
storage.max-chunk-size=5242880

# Thread Pool
async.core-pool-size=4
async.max-pool-size=10
async.queue-capacity=50
```

---

## Future Roadmap

### Near-term

| Feature | Rationale |
|---|---|
| Role-based access control | Owner / viewer / admin roles per file |
| Signed download URLs | Time-limited, shareable links without exposing JWT |
| File versioning | Track and restore previous versions |
| Rate limiting | Per-user upload rate limits via `Bucket4j` |

### Medium-term

| Feature | Rationale |
|---|---|
| Message queue integration | Replace in-process async with RabbitMQ/Kafka for distributed merge workers |
| Cloud object storage | Swap local disk with AWS S3 / Azure Blob via strategy pattern |
| File deduplication | SHA-256 content-hash comparison before storing duplicate files |
| Content compression | GZIP compression for text-heavy uploads |

### Long-term

| Feature | Rationale |
|---|---|
| Multi-node storage | Distribute chunk storage across multiple servers (mini-HDFS pattern) |
| Replication factor | Store N copies of each file across nodes for fault tolerance |
| Redis Pub/Sub | Broadcast merge completion events across nodes |
| Observability stack | Prometheus metrics + Grafana dashboards + distributed tracing (Jaeger) |
| Microservices split | Decompose into Auth Service, Upload Service, Download Service, Worker Service |
| CDN integration | Edge-cache frequently accessed files |

---

## Concepts Implementation

| **System Design** | Architecture diagram, caching strategy, chunking, metadata separation |
| **OS — Concurrency** | ThreadPoolTaskExecutor, CompletableFuture, optimistic locking |
| **DBMS — Transactions** | @Transactional, ACID discussion, normalization, indexing |
| **Computer Networks** | REST API design, multipart upload, JWT over HTTP, streaming |
| **OOP & Design Patterns** | Layered architecture, DI, strategy (storage backends), observer (async workers) |
| **Security** | JWT authentication, BCrypt, Spring Security filter chain |
| **Fault Tolerance** | Resumable uploads, retry workers, cleanup jobs |
| **Backend Engineering** | Spring Boot, JPA, Redis, Docker, Maven |

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

---
