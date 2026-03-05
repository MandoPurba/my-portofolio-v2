---
layout: ../../layouts/post.astro
title: "Building Real-Time Database Synchronization with Rust and PostgreSQL LISTEN/NOTIFY"
pubDate: 2025-03-04
description: "How we built a lightweight, production-ready service that synchronizes data between two PostgreSQL databases in real time — using Rust, tokio, sqlx, and PostgreSQL's built-in LISTEN/NOTIFY mechanism. No Kafka, no RabbitMQ, no Redis — just a single static binary and PostgreSQL's native pub/sub."
author: "romando"
isPinned: true
excerpt: "A deep dive into building a real-time data synchronization service between two PostgreSQL databases using Rust, Tokio, SQLx, and PostgreSQL's LISTEN/NOTIFY mechanism. We explore trigger-based conditional notifications, async broadcast channels, multi-stage Docker builds, and graceful permission handling in production environments."
image:
  src:
  alt:
tags: ["rust", "postgresql", "database-sync", "tokio", "sqlx", "listen-notify", "docker", "real-time"]
---
# Building Real-Time Database Synchronization with Rust and PostgreSQL LISTEN/NOTIFY

> How we built a lightweight, production-ready service that synchronizes data between two PostgreSQL databases in real time — using Rust, `tokio`, `sqlx`, and PostgreSQL's built-in LISTEN/NOTIFY mechanism.

---

## Table of Contents

1. [The Problem](#the-problem)
2. [Why Not Just Use Logical Replication?](#why-not-just-use-logical-replication)
3. [Architecture Overview](#architecture-overview)
4. [PostgreSQL LISTEN/NOTIFY — A Quick Primer](#postgresql-listennotify--a-quick-primer)
5. [Project Structure](#project-structure)
6. [Deep Dive: The Source Database Trigger](#deep-dive-the-source-database-trigger)
7. [Deep Dive: The Listener](#deep-dive-the-listener)
8. [Deep Dive: The Sync Engine](#deep-dive-the-sync-engine)
9. [Connecting It All Together](#connecting-it-all-together)
10. [Configuration and Environment](#configuration-and-environment)
11. [Dockerizing the Service](#dockerizing-the-service)
12. [CI/CD with GitLab](#cicd-with-gitlab)
13. [Handling Permission Constraints in Production](#handling-permission-constraints-in-production)
14. [Lessons Learned](#lessons-learned)
15. [Conclusion](#conclusion)

---

## The Problem

In our organization, we manage a **tutor registration system** (BIMON) where tutor data lives in a source PostgreSQL database. However, a separate downstream system needs access to a **subset of verified tutor records** — specifically, tutors who have passed all verification checks (identity, diploma, status, etc.).

We needed a solution that:

- Synchronizes data **in real time** (not batch ETL)
- Only forwards records that meet **specific business conditions**
- Is **lightweight** and resource-efficient
- Can run reliably in a **Docker Swarm** production environment
- Handles **permission-restricted** database environments gracefully

---

## Why Not Just Use Logical Replication?

PostgreSQL's built-in logical replication is powerful, but it comes with trade-offs:

| Concern | Logical Replication | Our Approach |
|---|---|---|
| **Conditional filtering** | Limited (requires pgoutput plugins or custom logic) | Built into the trigger — only verified tutors are synced |
| **Schema coupling** | Source and target must have matching schemas | We transform data during sync — schemas can differ |
| **Permission requirements** | Requires `REPLICATION` role and `pg_hba.conf` changes | Works with standard `SELECT`/`INSERT`/`UPDATE` permissions |
| **Operational complexity** | Replication slots, WAL retention, monitoring | A single lightweight binary |
| **Field-level control** | Replicates entire rows | We pick exactly which fields to sync |

For our use case, a trigger-based approach with PostgreSQL's `LISTEN/NOTIFY` was the sweet spot.

---

## Architecture Overview

Here's how the system works at a high level:

```
┌──────────────────────────────────────────────────────────────────┐
│                        bimon-sync-data                           │
│                                                                  │
│  ┌─────────────┐    broadcast::channel    ┌─────────────┐        │
│  │  PgListen   │ ───── TutorData ───────> │   PgSync    │        │
│  │  (listener) │                          │  (syncer)   │        │
│  └──────┬──────┘                          └──────┬──────┘        │
│         │                                        │               │
└─────────┼────────────────────────────────────────┼───────────────┘
          │                                        │
          │ LISTEN "data_change_tutor"             │ INSERT / UPDATE
          │                                        │
  ┌───────▼───────┐                        ┌───────▼───────┐
  │  Source DB    │                        │  Target DB    │
  │  (m_tutor)    │                        │ (m_tutor_ta)  │
  │  + trigger    │                        │               │
  └───────────────┘                        └───────────────┘
```

The flow is straightforward:

1. A **trigger** on the source database's `m_tutor` table fires on every `INSERT` or `UPDATE`.
2. The trigger checks business conditions and, if met, sends a **`pg_notify`** with the tutor's data as a JSON payload.
3. The Rust application's **`PgListen`** module receives the notification in real time.
4. The payload is deserialized into a `TutorData` struct and sent through a **Tokio broadcast channel**.
5. The **`PgSync`** module receives the data and executes the appropriate `INSERT` or `UPDATE` on the target database.

---

## PostgreSQL LISTEN/NOTIFY — A Quick Primer

PostgreSQL has a built-in pub/sub mechanism that most people overlook:

- **`NOTIFY channel, 'payload'`** — sends a message to a named channel.
- **`LISTEN channel`** — subscribes to messages on that channel.
- **`pg_notify(channel, payload)`** — the function equivalent of `NOTIFY`, usable inside triggers and PL/pgSQL.

Key characteristics:

- **Transactional**: Notifications are only delivered when the transaction that fired them commits. If the transaction rolls back, the notification is discarded.
- **Lightweight**: No WAL involvement, no replication slots, no disk I/O for the notification itself.
- **Payload limit**: 8000 bytes per notification (more than enough for a JSON record).

This makes it **perfect** for event-driven architectures where you need to react to database changes without polling.

---

## Project Structure

```
bimon-sync-data/
├── Cargo.toml              # Dependencies and project metadata
├── Cargo.lock              # Locked dependency versions
├── Dockerfile              # Multi-stage build for minimal image
├── compose.yaml            # Docker Compose for local development
├── .gitlab-ci.yml          # CI/CD pipeline definition
├── log4rs.yaml             # Logging configuration
├── sql/
│   ├── source.sql          # Trigger and function for source DB
│   └── target.sql          # Target table schema
└── src/
    ├── main.rs             # Application entry point
    ├── config.rs           # Configuration struct
    ├── model/
    │   ├── mod.rs
    │   └── tutor.rs        # TutorData struct
    └── event/
        ├── mod.rs
        ├── listen.rs       # PostgreSQL listener (source DB)
        └── sync.rs         # Data synchronization (target DB)
```

The dependency footprint is intentionally small:

| Crate | Purpose |
|---|---|
| `tokio` | Async runtime with full features |
| `sqlx` | Async PostgreSQL driver with compile-time checked queries |
| `serde` / `serde_json` | JSON serialization/deserialization |
| `log` / `log4rs` | Structured logging |
| `config` | Environment-based configuration |
| `dotenv` | `.env` file support for local development |

---

## Deep Dive: The Source Database Trigger

The heart of the system is a PostgreSQL trigger function that fires on the source database:

```sql
CREATE OR REPLACE FUNCTION notify_tutor_change() RETURNS TRIGGER AS $$
BEGIN
    IF (NEW.status = '1' AND
        NEW.status_ijazah = '1' AND
        NEW.status_ktp = '1' AND
        NEW.verifikasi = '1' AND
        NEW.status_tutor = '1') THEN
        PERFORM pg_notify('data_change_tutor', json_build_object(
            'operation', TG_OP,
            'kode_tutor', NEW.id_tutor,
            'nama', NEW.nama_lengkap,
            'nidn', NEW.nidn,
            'alamat_email', NEW.email,
            'nomor_hp', NEW.telepon,
            'institusi', NEW.institusi,
            'nip', NEW.nip,
            'status_aktif', CASE WHEN NEW.status = '1' THEN 1 ELSE 0 END,
            'created_by', NEW.created_by,
            'created_at', NEW.created_at,
            'updated_by', NEW.updated_by,
            'updated_at', NEW.updated_at
        )::text);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Key design decisions:**

1. **Conditional notification**: The `IF` block ensures we only notify when *all five* verification flags are set to `'1'`. This means the downstream system only ever receives fully-verified tutor records.

2. **`json_build_object`**: We construct a JSON payload right inside the trigger. This gives us full control over field naming — notice how `NEW.id_tutor` becomes `kode_tutor` and `NEW.nama_lengkap` becomes `nama`. The source and target schemas don't need to match.

3. **`TG_OP`**: PostgreSQL's built-in variable that tells us whether this was an `INSERT` or `UPDATE`, which the sync engine uses to decide the correct SQL operation on the target.

4. **`PERFORM pg_notify(...)`**: We use `PERFORM` instead of `SELECT` because we don't need the return value. This is idiomatic PL/pgSQL.

The trigger itself is attached like this:

```sql
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM pg_trigger WHERE tgname = 'tutor_notify_trigger'
    ) THEN
        CREATE TRIGGER tutor_notify_trigger
        AFTER INSERT OR UPDATE ON m_tutor
        FOR EACH ROW EXECUTE FUNCTION notify_tutor_change();
    END IF;
END $$;
```

The `IF NOT EXISTS` guard makes this idempotent — safe to run multiple times without error.

---

## Deep Dive: The Listener

The `PgListen` struct in `src/event/listen.rs` manages the connection to the source database and listens for notifications:

```rust
pub struct PgListen {
    pool: Pool<Postgres>,
    sender: Arc<Sender<TutorData>>,
}
```

It holds two things: a connection pool to the source database and an `Arc`-wrapped broadcast sender to forward events to the sync engine.

The listening loop is elegant in its simplicity:

```rust
pub async fn start_listening(&self) -> Result<(), sqlx::Error> {
    let mut listener = sqlx::postgres::PgListener::connect_with(&self.pool).await?;
    listener.listen("data_change_tutor").await?;

    info!("Listening for database changes...");

    while let Ok(notification) = listener.recv().await {
        if let Ok(payload) = serde_json::from_str::<TutorData>(notification.payload()) {
            debug!("{:?}", payload);
            let _ = self.sender.send(payload);
        }
    }
    Ok(())
}
```

**What's happening:**

1. **`PgListener::connect_with`** creates a dedicated PostgreSQL connection for listening. This is separate from the connection pool — important because `LISTEN` requires a persistent connection.
2. **`listener.listen("data_change_tutor")`** subscribes to the notification channel.
3. The `while let` loop blocks asynchronously until a notification arrives, then deserializes the JSON payload into a `TutorData` struct.
4. The deserialized data is sent through the broadcast channel with `self.sender.send(payload)`.

`sqlx`'s `PgListener` handles **automatic reconnection** under the hood — if the connection drops, it will attempt to reconnect and re-subscribe, which is critical for a long-running service.

---

## Deep Dive: The Sync Engine

The `PgSync` struct in `src/event/sync.rs` receives tutor data and writes it to the target database:

```rust
pub struct PgSync {
    target_pool: Pool<Postgres>,
    receiver: Receiver<TutorData>,
}
```

The core sync logic pattern-matches on the operation type:

```rust
async fn sync_data(&self, data: &TutorData) -> Result<(), sqlx::Error> {
    match data.operation.as_str() {
        "INSERT" => {
            sqlx::query(
                r#"INSERT INTO public.m_tutor_ta
                   (nidn, nama_lengkap, alamat_email, nomor_hp,
                    kode_tutor, nip, institusi,
                    created_at, created_by, updated_at, updated_by)
                VALUES ($1, $2, $3, $4, $5, $6, $7,
                        CAST($8 AS timestamp), $9,
                        CAST($10 AS timestamp), $11)"#
            )
            .bind(&data.nidn)
            .bind(&data.nama)
            // ... remaining bindings
            .execute(&self.target_pool)
            .await?;
        }
        "UPDATE" => {
            sqlx::query(
                r#"UPDATE public.m_tutor_ta
                   SET nidn = COALESCE($1, nidn),
                       nama_lengkap = COALESCE($2, nama_lengkap),
                       -- ... remaining fields
                   WHERE kode_tutor = $11"#
            )
            .bind(&data.nidn)
            .bind(&data.nama)
            // ... remaining bindings
            .execute(&self.target_pool)
            .await?;
        }
        _ => {
            warn!("Unknown operation: {}", data.operation);
        }
    }
    Ok(())
}
```

**Important patterns here:**

1. **`COALESCE` in UPDATE**: This ensures that if a field is `NULL` in the notification payload, we keep the existing value in the target database. This is a defensive strategy that prevents accidental data loss.

2. **`CAST($8 AS timestamp)`**: Timestamps come through as strings in the JSON payload, so we explicitly cast them on the database side.

3. **Operation routing**: The `TG_OP` value from the trigger cleanly maps to the correct SQL operation. This could be extended to handle `DELETE` operations if needed.

---

## Connecting It All Together

The `main.rs` ties everything together using Tokio's `select!` macro:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    dotenv::dotenv().ok();
    log4rs::init_file("log4rs.yaml", Default::default()).unwrap();

    let config = Config::builder()
        .add_source(Environment::default())
        .build()
        .unwrap();

    let app_config: AppConfig = config.try_deserialize().unwrap();

    let source_db = PgPoolOptions::new()
        .max_connections(5)
        .connect(&app_config.source_db_url)
        .await?;

    let target_db = PgPoolOptions::new()
        .max_connections(5)
        .connect(&app_config.target_db_url)
        .await?;

    let (sender, receiver) = broadcast::channel::<TutorData>(100);
    let sender = Arc::new(sender);

    let listener = PgListen::new(source_db, sender.clone());
    listener.init_database_safe().await?;

    let mut syncer = PgSync::new(target_db, receiver);

    tokio::select! {
        _ = listener.start_listening() => info!("Listener stopped"),
        _ = syncer.start_sync() => info!("Syncer stopped"),
    }
    Ok(())
}
```

**Design highlights:**

1. **`broadcast::channel` with capacity 100**: A Tokio broadcast channel decouples the listener from the syncer. The buffer of 100 messages provides backpressure tolerance — if the sync engine is temporarily slow, up to 100 notifications can queue without blocking the listener.

2. **`tokio::select!`**: This runs both the listener and syncer concurrently. If either one completes (or errors), the other is cancelled. This is a clean way to manage two long-running async tasks.

3. **`init_database_safe()`**: Before starting, the application ensures the required trigger and function exist on the source database — with graceful degradation if the user lacks `CREATE` permissions (more on this below).

---

## Configuration and Environment

Configuration is handled through environment variables, making it 12-factor app friendly:

```rust
#[derive(Debug, Deserialize)]
pub struct AppConfig {
    pub source_db_url: String,
    pub target_db_url: String,
}
```

For local development, you create a `.env` file:

```
SOURCE_DB_URL=postgresql://user:password@localhost:5432/source_db
TARGET_DB_URL=postgresql://user:password@localhost:5432/target_db
```

The `config` crate reads from environment variables automatically, and `dotenv` loads the `.env` file. In production (Docker), these are passed directly as environment variables.

---

## Dockerizing the Service

The Dockerfile uses a **multi-stage build** to produce a minimal final image:

```dockerfile
FROM rust:1.83.0-alpine AS build
WORKDIR /usr/src/app

RUN apk add --no-cache musl-dev pkgconfig openssl-dev openssl-libs-static

# Cache dependencies separately from source code
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo fetch
COPY . .
RUN cargo build --release

FROM alpine:3.18 AS final
RUN apk add --no-cache libgcc openssl
WORKDIR /app
COPY --from=build /usr/src/app/target/release/pg-notify-rust /app/pg-notify-rust
RUN chmod +x /app/pg-notify-rust
COPY log4rs.yaml /app/

RUN addgroup -S appgroup && adduser -S appuser -G appgroup && \
    chown -R appuser:appgroup /app/pg-notify-rust
USER appuser

CMD ["/app/pg-notify-rust"]
```

**Optimization techniques used:**

1. **Alpine-based images**: Both the build stage (`rust:alpine`) and the runtime stage (`alpine:3.18`) are minimal, resulting in a final image that's typically under 20MB.

2. **Dependency caching**: The `Cargo.toml` and `Cargo.lock` are copied first with a dummy `main.rs`. This means the dependency download and compilation layer is cached — rebuilds only recompile your source code, not all 50+ dependencies.

3. **Static linking with musl**: Building on Alpine with `musl` produces a statically linked binary that runs anywhere without glibc dependency issues.

4. **Non-root user**: The final image runs as `appuser` for security hardening.

---

## CI/CD with GitLab

The `.gitlab-ci.yml` defines a three-stage pipeline:

```
build_push → deploy_development → deploy_production
```

- **`build_push`**: Builds the Docker image and pushes it to a private registry, tagged with the commit SHA.
- **`deploy_development`**: Automatically deploys to the dev environment via SSH + Docker Swarm service update.
- **`deploy_production`**: Manual trigger deployment to production (requires a human to click the button).

The deployment strategy uses **Docker Swarm service updates**, which provides rolling updates with zero downtime:

```bash
docker service update --with-registry-auth \
    --image $DOCKER_REPO:$CI_COMMIT_SHORT_SHA $SERVICE_NAME
```

---

## Handling Permission Constraints in Production

In real enterprise environments, your application's database user often **doesn't have permission** to create functions or triggers. The `init_database_safe()` method handles this gracefully:

```rust
pub async fn init_database_safe(&self) -> Result<(), sqlx::Error> {
    match self.init_database().await {
        Ok(_) => {
            info!("Successfully created database triggers and functions");
            Ok(())
        }
        Err(e) => {
            if let SqlxError::Database(db_err) = &e {
                if db_err.code().as_deref() == Some("42501") {
                    // Permission denied — check if objects already exist
                    if self.check_function_exists().await? {
                        if self.check_trigger_exists().await? {
                            return Ok(()); // All good!
                        }
                    }
                    // Objects don't exist and we can't create them
                    return Err(/* descriptive error */);
                }
            }
            Err(e)
        }
    }
}
```

The strategy is:

1. **Try to create** the function and trigger (optimistic approach).
2. If we get PostgreSQL error code `42501` (insufficient privilege), **check if they already exist**.
3. If they exist, **continue normally** — a DBA must have pre-created them.
4. If they don't exist, **fail with a clear error message** telling the operator to contact the DBA.

This is a pattern I'd recommend for any application that needs database objects but might run with restricted permissions.

---

## Lessons Learned

### 1. LISTEN/NOTIFY is Transactional — And That's a Feature

Because notifications are only sent when the transaction commits, you never get phantom events from rolled-back transactions. This saved us from many potential consistency issues.

### 2. Keep the Notification Payload Small

PostgreSQL limits `NOTIFY` payloads to 8000 bytes. We only include the fields we need in `json_build_object`. If you're tempted to send entire rows with large text columns, consider sending just the primary key and having the receiver query for the full data.

### 3. Broadcast Channels Handle Backpressure Naturally

Tokio's `broadcast::channel` drops the oldest messages when the buffer is full, and the receiver gets a `Lagged` error. In our case, with a buffer of 100 and near-instant sync operations, we've never hit this in production. But it's good to know the behavior.

### 4. Separate Your Listener Connection

`sqlx::PgListener` uses a dedicated connection, not one from the pool. This is correct — a `LISTEN` connection needs to stay open indefinitely, and you don't want it competing with your pool's regular query connections.

### 5. COALESCE for Defensive Updates

Using `COALESCE` in UPDATE statements prevents null values in the payload from accidentally wiping out existing data. This is especially important when the source trigger might not include all fields in every notification.

### 6. Multi-Stage Docker Builds Are Non-Negotiable for Rust

Rust compile times and toolchain sizes make single-stage Docker images impractical. The multi-stage approach with Alpine gives us a final image that's a fraction of the build image size.

---

## Conclusion

PostgreSQL's `LISTEN/NOTIFY` is one of the most underrated features in the database world. Combined with Rust's async ecosystem (`tokio` + `sqlx`), we built a **real-time data synchronization service** that:

- Uses **~5MB of RAM** in production
- Syncs data in **sub-millisecond** latency after commit
- Has been running for months with **zero downtime**
- Requires **no external message broker** (no Kafka, no RabbitMQ, no Redis)
- Is a **single static binary** deployed in a minimal Docker container

If you're working with PostgreSQL and need to react to data changes in real time — especially with conditional business logic — consider this approach before reaching for heavier tools. Sometimes the simplest solution is the most robust one.

---

*Built with Rust 1.83, SQLx 0.8, Tokio 1.43, and PostgreSQL 15+. *