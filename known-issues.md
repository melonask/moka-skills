# Known Issues in Moka Skill (as of April 2026, moka 0.12.15)

This document catalogues all errors found when compiling code copied directly from `SKILL.md` against the latest `moka` crate (v0.12.15). Each issue includes:

1. The original (broken) snippet from the skill.
2. The compilation error it produces.
3. The corrected, verifiable snippet.

---

## Issue 1: Wrong `Expiry` trait method signatures

**Location:** SKILL.md, "Per-Entry Custom Expiration via the `Expiry` Trait" section.

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

## Issue 2: `cache.get(&"key1")` fails for `String` keys

**Location:** SKILL.md, "Basic Cache Operations (Sync)", "Key and Value Types", and multiple other sections.

**Root Cause:** For borrowed-key lookups, moka uses the `Equivalent<K>` trait which is implemented as `str: Equivalent<String>` (via `String: Borrow<str>`). Passing `&"key1"` gives type `&&str`, but `&str` does **not** implement `Equivalent<String>`. You must pass `"key1"` (type `&str`) directly.

### Broken code (from skill)

```rust
if let Some(value) = cache.get(&"key1".to_string()) {
    println!("Found: {}", value);
}
assert!(cache.contains_key(&"key1".to_string()));
cache.invalidate(&"key1".to_string());
```

### Compilation error

```
error[E0277]: the trait bound `String: Borrow<&str>` is not satisfied
```

### Working fix

```rust
if let Some(value) = cache.get("key1") {
    println!("Found: {}", value);
}
assert!(cache.contains_key("key1"));
cache.invalidate("key1");
```

The same fix applies to `remove`, `contains_key`, and all other methods taking `&Q where Q: Equivalent<K>`.

---

## Issue 3: `cache.entry_by_ref(&"key2")` fails

**Location:** SKILL.md, "The Entry API" section.

**Root Cause:** Same as Issue 2 -- `entry_by_ref` requires `Q: Equivalent<K> + ToOwned<Owned = K> + Hash + ?Sized`. Passing `&"key2"` gives `&&str`, which does not satisfy `ToOwned<Owned = String>`.

### Broken code (from skill)

```rust
let entry = cache.entry_by_ref(&"key2").or_insert(30);
let entry = cache.entry_by_ref(&"key3").or_insert_with(|| expensive_compute());
let entry = cache.entry_by_ref(&"key4").or_default();
```

### Compilation error

```
error[E0271]: type mismatch resolving `<&str as ToOwned>::Owned == String`
error[E0277]: the trait bound `String: Borrow<&str>` is not satisfied
```

### Working fix

```rust
let entry = cache.entry_by_ref("key2").or_insert(30);
let entry = cache.entry_by_ref("key3").or_insert_with(|| expensive_compute());
let entry = cache.entry_by_ref("key4").or_default();
```

---

## Issue 4: `cache.invalidate_all().await` on async cache returns `()`, not a Future

**Location:** SKILL.md, "Basic Cache Operations (Async)".

**Root Cause:** `invalidate_all` on the async (`moka::future::Cache`) cache returns `()` synchronously, not a `Future`. The `.await` in the skill causes a compile error.

### Broken code (from skill)

```rust
cache.invalidate_all().await;
```

### Compilation error

```
error[E0277]: `()` is not a future
cache.invalidate_all().await;
```

### Working fix

```rust
cache.invalidate_all(); // no .await needed
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

## Issue 6: `build::<String, String>()` with explicit generic arguments

**Location:** SKILL.md appears in multiple examples (Eviction Policies, Size-Aware Eviction, TTL, TTI, etc.).

**Root Cause:** `Cache::builder().build()` and `SegmentedCache::builder().build()` take **zero** generic arguments. The type is inferred from the context or by binding the result to an annotated variable. Adding turbofish generics causes a compile error.

### Broken code (from skill)

```rust
let cache = Cache::builder()
    .max_capacity(10_000)
    .eviction_policy(EvictionPolicy::tiny_lfu())
    .build(); // sometimes written as .build::<String, String>();
```

Wait -- `.build()` with no generics actually compiles when type inference can determine the types. The problem arises when the skill writes `.build::<String, String>()`. However, if the result is assigned to a variable with no type annotation and no other context, inference can fail. The robust fix is to either:

1. Annotate the variable: `let cache: Cache<String, String> = Cache::builder()...`
2. Or use no annotation if inference works.

**But note:** In the skill, `build::<String, String>()` IS written with turbofish in several places, causing:

### Compilation error

```
error[E0107]: method takes 0 generic arguments but 2 generic arguments were supplied
```

### Working fix

Remove the turbofish or assign to a typed variable:

```rust
let cache: Cache<String, String> = Cache::builder()
    .max_capacity(10_000)
    .eviction_policy(EvictionPolicy::tiny_lfu())
    .build();
```

---

## Issue 7: `cache.remove_predicates(&[predicate_id])` does not exist in public API

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

## Issue 8: `RemovalCause` does not implement `Display`

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

## Issue 9: `policy.name()` does not exist on `Policy`

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

## Issue 10: Custom hasher type annotation

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

Or let Rust infer it (if the variable is not otherwise constrained):

```rust
let _cache = Cache::builder()
    .max_capacity(10_000)
    .build_with_hasher(ahash::RandomState::default());
```

---

## Issue 11: Missing `#[derive(Clone)]` on value type for custom expiry

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
