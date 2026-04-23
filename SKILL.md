---
name: moka
description: Comprehensive guide for building high-performance caching solutions with the Moka Rust library (by moka-rs). Use this skill whenever the user wants to implement in-memory caching in Rust, create concurrent caches, add TTL/TTI expiration to data, build size-aware eviction policies, use async caches with tokio or async-std, implement eviction listeners, use per-entry custom expiry policies, build atomic get-or-initialize patterns, implement cache invalidation strategies, or work with any kind of Rust caching layer. Make sure to use this skill whenever the user mentions moka, Rust cache, concurrent cache, cache eviction, TinyLFU, LRU cache, TTL cache, or wants to speed up their Rust application with caching, even if they don't explicitly ask for the "moka" crate by name.
---

# Moka - Fast Concurrent Cache for Rust

Moka is a fast, concurrent, in-memory cache library for Rust. Inspired by Java's Caffeine, it provides near-optimal hit ratios through the TinyLFU eviction policy, full concurrency for reads and high expected concurrency for writes, and a rich set of features including TTL/TTI expiration, per-entry custom expiry, eviction listeners, size-aware eviction, and both synchronous and asynchronous APIs.

At a high level, Moka works by combining a lock-free concurrent hash table (for key-value storage) with policy structures (for eviction, expiration, and admission) that are updated in batched operations. This design gives strong consistency for reads and eventual consistency for policy metadata, which is the right trade-off for a cache: you never return stale data, but eviction decisions may lag slightly behind the actual state.

The library provides three cache types that cover virtually every caching scenario in Rust applications. The `sync::Cache` and `sync::SegmentedCache` are for synchronous, multi-threaded contexts. The `future::Cache` is for async runtimes like tokio, async-std, and actix-rt. All three share the same builder API, eviction policies, and expiration features, so once you learn one, the others follow naturally.

## Which Cache Type Should You Use?

| Cache Type             | Module         | Feature Flag | When to Use                                                                                                                                                                                             |
| ---------------------- | -------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sync::Cache`          | `moka::sync`   | `sync`       | General-purpose multi-threaded caching. Single lock for policy operations means simpler internals and slightly lower overhead for moderate concurrency.                                                 |
| `sync::SegmentedCache` | `moka::sync`   | `sync`       | High-contention workloads with many threads. Splits the cache into segments to reduce lock contention on policy operations. Choose this when you have many cores hitting the same cache simultaneously. |
| `future::Cache`        | `moka::future` | `future`     | Async applications using tokio, async-std, or actix-rt. Uses async-aware locking so `get_with` can deduplicate concurrent async initializations without blocking the runtime.                           |

For most applications, `sync::Cache` is the right default. Switch to `SegmentedCache` only if you observe lock contention (many threads, high write rate). Switch to `future::Cache` only when you are already in an async context and need async initialization logic.

## Cargo Setup

Moka requires at least one of the `sync` or `future` features. Building without either causes a compile error.

```toml
# Cargo.toml

# For synchronous caches only
[dependencies]
moka = { version = "0.12", features = ["sync"] }

# For async caches only
[dependencies]
moka = { version = "0.12", features = ["future"] }

# For both sync and async caches
[dependencies]
moka = { version = "0.12", features = ["sync", "future"] }

# Optional: enable logging of panics in eviction listeners
moka = { version = "0.12", features = ["sync", "logging"] }
```

The `logging` feature enables `log` crate integration. When enabled, if an eviction listener panics, Moka catches the panic, logs it at `error` level, and permanently disables that listener. Without `logging`, the panic is silently caught and the listener is disabled without any diagnostic output. Enabling this feature is strongly recommended for production use.

## Feature Flags Reference

| Feature                   | Default | Description                                                                                                                                 |
| ------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `sync`                    | No      | Enables `moka::sync::{Cache, SegmentedCache}`. No additional dependencies.                                                                  |
| `future`                  | No      | Enables `moka::future::Cache`. Requires `async-lock`, `event-listener`, `futures-util`.                                                     |
| `logging`                 | No      | Logs eviction listener panics via the `log` crate at `error` level.                                                                         |
| `unstable-debug-counters` | No      | Adds `GlobalDebugCounters::current()` for debugging object construction/destruction. Has performance impact. Intended for development only. |

## Core API Patterns

### 1. Basic Cache Operations (Sync)

The simplest way to get started is with `Cache::new(max_capacity)`. This creates a cache using the TinyLFU eviction policy, which provides near-optimal hit ratios for most workloads.

```rust
use moka::sync::Cache;

fn main() {
    // Create a cache that holds up to 10,000 entries
    let cache: Cache<String, String> = Cache::new(10_000);

    // Insert entries
    cache.insert("key1".to_string(), "value1".to_string());
    cache.insert("key2".to_string(), "value2".to_string());

    // Retrieve entries (borrowed-key lookup)
    if let Some(value) = cache.get("key1") {
        println!("Found: {}", value);
    }

    // Check existence (does NOT update access recency)
    assert!(cache.contains_key("key1"));

    // Remove a single entry
    cache.invalidate("key1");

    // Remove all entries (lazy invalidation — marks a timestamp, entries are
    // actually evicted on subsequent accesses)
    cache.invalidate_all();

    // Get approximate counts
    println!("Entries: {}", cache.entry_count());
}
```

Note the difference between `get` and `contains_key`: `get` updates the entry's last-accessed time (which affects LRU eviction order and TTI expiration), while `contains_key` does not. Use `contains_key` when you only need to check existence and want to avoid influencing eviction decisions.

### 2. Basic Cache Operations (Async)

The async cache has the same API surface but with `.await` on all mutating methods.

```rust
use moka::future::Cache;

#[tokio::main]
async fn main() {
    let cache: Cache<String, String> = Cache::new(10_000);

    cache.insert("key1".to_string(), "value1".to_string()).await;

    if let Some(value) = cache.get("key1").await {
        println!("Found: {}", value);
    }

    cache.invalidate("key1").await;
    cache.invalidate_all();
}
```

### 3. Atomic Get-or-Initialize with `get_with`

This is one of the most powerful patterns in Moka. `get_with` checks if a key exists; if not, it initializes the value using the provided closure and inserts it atomically. In the sync cache, concurrent calls for the same key are deduplicated — only one thread runs the initialization while others wait. In the async cache, the same deduplication happens but without blocking the async runtime.

```rust
use moka::sync::Cache;
use std::thread;

fn expensive_computation(key: &str) -> String {
    println!("Computing for {}", key);
    thread::sleep(std::time::Duration::from_secs(1));
    format!("result_{}", key)
}

fn main() {
    let cache: Cache<String, String> = Cache::new(100);

    // If "key1" is not in cache, expensive_computation is called.
    // If "key1" IS in cache, the closure is never called.
    let value = cache.get_with("key1".to_string(), || expensive_computation("key1"));
    println!("Got: {}", value);
}
```

The async version uses an async block:

```rust
use moka::future::Cache;

#[tokio::main]
async fn main() {
    let cache: Cache<String, String> = Cache::new(100);

    let value = cache.get_with("key1".to_string(), async {
        expensive_async_lookup("key1").await
    }).await;
}
```

### 4. Fallible Initialization with `try_get_with`

When the initialization can fail, use `try_get_with`. Errors are shared across concurrent callers — if one caller's initialization fails, all waiting callers receive the same error wrapped in `Arc<E>`.

```rust
use moka::sync::Cache;
use std::sync::Arc;

fn read_from_db(key: &str) -> Result<String, std::io::Error> {
    // Simulate a DB read that can fail
    Ok(format!("db_value_{}", key))
}

fn main() -> Result<(), Arc<std::io::Error>> {
    let cache: Cache<String, String> = Cache::new(100);

    // Returns Result<String, Arc<std::io::Error>>
    let value = cache.try_get_with("key1".to_string(), || read_from_db("key1"))?;

    // If the init closure panicked, try_get_with propagates it as Arc<E>
    // where E is the panic payload type.
    Ok(())
}
```

The async version works the same way but with `.await`:

```rust
use moka::future::Cache;

#[tokio::main]
async fn main() -> Result<(), Arc<std::io::Error>> {
    let cache: Cache<String, String> = Cache::new(100);
    let value = cache.try_get_with("key1".to_string(), async {
        async_db_read("key1").await
    }).await?;
    Ok(())
}
```

### 5. Conditional Initialization with `get_with_if` (Deprecated — prefer `entry().or_insert_with_if()`)

Use `entry().or_insert_with_if()` when you want to re-initialize an existing entry only if a condition is met. This is useful for implementing refresh-before-expiry patterns.

```rust
use moka::sync::Cache;

fn main() {
    let cache: Cache<String, u64> = Cache::new(100);
    cache.insert("counter".to_string(), 0);

    // Only re-initialize if the current value is 0
    let entry = cache
        .entry("counter".to_string())
        .or_insert_with_if(|| 42, |current_value| *current_value == 0);
    assert_eq!(*entry.value(), 42);

    // Now the value is 42, so the condition is false — returns existing value
    let entry = cache
        .entry("counter".to_string())
        .or_insert_with_if(|| 99, |current_value| *current_value == 0);
    assert_eq!(*entry.value(), 42);
}
```

### 6. Optional Initialization with `optionally_get_with`

When the init closure returns `Option<V>`, use `optionally_get_with`. If the closure returns `None`, the key is not inserted and the method returns `None`.

```rust
use moka::sync::Cache;

fn main() {
    let cache: Cache<String, String> = Cache::new(100);

    // Init returns None, so nothing is inserted
    let value = cache.optionally_get_with("key1".to_string(), || {
        let result: Option<String> = None;
        result
    });
    assert!(value.is_none());

    // Init returns Some, so the value is inserted
    let value = cache.optionally_get_with("key2".to_string(), || {
        Some("hello".to_string())
    });
    assert_eq!(value, Some("hello".to_string()));
}
```

## The Entry API

The Entry API provides fine-grained control over cache insertion. Instead of `get_with` which always initializes on miss, the entry selectors give you access to the existing value (if any) and let you decide what to do.

```rust
use moka::sync::Cache;

fn main() {
    let cache: Cache<String, u32> = Cache::new(100);
    let key = "key1".to_string();

    // Owned key version
    let entry = cache.entry(key.clone()).or_insert(10);
    // entry.is_fresh() == true (key was not in cache)

    let entry = cache.entry(key.clone()).or_insert(20);
    // entry.is_fresh() == false (key already existed)
    // entry.into_value() == 10 (the old value, NOT 20)

    // Borrowed key version (avoids cloning the key)
    let entry = cache.entry_by_ref("key2").or_insert(30);
    // entry.is_fresh() == true

    // Lazy initialization
    let entry = cache.entry_by_ref("key3").or_insert_with(|| expensive_compute());

    // Use default value
    let entry = cache.entry_by_ref("key4").or_default();
}
```

The `Entry` struct provides these inspection methods:

- `key()` — returns a reference to the entry's key
- `value()` — returns a reference to the entry's value
- `into_value()` — consumes the entry and returns the value
- `is_fresh()` — returns `true` if the entry was just created (key was not in the cache)
- `is_old_value_replaced()` — returns `true` if an existing value was replaced by a new one

## Eviction Policies

### TinyLFU (Default)

TinyLFU is the default and recommended eviction policy. It combines an admission policy (LFU — Least Frequently Used) with an eviction policy (LRU — Least Recently Used) to achieve near-optimal hit ratios.

The admission phase uses a Count-Min Sketch to estimate the access frequency of all keys (both cached and non-cached). When a new entry needs to be admitted and the cache is full, the new entry competes against the least-recently-used entry. If the new entry is estimated to be more popular (higher frequency), it is admitted and the LRU entry is evicted. This is what makes TinyLFU powerful: it considers the popularity of ALL observed keys, not just the ones currently in the cache.

```rust
use moka::sync::Cache;
use moka::policy::EvictionPolicy;

let cache = Cache::builder()
    .max_capacity(10_000)
    // TinyLFU is the default, but you can be explicit:
    .eviction_policy(EvictionPolicy::tiny_lfu())
    .build();
```

### LRU

For workloads dominated by recency (e.g., job queues, event streams, time-series data), a simple LRU policy may perform better because TinyLFU's frequency estimation can be misleading when access patterns shift rapidly.

```rust
use moka::sync::Cache;
use moka::policy::EvictionPolicy;

let cache = Cache::builder()
    .max_capacity(10_000)
    .eviction_policy(EvictionPolicy::lru())
    .build();
```

Choose LRU when your access pattern has strong temporal locality and the working set changes frequently. Choose TinyLFU for general-purpose caching where the popular items tend to remain popular over time.

## Size-Aware Eviction

By default, `max_capacity` limits the number of entries. When you configure a `weigher`, `max_capacity` instead limits the total weighted size. Each entry's weight is calculated from its key and value via a closure you provide.

This is essential when cached values have vastly different sizes (e.g., caching HTTP responses that range from 100 bytes to 10 MB).

```rust
use moka::sync::Cache;

fn main() {
    // Cache bounded by total byte size (32 MiB), not entry count
    let cache: Cache<String, Vec<u8>> = Cache::builder()
        .max_capacity(32 * 1024 * 1024) // 32 MiB total
        .weigher(|_key: &String, value: &Vec<u8>| -> u32 {
            value.len().try_into().unwrap_or(u32::MAX)
        })
        .build();

    cache.insert("small".to_string(), vec![0u8; 1024]);      // weight = 1024
    cache.insert("large".to_string(), vec![0u8; 1024 * 1024]); // weight = 1,048,576

    println!("Weighted size: {}", cache.weighted_size());
}
```

The weigher closure takes `(&K, &V)` and returns `u32`. When the total weight exceeds `max_capacity`, entries are evicted in LRU order until the total weight is within capacity. Note that eviction happens in entry-level granularity, not byte-level — an entire entry is evicted at once, even if it would be more efficient to evict a portion of it.

## Expiration

### Time-to-Live (TTL)

TTL limits how long an entry can live from the time it was created or last updated. After the TTL expires, the entry is no longer accessible and will eventually be evicted by the housekeeper.

```rust
use moka::sync::Cache;
use std::time::Duration;

let cache = Cache::builder()
    .max_capacity(10_000)
    .time_to_live(Duration::from_secs(30 * 60)) // 30 minutes
    .build();

cache.insert("session".to_string(), "data".to_string());
// After 30 minutes, cache.get("session") returns None
```

### Time-to-Idle (TTI)

TTI limits how long an entry can remain idle (not accessed) from the time it was last read or written. After the TTI expires, the entry is evicted. This is useful for session caches, user preferences, and any data that should be discarded when no longer actively used.

```rust
use moka::sync::Cache;
use std::time::Duration;

let cache = Cache::builder()
    .max_capacity(10_000)
    .time_to_idle(Duration::from_secs(5 * 60)) // 5 minutes of inactivity
    .build();

cache.insert("user:123".to_string(), "preferences".to_string());
// If nobody accesses "user:123" for 5 minutes, it is evicted
```

TTL and TTI can be combined. When both are set, the entry expires when either condition is met first.

### Per-Entry Custom Expiration via the `Expiry` Trait

For fine-grained control over expiration, implement the `Expiry` trait. This allows different entries to have different expiration times, computed from the key, value, and timestamps.

```rust
use moka::sync::Cache;
use moka::Expiry;
use std::time::{Duration, Instant};

#[derive(Clone)]
struct ApiKeyData {
    ttl: Duration,
    // ... other fields
}

struct ApiKeyExpiry;

impl Expiry<String, ApiKeyData> for ApiKeyExpiry {
    // Called when a new entry is created
    fn expire_after_create(
        &self,
        _key: &String,
        value: &ApiKeyData,
        _created_at: Instant,
    ) -> Option<Duration> {
        Some(value.ttl)
    }

    // Called when an existing entry is read
    fn expire_after_read(
        &self,
        _key: &String,
        _value: &ApiKeyData,
        _read_at: Instant,
        _duration_until_expiry: Option<Duration>,
        _last_modified_at: Instant,
    ) -> Option<Duration> {
        // Don't extend expiration on read — fixed TTL
        None
    }

    // Called when an existing entry is updated
    fn expire_after_update(
        &self,
        _key: &String,
        value: &ApiKeyData,
        _updated_at: Instant,
        _duration_until_expiry: Option<Duration>,
    ) -> Option<Duration> {
        Some(value.ttl)
    }
}

let cache = Cache::builder()
    .max_capacity(100)
    .expire_after(ApiKeyExpiry)
    .build();
```

Each method returns `Option<Duration>`. `Some(duration)` sets the expiration from the current time. `None` means no change to the current expiration. All three methods have default implementations that return `None`, so you only need to override the ones relevant to your use case.

Per-entry expiration is powered by a hierarchical timer wheel data structure (ported from Caffeine), which provides O(1) scheduling and efficient expiration across millions of entries.

## Eviction Listeners

An eviction listener is a callback that fires whenever an entry is removed from the cache. The `RemovalCause` enum tells you why the entry was removed:

| Cause                    | Meaning                                                           |
| ------------------------ | ----------------------------------------------------------------- |
| `RemovalCause::Expired`  | Entry expired (TTL or TTI or custom Expiry)                       |
| `RemovalCause::Explicit` | Entry was explicitly removed via `invalidate` or `invalidate_all` |
| `RemovalCause::Replaced` | Entry was replaced by a new value for the same key                |
| `RemovalCause::Size`     | Entry was evicted to make room for new entries                    |

```rust
use moka::sync::Cache;
use std::time::Duration;

let cache = Cache::builder()
    .max_capacity(100)
    .time_to_live(Duration::from_secs(10))
    .eviction_listener(|key, value: String, cause| {
        match cause {
            moka::notification::RemovalCause::Expired => {
                println!("TTL expired for key {}: {}", key, value);
            }
            moka::notification::RemovalCause::Size => {
                println!("Evicted by size for key {}: {}", key, value);
            }
            _ => {
                println!("Removed key {}: {:?} — {:?}", key, value, cause);
            }
        }
    })
    .build();
```

**Critical**: Eviction listeners must never panic. If a listener panics, Moka catches the panic and permanently disables that listener. Enable the `logging` feature to log the panic at `error` level for diagnostics. Listeners also must not call back into the cache (this can cause deadlocks).

For async caches, use `notification::ListenerFuture` and the `FutureExt` trait:

```rust
use moka::future::{Cache, FutureExt};
use moka::notification::RemovalCause;
use std::sync::Arc;
use std::time::Duration;

async fn async_cleanup(_key: Arc<String>, _value: String, _cause: RemovalCause) {
    // async cleanup logic
}

let cache: Cache<String, String> = Cache::builder()
    .max_capacity(100)
    .time_to_live(Duration::from_secs(10))
    .async_eviction_listener(|key, value: String, cause| {
        // Box the async block for the eviction listener
        async move {
            async_cleanup(key, value, cause).await;
        }
        .boxed()  // Requires FutureExt trait in scope
    })
    .build();
```

## Invalidation

### Single Entry Invalidation

```rust
cache.invalidate(&key);    // Remove by key reference (borrowed)
cache.remove(&key);         // Remove by key reference, returns Option<V>
```

### Bulk Invalidation

```rust
cache.invalidate_all(); // Mark all entries as invalid (lazy, fast)
```

`invalidate_all` does not immediately remove entries. Instead, it records a timestamp. On subsequent `get` calls, entries that were created before this timestamp are treated as invalid. This makes `invalidate_all` extremely fast regardless of cache size, but it means entries are not actually freed until they are accessed or until the housekeeper runs.

### Predicate-Based Invalidation

Predicate-based invalidation lets you remove entries based on custom conditions. This requires enabling `support_invalidation_closures` in the builder.

```rust
use moka::sync::Cache;

let cache = Cache::builder()
    .max_capacity(10_000)
    .support_invalidation_closures()
    .build();

// Store user session data
cache.insert("user:1".to_string(), SessionData { is_active: false });
cache.insert("user:2".to_string(), SessionData { is_active: true });

// Remove all inactive sessions
let _predicate_id = cache.invalidate_entries_if(|_k, v: &SessionData| {
    !v.is_active
}).expect("Invalidation predicate should not conflict");
```

Each call to `invalidate_entries_if` returns a `PredicateId`. Predicates are cleaned up automatically by the housekeeper over time; there is no public `remove_predicates` method. You can let them expire naturally, or force cleanup by calling `run_pending_tasks()`.

## Iteration

Both sync and async caches support iteration. The iterator yields `(Arc<K>, V)` pairs, where the key is wrapped in `Arc` because it is shared with the cache's internal storage.

```rust
use moka::sync::Cache;

let cache: Cache<String, u32> = Cache::new(100);
cache.insert("a".to_string(), 1);
cache.insert("b".to_string(), 2);
cache.insert("c".to_string(), 3);

// Iterate over all entries
for (key, value) in cache.iter() {
    println!("{}: {}", key, value);
}
```

The iterator provides a snapshot-like view but is not guaranteed to be perfectly consistent with concurrent modifications. Entries that are inserted or invalidated during iteration may or may not appear.

## Segmented Cache

The segmented cache splits the internal policy structures into independent segments, reducing lock contention under high concurrency. The API is identical to the regular `Cache`.

```rust
use moka::sync::SegmentedCache;

// Create a segmented cache with 16 segments, holding up to 10,000 entries total
let cache: SegmentedCache<String, String> = SegmentedCache::new(10_000, 16);

// Or use the builder for full configuration
let cache = SegmentedCache::builder(16)
    .max_capacity(10_000)
    .time_to_live(Duration::from_secs(30 * 60))
    .build();

// Same API as Cache
cache.insert("key".to_string(), "value".to_string());
let value = cache.get("key");
```

The number of segments should be a power of two. A good default is the number of CPU cores, or a nearby power of two. Each segment operates independently for policy operations (eviction, expiration), so the effective concurrency scales with the number of segments.

## Custom Hasher

Moka uses `std::collections::hash_map::RandomState` by default. You can specify a custom hasher for better performance or deterministic hashing.

```rust
use moka::sync::Cache;

let cache: Cache<String, String, ahash::RandomState> = Cache::builder()
    .max_capacity(10_000)
    .build_with_hasher(ahash::RandomState::default());

// Or use the dashmap hasher
// build_with_hasher(dashmap::DashMap::default().hasher())
```

## Common Patterns

### Atomic Counter with `get_with`

```rust
use moka::sync::Cache;
use std::sync::Arc;
use std::thread;

fn main() {
    let cache: Cache<String, u64> = Cache::new(100);

    let handles: Vec<_> = (0..4)
        .map(|i| {
            let cache = cache.clone();
            thread::spawn(move || {
                for _ in 0..10_000 {
                    // Atomically increment the counter
                    let value = cache.get_with("counter".to_string(), || 0u64);
                    // Note: This reads then re-inserts, so it is NOT atomic
                    // for increment. Use try_append pattern for true atomicity.
                    drop(value);
                }
            })
        })
        .collect();

    for h in handles {
        h.join().unwrap();
    }
}
```

### Append-to-Cached-Value with `get_with`

The `get_with` pattern naturally supports appending to a cached collection. Since concurrent callers for the same key are deduplicated, only one thread initializes at a time.

```rust
use moka::sync::Cache;
use std::sync::Arc;
use std::thread;

fn main() {
    let cache: Cache<String, Vec<String>> = Cache::new(100);

    let handles: Vec<_> = (0..4)
        .map(|i| {
            let cache = cache.clone();
            thread::spawn(move || {
                let key = format!("log:{}", i);
                let mut entries = cache.get_with(key.clone(), Vec::new);
                entries.push(format!("entry from thread {}", i));
                cache.insert(key, entries); // re-insert to persist
            })
        })
        .collect();

    for h in handles {
        h.join().unwrap();
    }

    for (key, entries) in cache.iter() {
        println!("{}: {:?}", key, entries);
    }
}
```

### Try-Append with `try_get_with`

When appending to a cached value that might not exist, and the initialization itself can fail, use `try_get_with`:

```rust
use moka::sync::Cache;
use std::sync::Arc;

fn main() -> Result<(), Arc<std::io::Error>> {
    let cache: Cache<String, Vec<String>> = Cache::new(100);

    let mut entries = cache.try_get_with("log:1".to_string(), || {
        let mut v = Vec::new();
        v.push("initialized".to_string());
        Ok(v)
    })?;

    entries.push("appended".to_string());
    cache.insert("log:1".to_string(), entries); // re-insert to persist
    Ok(())
}
```

### Refresh-After-Expiry via Custom Expiry

A common pattern is to allow expired entries to serve stale data while a refresh happens in the background. While Moka does not have a built-in `refreshAfterWrite` like Caffeine, you can approximate this with a custom `Expiry` implementation:

```rust
use moka::{sync::Cache, Expiry};
use std::time::{Duration, Instant};

struct RefreshExpiry;

impl Expiry<String, String> for RefreshExpiry {
    fn expire_after_create(
        &self,
        _key: &String,
        _value: &String,
        _created_at: Instant,
    ) -> Option<Duration> {
        Some(Duration::from_secs(60)) // Expire after 60 seconds
    }

    fn expire_after_read(
        &self,
        _key: &String,
        _value: &String,
        _read_at: Instant,
        _duration_until_expiry: Option<Duration>,
        _last_modified_at: Instant,
    ) -> Option<Duration> {
        Some(Duration::from_secs(60)) // Reset TTL on read
    }
}

let cache = Cache::builder()
    .max_capacity(1_000)
    .expire_after(RefreshExpiry)
    .build();
```

## CacheBuilder Quick Reference

All configuration options in one place:

```rust
use moka::sync::Cache;
use moka::policy::EvictionPolicy;
use std::time::Duration;

let cache: Cache<String, String, ahash::RandomState> = Cache::builder()
    // Identity
    .name("my-cache")                              // Optional name for logging/debugging

    // Capacity
    .max_capacity(10_000)                           // Max entries OR max weighted size (with weigher)
    .initial_capacity(1_000)                        // Initial hash table size (reduces early rehashing)

    // Eviction policy
    .eviction_policy(EvictionPolicy::tiny_lfu())    // or EvictionPolicy::lru() (default: tiny_lfu)

    // Size-aware eviction
    .weigher(|_key: &String, value: &Vec<u8>| -> u32 {
        value.len() as u32
    })

    // Expiration
    .time_to_live(Duration::from_secs(30 * 60))    // Max age from create/update
    .time_to_idle(Duration::from_secs(5 * 60))     // Max age from last access
    // .expire_after(MyExpiry)                      // Per-entry custom expiry (overrides TTL/TTI)

    // Callbacks
    .eviction_listener(|_key, _value, _cause| {     // Called on entry removal
        // cleanup logic
    })

    // Invalidation
    .support_invalidation_closures()                // Required for invalidate_entries_if

    // Hasher
    .build_with_hasher(ahash::RandomState::default())  // Custom hasher
    // .build()                                     // Default RandomState
    ;
```

## Key and Value Types

Moka requires that keys implement `Eq + Hash` and both keys and values must be `Clone + Send + Sync + 'static`. These bounds enable the lock-free concurrent hash table and the internal `Arc`-based value storage.

For borrowed-key lookups (e.g., looking up a `String` key with `&str`), the lookup type must implement the `Equivalent<K>` trait. Moka re-exports the `equivalent` crate's `Equivalent` trait for this purpose. The standard library types already implement this (e.g., `&str` is `Equivalent<String>`, `&[u8]` is `Equivalent<Vec<u8>>`).

```rust
use moka::sync::Cache;

let cache: Cache<String, Vec<u8>> = Cache::new(100);

// Insert with owned types
cache.insert("key1".to_string(), vec![1, 2, 3]);

// Look up with borrowed types (no allocation)
if let Some(value) = cache.get("key1") {  // &str -> String via Equivalent
    println!("{:?}", value);
}
```

## Housekeeper and Maintenance

Moka does not spawn background threads. Instead, maintenance tasks (expiration checks, eviction processing) run on the calling thread when certain cache methods are invoked. This design avoids the complexity and overhead of background threads.

Maintenance is triggered by:

- Write methods (`insert`, `get_with`, etc.)
- Some read methods (when the read channel is ready to be drained)
- Explicitly via `run_pending_tasks()`

The housekeeper uses bounded channels. Write operations are recorded and processed in batches when:

- The channel reaches 64 pending recordings, OR
- 300 milliseconds have passed since the last drain

In a long-running application with very low cache activity, expired entries may linger longer than expected. Call `run_pending_tasks()` periodically (e.g., via a timer) if you need stricter expiration enforcement.

```rust
use moka::sync::Cache;

let cache: Cache<String, String> = Cache::new(10_000);

// Force maintenance (expiration, eviction) to run
cache.run_pending_tasks();
```

## Policy Access

The `policy()` method returns a read-only `Policy` struct that exposes cache configuration and statistics:

```rust
use moka::sync::Cache;

let cache = Cache::builder()
    .name("my-cache")
    .max_capacity(10_000)
    .time_to_live(Duration::from_secs(60))
    .build();

let policy = cache.policy();
assert_eq!(policy.max_capacity(), Some(10_000));
assert_eq!(policy.time_to_live(), Some(Duration::from_secs(60)));
assert_eq!(cache.name(), Some("my-cache"));
```

## Common Pitfalls

1. **Forgetting to enable a feature flag** — Building `moka` without `sync` or `future` causes a compile error. Always specify at least one in `Cargo.toml`.

2. **Blocking in async eviction listeners** — In async caches, eviction listeners run on the calling task. If a listener performs blocking I/O, it can stall the async runtime. Use `tokio::task::spawn_blocking` for heavy work.

3. **Panicking in eviction listeners** — A panicking listener is permanently disabled with no recovery. Always wrap listener logic in `std::panic::catch_unwind` if there is any risk, or enable the `logging` feature to at least get diagnostics.

4. **Not accounting for predicate overhead** — Each `invalidate_entries_if` call registers a predicate that the cache evaluates lazily during maintenance. While the housekeeper cleans them up automatically, excessive use without forced maintenance can accumulate predicates. Call `run_pending_tasks()` periodically if you use many predicates.

5. **Expecting immediate invalidation from `invalidate_all`** — `invalidate_all` is lazy. It sets a timestamp, and entries are actually invalidated on subsequent access. Call `run_pending_tasks()` after `invalidate_all()` if you need immediate cleanup.

6. **Using `contains_key` for access-time updates** — `contains_key` does NOT update the entry's last-accessed time. Use `get` if you want to refresh access recency (e.g., to prevent TTI expiration).

7. **Over-weighing with `weigher`** — If the `weigher` closure returns a value larger than `u32::MAX`, it is clamped to `u32::MAX`. For very large values, consider using a coarser unit (e.g., KiB instead of bytes) to avoid losing precision.

## MSRV and Edition

Moka requires Rust 1.71.1 or later and uses the Rust 2021 edition. The project follows a rolling 6-month MSRV policy.
