# Known Issues in Moka Skill (as of April 2026, moka 0.12.15)

This document catalogues all errors found when compiling and running code copied directly from `SKILL.md` against the latest `moka` crate (v0.12.15). Each issue includes:

1. The original (broken) snippet from the skill.
2. The compilation or runtime error it produces.
3. The corrected, verifiable snippet.

---

## Issue 1: Wrong `Expiry` trait method signatures

**Location:** SKILL.md, "Per-Entry Custom Expiration via the `Expiry` Trait" and "Refresh-After-Expiry via Custom Expiry" sections.

**Root Cause:** The skill documents outdated `expire_after_read` and `expire_after_update` signatures that were changed in a recent release of moka. The current signatures include `duration_until_expiry: Option<Duration>` (the remaining TTL/TTI at the time of the call) rather than simple time instants.

### Broken code (from skill)

```rust
impl Expiry<String, ApiKeyData> for ApiKeyExpiry {
    fn expire_after_create(
        &self,
        _key: &String,
        value: &ApiKeyData,
        _created_at: Instant,
    ) -> Option<Duration> {
        Some(value.ttl)
    }

    fn expire_after_read(
        &self,
        _key: &String,
        _value: &ApiKeyData,
        last_modified_at: Instant,
        last_accessed_at: Instant,
        current_time: Instant,
    ) -> Option<Duration> {
        None
    }

    fn expire_after_update(
        &self,
        _key: &String,
        value: &ApiKeyData,
        _last_modified_at: Instant,
        _current_time: Instant,
    ) -> Option<Duration> {
        Some(value.ttl)
    }
}
```

### Compilation error

```
error[E0053]: method `expire_after_read` has an incompatible type for trait
expected signature `fn(&ApiKeyExpiry, &String, &ApiKeyData, Instant, Option<Duration>, Instant) -> Option<_>`
found signature `fn(&ApiKeyExpiry, &String, &ApiKeyData, Instant, Instant, Instant) -> Option<_>`

error[E0053]: method `expire_after_update` has an incompatible type for trait
expected signature `fn(&ApiKeyExpiry, &String, &ApiKeyData, Instant, Option<Duration>) -> Option<_>`
found signature `fn(&ApiKeyExpiry, &String, &ApiKeyData, Instant, Instant) -> Option<_>`
```

### Working fix

```rust
use moka::Expiry;
use std::time::{Duration, Instant};

#[derive(Clone)]
struct ApiKeyData {
    ttl: Duration,
}

struct ApiKeyExpiry;

impl Expiry<String, ApiKeyData> for ApiKeyExpiry {
    fn expire_after_create(
        &self,
        _key: &String,
        value: &ApiKeyData,
        _created_at: Instant,
    ) -> Option<Duration> {
        Some(value.ttl)
    }

    fn expire_after_read(
        &self,
        _key: &String,
        _value: &ApiKeyData,
        _read_at: Instant,
        _duration_until_expiry: Option<Duration>,
        _last_modified_at: Instant,
    ) -> Option<Duration> {
        // Don't extend expiration on read -- fixed TTL
        None
    }

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
```

---

## Issue 2: `get_with_if` is deprecated

**Location:** SKILL.md, "Conditional Initialization with `get_with_if`" section.

**Root Cause:** `Cache::get_with_if` was deprecated in moka 0.12.15 and replaced with `entry().or_insert_with_if()`. The old method still compiles but emits a deprecation warning.

### Broken code (from skill)

```rust
let value = cache.get_with_if(
    "counter".to_string(),
    || 42,
    |current_value| *current_value == 0,
);
```

### Compilation warning

```
warning: use of deprecated method `moka::sync::Cache::<K, V, S>::get_with_if`: Replaced with `entry().or_insert_with_if()`
```

### Working fix

```rust
let entry = cache
    .entry("counter".to_string())
    .or_insert_with_if(|| 42, |current_value| *current_value == 0);
assert_eq!(*entry.value(), 42);
```

---

## Issue 3: Append-to-cached-value patterns silently drop mutations

**Location:** SKILL.md, "Append-to-Cached-Value with `get_with`" and "Try-Append with `try_get_with`" sections.

**Root Cause:** `get_with` and `try_get_with` return a **clone** of the cached value. Calling `.push()` on that clone mutates the temporary copy, not the value stored in the cache. The skill's example code silently fails to persist the appended data.

### Broken code (from skill)

```rust
cache.get_with(key.clone(), Vec::new).push(format!("entry from thread {}", i));
```

### Runtime behavior

`cache.get(key).unwrap().len()` is `0` because the `push` was applied to a temporary clone.

### Working fix

Use `Arc<Mutex<T>>` for shared mutable state, or re-insert after mutation:

```rust
use moka::sync::Cache;
use std::sync::{Arc, Mutex};

let cache: Cache<String, Arc<Mutex<Vec<String>>>> = Cache::new(100);
let vec = cache.get_with(key.clone(), || Arc::new(Mutex::new(Vec::new())));
vec.lock().unwrap().push(format!("entry from thread {}", i));
```

Or re-insert:

```rust
let mut entries = cache.try_get_with("log:1".to_string(), || {
    let mut v = Vec::new();
    v.push("initialized".to_string());
    Ok::<_, std::io::Error>(v)
})?;
entries.push("appended".to_string());
cache.insert("log:1".to_string(), entries); // re-insert to persist the mutation
```

---

## Issue 4: `RemovalCause` does not implement `Display`

**Location:** SKILL.md, "Eviction Listeners" sync example.

**Root Cause:** `RemovalCause` only implements `Debug`, not `Display`. Using `{}` format specifier in `println!` fails.

### Broken code (from skill)

```rust
println!("Removed key {}: {:?} — {}", key, value, cause);
```

### Compilation error

```
error[E0277]: `RemovalCause` doesn't implement `std::fmt::Display`
```

### Working fix

Use `{:?}` instead of `{}`:

```rust
println!("Removed key {}: {:?} — {:?}", key, value, cause);
```

---

## Issue 5: Async eviction listener API mismatch

**Location:** SKILL.md, "Eviction Listeners" async example.

**Root Cause:** The async cache builder uses `async_eviction_listener`, not `eviction_listener`. The skill mixes the sync closure style with the async `FutureExt::boxed()` pattern, which does not compile because `eviction_listener` on the future builder expects a _sync_ closure and wraps it internally.

### Broken code (from skill)

```rust
use moka::future::{Cache, FutureExt};

let cache = Cache::builder()
    .max_capacity(100)
    .time_to_live(Duration::from_secs(10))
    .eviction_listener(|key, value: String, cause| {
        async move {
            async_cleanup(key, value, cause).await;
        }
        .boxed()  // Requires FutureExt trait in scope
    })
    .build();
```

### Compilation error

```
error[E0308]: mismatched types
expected `()`, found `Pin<Box<dyn Future<Output = ()> + Send>>`
```

### Working fix

```rust
use moka::future::{Cache, FutureExt};
use moka::notification::RemovalCause;
use std::sync::Arc;

async fn async_cleanup(_key: Arc<String>, _value: String, _cause: RemovalCause) {
    // async cleanup logic
}

let cache: Cache<String, String> = Cache::builder()
    .max_capacity(100)
    .time_to_live(Duration::from_secs(1))
    .async_eviction_listener(|key, value: String, cause| {
        async move {
            async_cleanup(key, value, cause).await;
        }
        .boxed()
    })
    .build();
```

---

## Issue 6: `policy.name()` does not exist on `Policy`

**Location:** SKILL.md, "Policy Access" section.

**Root Cause:** The `Policy` struct does not have a `name()` method. The cache name is accessible via `cache.name() -> Option<&str>` on the `Cache` struct itself.

### Broken code (from skill)

```rust
let policy = cache.policy();
assert_eq!(policy.name(), Some("my-cache"));
```

### Compilation error

```
error[E0599]: no method named `name` found for struct `Policy` in the current scope
```

### Working fix

```rust
let policy = cache.policy();
assert_eq!(cache.name(), Some("my-cache"));
```

---

## Issue 7: `build_with_hasher` requires type annotation with the hasher type

**Location:** SKILL.md, "Custom Hasher" section.

**Root Cause:** `build_with_hasher` returns `Cache<K, V, S>` where `S` is the hasher type. If the cache variable is annotated as `Cache<String, String>` (which defaults the hasher to `RandomState`), passing an `ahash::RandomState` causes a type mismatch.

### Broken code (from skill)

```rust
let cache = Cache::builder()
    .max_capacity(10_000)
    .build_with_hasher(ahash::RandomState::default());
```

### Compilation error

```
error[E0308]: mismatched types
expected `std::hash::RandomState`, found `ahash::RandomState`
```

### Working fix

Include the hasher type in the cache type:

```rust
let cache: Cache<String, String, ahash::RandomState> = Cache::builder()
    .max_capacity(10_000)
    .build_with_hasher(ahash::RandomState::default());
```

---

## Issue 8: `cache.remove_predicates(&[predicate_id])` does not exist in public API

**Location:** SKILL.md, "Predicate-Based Invalidation" section.

**Root Cause:** Moka's public API does **not** expose a `remove_predicates` method. Looking at the source, `remove_predicates` is a private helper inside the invalidator module; predicates are cleaned up automatically by the housekeeper. The skill incorrectly instructs users to call `cache.remove_predicates(&[predicate_id])`.

### Broken code (from skill)

```rust
let predicate_id = cache.invalidate_entries_if(|k, v: &SessionData| {
    !v.is_active
}).expect("Invalidation predicate should not conflict");

// Clean up the predicate when no longer needed (prevents memory leak)
cache.remove_predicates(&[predicate_id]);
```

### Compilation error

```
error[E0599]: no method named `remove_predicates` found for struct `moka::sync::Cache<K, V, S>` in the current scope
```

### Working fix

Remove the `remove_predicates` call. Just use `invalidate_entries_if`:

```rust
let _predicate_id = cache
    .invalidate_entries_if(|_k, v: &SessionData| !v.is_active)
    .expect("Invalidation predicate should not conflict");
```

---

## Issue 9: Missing `#[derive(Clone)]` on value type for custom expiry

**Location:** SKILL.md, "Per-Entry Custom Expiration via the `Expiry` Trait".

**Root Cause:** Moka requires `V: Clone + Send + Sync + 'static`. The skill's `ApiKeyData` struct is missing `Clone`.

### Broken code (from skill)

```rust
struct ApiKeyData {
    ttl: Duration,
}
```

### Compilation error

```
error[E0277]: the trait bound `ApiKeyData: Clone` is not satisfied
```

### Working fix

```rust
#[derive(Clone)]
struct ApiKeyData {
    ttl: Duration,
}
```

---

## Issue 10: `invalidate` / `remove` comment about borrowed keys is misleading

**Location:** SKILL.md, "Invalidation" → "Single Entry Invalidation".

**Root Cause:** The skill shows `cache.invalidate(&key);` and `cache.remove(&key);` with the comment "Remove by key reference (borrowed)". This compiles when `key` is an owned `String`, but the comment is misleading because `&String` is not the typical "borrowed key" pattern moka users expect (which is `&str`). The standard borrowed-key lookup is `cache.invalidate("key")` where `"key"` is `&str`.

### Working fix

Use `&str` for borrowed-key lookups, which is moka's idiomatic pattern:

```rust
cache.invalidate("key");         // &str -> String via Equivalent
cache.remove("key");           // &str -> String via Equivalent, returns Option<V>
```

---

## Verification Summary

All other snippets in SKILL.md compile successfully against moka 0.12.15. Verified April 2026 by building a full test harness in `moka-test/` covering every code block in the skill.
