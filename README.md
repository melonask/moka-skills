# moka — Rust Concurrent Cache Skill

A comprehensive AI skill for building high-performance caching solutions using the [Moka](https://github.com/moka-rs/moka) Rust library. Moka is a fast, concurrent, in-memory cache library inspired by Java's Caffeine, providing near-optimal hit ratios through the TinyLFU eviction policy.

## Overview

This skill enables an LLM to accurately use and develop solutions based on the Moka caching library. It covers the full API surface of Moka v0.12, including synchronous and asynchronous cache types, eviction policies, expiration strategies, and advanced features — all with concrete Rust code examples.

The skill is structured as a single `SKILL.md` file that serves as a complete reference and coding guide. It follows progressive disclosure principles: the metadata (name + description) is always in context for triggering, and the full instructions load when the skill is invoked.

## What This Skill Covers

### Cache Types

- **`sync::Cache`** — Thread-safe synchronous cache for general-purpose use
- **`sync::SegmentedCache`** — Multi-segment sync cache for high-contention workloads
- **`future::Cache`** — Async-aware cache for tokio, async-std, and actix-rt

### Eviction Policies

- **TinyLFU** (default) — Near-optimal hit ratios combining LFU admission with LRU eviction
- **LRU** — Simple least-recently-used eviction for recency-biased workloads
- **Size-aware eviction** — Bounded by total weighted size via custom `weigher` closures

### Expiration

- **TTL** (time-to-live) — Max age from creation/update
- **TTI** (time-to-idle) — Max age since last access
- **Per-entry custom expiry** — Fine-grained control via the `Expiry` trait (variable TTL per key/value)

### Advanced Features

- **Atomic get-or-initialize** — `get_with`, `try_get_with`, `get_with_if`, `optionally_get_with`
- **Entry API** — Fine-grained insertion control via `entry()` / `entry_by_ref()` selectors
- **Eviction listeners** — Sync and async callbacks on entry removal (`Expired`, `Explicit`, `Replaced`, `Size`)
- **Invalidation** — Single, bulk, and predicate-based (`invalidate_entries_if`)
- **Borrowed-key lookups** — Zero-allocation lookups via the `Equivalent` trait
- **Custom hashers** — Support for `ahash`, `dashmap`, or any `BuildHasher`
- **Housekeeper model** — No background threads; maintenance runs on caller threads

### Developer Guidance

- Cargo feature flag setup (`sync`, `future`, `logging`, `unstable-debug-counters`)
- CacheBuilder configuration reference (all options in one code block)
- Common patterns with production-ready code examples
- Seven documented pitfalls with explanations and mitigations
- MSRV and edition requirements

## Skill File Structure

```
moka/
└── SKILL.md          # Complete skill definition (~500 lines)
    ├── YAML frontmatter (name, description — triggering metadata)
    └── Markdown body (instructions, API reference, code examples)
```

## Triggering

The skill triggers automatically when the user asks about:

- In-memory caching in Rust
- Concurrent or thread-safe caches
- TTL, TTI, or expiration in caching
- Cache eviction policies (TinyLFU, LRU)
- Async caches with tokio or async-std
- The `moka` crate by name
- Any request to speed up a Rust application with caching

## Quick Start Example

```toml
# Cargo.toml
[dependencies]
moka = { version = "0.12", features = ["sync"] }
```

```rust
use moka::sync::Cache;

fn main() {
    let cache: Cache<String, String> = Cache::new(10_000);

    // Atomic get-or-initialize with concurrent deduplication
    let value = cache.get_with("key".to_string(), || {
        expensive_database_lookup()
    });

    println!("Got: {}", value);
}
```

## Requirements

- **Moka crate**: v0.12.x
- **Rust MSRV**: 1.71.1+
- **Rust Edition**: 2021
- At least one of `sync` or `future` feature flags must be enabled

## License

This skill document is provided for educational and development purposes. The Moka library itself is licensed under MIT OR Apache-2.0.
