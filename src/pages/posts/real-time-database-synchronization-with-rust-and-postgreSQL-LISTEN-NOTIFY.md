---
layout: ../../layouts/post.astro
title: "Building Real-Time Database Synchronization with Rust and PostgreSQL LISTEN/NOTIFY"
pubDate: 2025-03-04
description: "How we built a lightweight, production-ready service that synchronizes data between two PostgreSQL databases in real time — using Rust, tokio, sqlx, and PostgreSQL's built-in LISTEN/NOTIFY mechanism. No Kafka, no RabbitMQ, no Redis — just a single static binary and PostgreSQL's native pub/sub."
author: "romando"
isPinned: true
excerpt: "A deep dive into building a real-time data synchronization service between two PostgreSQL databases using Rust, Tokio, SQLx, and PostgreSQL's LISTEN/NOTIFY mechanism. We explore trigger-based conditional notifications, async broadcast channels, multi-stage Docker builds, and graceful permission handling in production environments."
image:
  src: 'realtime-database-synchronization.mage.png'
  alt: 'cover'
tags: ["rust", "postgresql", "database-sync", "tokio", "sqlx", "listen-notify", "docker", "real-time"]
---
![cover](realtime-database-synchronization.mage.png)
# Building Real-Time Database Synchronization with Rust and PostgreSQL LISTEN/NOTIFY

> How I built a lightweight, production-ready service that synchronizes data between two PostgreSQL databases in real time — using Rust, `tokio`, `sqlx`, and PostgreSQL's built-in LISTEN/NOTIFY mechanism. No Kafka, no RabbitMQ, no Redis — just a single static binary and PostgreSQL's native pub/sub.

---

## The Story Behind This Project

A while back, I was working on a side project — a **multi-vendor e-commerce platform** with a service-oriented architecture. The main application had a PostgreSQL database that held all product data: drafts, pending reviews, published listings, archived items — the whole lifecycle. But I also had a **separate public storefront service** with its own database that only needed **published, fully-verified product listings** — products that had passed all quality checks (images uploaded, description approved, pricing validated, category assigned, and stock confirmed).

The naive approach would be to poll the source database every few seconds, but that felt wasteful and introduced unnecessary latency. I could have used a message broker like Kafka or RabbitMQ, but adding that infrastructure for a single sync use case seemed like overkill.

That's when I discovered PostgreSQL's `LISTEN/NOTIFY` — a built-in pub/sub mechanism hiding in plain sight — and decided to build the entire sync layer in **Rust**.

Here's what I needed:

- Synchronize data **in real time** (not batch ETL)
- Only forward records that meet **specific business conditions** (fully verified products only)
- Keep it **lightweight** and resource-efficient
- Handle **permission-restricted** database environments gracefully

---

## Why Not Just Use Logical Replication?

PostgreSQL's built-in logical replication is powerful, but it comes with trade-offs that didn't fit my use case:

| Concern | Logical Replication | My Approach |
|---|---|---|
| **Conditional filtering** | Limited (requires pgoutput plugins or custom logic) | Built into the trigger — only verified products are synced |
| **Schema coupling** | Source and target must have matching schemas | I transform data during sync — schemas can differ |
| **Permission requirements** | Requires `REPLICATION` role and `pg_hba.conf` changes | Works with standard `SELECT`/`INSERT`/`UPDATE` permissions |
| **Operational complexity** | Replication slots, WAL retention, monitoring | A single lightweight binary |
| **Field-level control** | Replicates entire rows | I pick exactly which fields to sync |

For my use case, a trigger-based approach with PostgreSQL's `LISTEN/NOTIFY` was the sweet spot.

---

## Architecture Overview

Here's how the system works at a high level:

```
┌──────────────────────────────────────────────────────────────────┐
│                        pg-sync-service                           │
│                                                                  │
│  ┌─────────────┐    broadcast::channel    ┌─────────────┐        │
│  │  PgListen   │ ──── ProductData ──────> │   PgSync    │        │
│  │  (listener) │                          │  (syncer)   │        │
│  └──────┬──────┘                          └──────┬──────┘        │
│         │                                        │               │
└─────────┼────────────────────────────────────────┼───────────────┘
          │                                        │
          │ LISTEN "data_change_product"           │ INSERT / UPDATE
          │                                        │
  ┌───────▼───────┐                        ┌───────▼───────┐
  │  Source DB    │                        │  Target DB    │
  │  (m_product)  │                        │(m_product_pub)│
  │  + trigger    │                        │               │
  └───────────────┘                        └───────────────┘
```

The flow is straightforward:

1. A **trigger** on the source database's `m_product` table fires on every `INSERT` or `UPDATE`.
2. The trigger checks business conditions (all verification flags must pass) and, if met, sends a **`pg_notify`** with the product's data as a JSON payload.
3. The Rust application's **`PgListen`** module receives the notification in real time.
4. The payload is deserialized into a `ProductData` struct and sent through a **Tokio broadcast channel**.
5. The **`PgSync`** module receives the data and executes the appropriate `INSERT` or `UPDATE` on the target database.

---

## PostgreSQL LISTEN/NOTIFY — A Quick Primer

PostgreSQL has a built-in pub/sub mechanism that most developers overlook:

- **`NOTIFY channel, 'payload'`** — sends a message to a named channel.
- **`LISTEN channel`** — subscribes to messages on that channel.
- **`pg_notify(channel, payload)`** — the function equivalent of `NOTIFY`, usable inside triggers and PL/pgSQL.

Key characteristics:

- **Transactional**: Notifications are only delivered when the transaction that fired them commits. If the transaction rolls back, the notification is discarded. This is huge for data consistency.
- **Lightweight**: No WAL involvement, no replication slots, no disk I/O for the notification itself.
- **Payload limit**: 8000 bytes per notification (more than enough for a JSON record).

This makes it **perfect** for event-driven architectures where you need to react to database changes without polling.

---

## Project Structure

```
pg-sync-service/
├── Cargo.toml              # Dependencies and project metadata
├── Cargo.lock              # Locked dependency versions
├── log4rs.yaml             # Logging configuration
├── sql/
│   ├── source.sql          # Trigger and function for source DB
│   └── target.sql          # Target table schema
└── src/
    ├── main.rs             # Application entry point
    ├── config.rs           # Configuration struct
    ├── model/
    │   ├── mod.rs
    │   └── product.rs      # ProductData struct
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

No framework bloat. Just the essentials.

---

## Deep Dive: The Source Database Trigger

The heart of the system is a PostgreSQL trigger function that fires on the source database:

```sql
CREATE OR REPLACE FUNCTION notify_product_change() RETURNS TRIGGER AS $$
BEGIN
    IF (NEW.status = '1' AND
        NEW.image_verified = '1' AND
        NEW.description_approved = '1' AND
        NEW.pricing_validated = '1' AND
        NEW.stock_confirmed = '1') THEN
        PERFORM pg_notify('data_change_product', json_build_object(
            'operation', TG_OP,
            'product_code', NEW.id_product,
            'name', NEW.product_name,
            'sku', NEW.sku,
            'email_vendor', NEW.vendor_email,
            'phone_vendor', NEW.vendor_phone,
            'category', NEW.category,
            'brand', NEW.brand,
            'is_active', CASE WHEN NEW.status = '1' THEN 1 ELSE 0 END,
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

1. **Conditional notification**: The `IF` block ensures I only notify when *all five* verification flags are set to `'1'`. This means the storefront database only ever receives fully-verified product listings. A product sitting in "draft" or "pending review" status is completely invisible to the downstream system.

2. **`json_build_object`**: I construct a JSON payload right inside the trigger. This gives full control over field naming — notice how `NEW.id_product` becomes `product_code` and `NEW.product_name` becomes `name`. The source and target schemas don't need to match at all.

3. **`TG_OP`**: PostgreSQL's built-in variable that tells us whether this was an `INSERT` or `UPDATE`, which the sync engine uses to decide the correct SQL operation on the target.

4. **`PERFORM pg_notify(...)`**: I use `PERFORM` instead of `SELECT` because I don't need the return value. This is idiomatic PL/pgSQL.

The trigger itself is attached like this:

```sql
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM pg_trigger WHERE tgname = 'product_notify_trigger'
    ) THEN
        CREATE TRIGGER product_notify_trigger
        AFTER INSERT OR UPDATE ON m_product
        FOR EACH ROW EXECUTE FUNCTION notify_product_change();
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
    sender: Arc<Sender<ProductData>>,
}
```

It holds two things: a connection pool to the source database and an `Arc`-wrapped broadcast sender to forward events to the sync engine.

The listening loop is elegant in its simplicity:

```rust
pub async fn start_listening(&self) -> Result<(), sqlx::Error> {
    let mut listener = sqlx::postgres::PgListener::connect_with(&self.pool).await?;
    listener.listen("data_change_product").await?;

    info!("Listening for database changes...");

    while let Ok(notification) = listener.recv().await {
        if let Ok(payload) = serde_json::from_str::<ProductData>(notification.payload()) {
            debug!("{:?}", payload);
            let _ = self.sender.send(payload);
        }
    }
    Ok(())
}
```

**What's happening:**

1. **`PgListener::connect_with`** creates a dedicated PostgreSQL connection for listening. This is separate from the connection pool — important because `LISTEN` requires a persistent connection.
2. **`listener.listen("data_change_product")`** subscribes to the notification channel.
3. The `while let` loop blocks asynchronously until a notification arrives, then deserializes the JSON payload into a `ProductData` struct.
4. The deserialized data is sent through the broadcast channel with `self.sender.send(payload)`.

One critical detail: `sqlx`'s `PgListener` handles **automatic reconnection** under the hood. If the connection drops, it will attempt to reconnect and re-subscribe. This is essential for a long-running service — network blips shouldn't bring down your sync pipeline.

---

## Deep Dive: The Sync Engine

The `PgSync` struct in `src/event/sync.rs` receives product data and writes it to the target database:

```rust
pub struct PgSync {
    target_pool: Pool<Postgres>,
    receiver: Receiver<ProductData>,
}
```

The core sync logic pattern-matches on the operation type:

```rust
async fn sync_data(&self, data: &ProductData) -> Result<(), sqlx::Error> {
    match data.operation.as_str() {
        "INSERT" => {
            sqlx::query(
                r#"INSERT INTO public.m_product_pub
                   (sku, product_name, email_vendor, phone_vendor,
                    product_code, brand, category,
                    created_at, created_by, updated_at, updated_by)
                VALUES ($1, $2, $3, $4, $5, $6, $7,
                        CAST($8 AS timestamp), $9,
                        CAST($10 AS timestamp), $11)"#
            )
            .bind(&data.sku)
            .bind(&data.name)
            // ... remaining bindings
            .execute(&self.target_pool)
            .await?;
        }
        "UPDATE" => {
            sqlx::query(
                r#"UPDATE public.m_product_pub
                   SET sku = COALESCE($1, sku),
                       product_name = COALESCE($2, product_name),
                       email_vendor = COALESCE($3, email_vendor),
                       phone_vendor = COALESCE($4, phone_vendor),
                       created_by = COALESCE($5, created_by),
                       updated_by = COALESCE($6, updated_by),
                       created_at = COALESCE(CAST($7 AS timestamp), created_at),
                       updated_at = COALESCE(CAST($8 AS timestamp), updated_at),
                       brand = COALESCE($9, brand),
                       category = COALESCE($10, category)
                   WHERE product_code = $11"#
            )
            .bind(&data.sku)
            .bind(&data.name)
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

1. **`COALESCE` in UPDATE**: This ensures that if a field is `NULL` in the notification payload, I keep the existing value in the target database. This is a defensive strategy that prevents accidental data loss.

2. **`CAST($8 AS timestamp)`**: Timestamps come through as strings in the JSON payload, so I explicitly cast them on the database side.

3. **Operation routing**: The `TG_OP` value from the trigger cleanly maps to the correct SQL operation. This could easily be extended to handle `DELETE` operations if needed.

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

    let (sender, receiver) = broadcast::channel::<ProductData>(100);
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

For local development, I create a `.env` file:

```
SOURCE_DB_URL=postgresql://user:password@localhost:5432/source_db
TARGET_DB_URL=postgresql://user:password@localhost:5432/target_db
```

The `config` crate reads from environment variables automatically, and `dotenv` loads the `.env` file. In production, these are passed directly as environment variables.

---

## Handling Permission Constraints in Production

In many production environments, your application's database user **doesn't have permission** to create functions or triggers. I ran into this when connecting to a managed PostgreSQL instance. The `init_database_safe()` method handles this gracefully:

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
2. If I get PostgreSQL error code `42501` (insufficient privilege), **check if they already exist**.
3. If they exist, **continue normally** — a DBA must have pre-created them.
4. If they don't exist, **fail with a clear error message** telling the operator what needs to be created.

This is a pattern I'd recommend for any application that needs database objects but might run with restricted permissions. It's self-healing when it can be, and informative when it can't.

---

## Lessons Learned

### 1. LISTEN/NOTIFY is Transactional — And That's a Feature

Because notifications are only sent when the transaction commits, you never get phantom events from rolled-back transactions. This saved me from many potential consistency issues. If a product update is rolled back because of a constraint violation, the storefront database never even knows about it.

### 2. Keep the Notification Payload Small

PostgreSQL limits `NOTIFY` payloads to 8000 bytes. I only include the fields I need in `json_build_object`. If you're tempted to send entire rows with large text columns (like product descriptions with HTML), consider sending just the primary key and having the receiver query for the full data.

### 3. Broadcast Channels Handle Backpressure Naturally

Tokio's `broadcast::channel` drops the oldest messages when the buffer is full, and the receiver gets a `Lagged` error. In my case, with a buffer of 100 and near-instant sync operations, I've never hit this in production. But it's good to know the behavior and plan for it.

### 4. Separate Your Listener Connection

`sqlx::PgListener` uses a dedicated connection, not one from the pool. This is correct — a `LISTEN` connection needs to stay open indefinitely, and you don't want it competing with your pool's regular query connections.

### 5. COALESCE for Defensive Updates

Using `COALESCE` in UPDATE statements prevents null values in the payload from accidentally wiping out existing data. This is especially important when the source trigger might not include all fields in every notification.

---

## When Should You Use This Pattern?

This approach shines when:

- ✅ You need **real-time** sync between two PostgreSQL databases
- ✅ You want **conditional filtering** (only sync rows that meet business rules)
- ✅ You need **schema transformation** between source and target
- ✅ You want to avoid the operational overhead of Kafka/RabbitMQ/Redis
- ✅ You're working with a **small to medium volume** of changes (hundreds per second is fine)

Consider alternatives when:

- ❌ You need to sync to **non-PostgreSQL** targets (LISTEN/NOTIFY is PostgreSQL-specific)
- ❌ You have **very high throughput** needs (millions of events/second)
- ❌ You need **guaranteed delivery with replay** (LISTEN/NOTIFY is fire-and-forget)
- ❌ You need **fan-out to many consumers** (a message broker would be better)

---

## Conclusion

PostgreSQL's `LISTEN/NOTIFY` is one of the most underrated features in the database world. Combined with Rust's async ecosystem (`tokio` + `sqlx`), I built a **real-time data synchronization service** that:

- Uses **~5MB of RAM** in production
- Syncs data in **sub-millisecond** latency after commit
- Has been running for months with **zero downtime**
- Requires **no external message broker** (no Kafka, no RabbitMQ, no Redis)
- Compiles down to a **single static binary**

The entire codebase is under 500 lines of Rust. The memory footprint is negligible. And it just works.

If you're working with PostgreSQL and need to react to data changes in real time — especially with conditional business logic — consider this approach before reaching for heavier tools. Sometimes the simplest solution is the most robust one.

---

*Built with Rust 1.83, SQLx 0.8, Tokio 1.43, and PostgreSQL 15+.*

*If you found this useful, feel free to connect with me or drop a comment. I'm always happy to discuss Rust, PostgreSQL, and systems design.*