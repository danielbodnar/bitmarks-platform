# CLAUDE.md - bitmarks-api

## Package-Specific Guidelines for Claude Code

### Architectural Invariants

When modifying `bitmarks-api`, maintain these critical invariants:

1. **Zero-Copy Serialization**: All data structures MUST implement `serde::Serialize` with `#[serde(borrow)]` for string fields to avoid allocation overhead.

2. **Memory Alignment**: Structs accessed frequently MUST be cache-line aligned (64 bytes) using `#[repr(C, align(64))]`.

3. **Async Runtime**: ONLY use `tokio` with these specific features:
   ```toml
   tokio = { version = "1", features = ["rt-multi-thread", "macros", "time", "sync"] }
   ```

4. **WASM Compatibility**: Ensure all dependencies are `wasm32-unknown-unknown` compatible. Verify with:
   ```bash
   cargo check --target wasm32-unknown-unknown
   ```

### Code Generation Patterns

#### Request Handler Template
```rust
#[worker::send]
pub async fn handle_${OPERATION}(
    req: HttpRequest,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    // 1. Parse and validate input
    let params = req.query::<${ParamsType}>()?
        .validate()
        .map_err(|e| Error::ValidationError(e))?;
    
    // 2. Acquire resources
    let db = env.d1("DB")?;
    let kv = env.kv("KV_CACHE")?;
    
    // 3. Check cache
    let cache_key = format!("${OPERATION}:{}", params.cache_key());
    if let Some(cached) = kv.get(&cache_key).json::<${ResponseType}>().await? {
        return Response::from_json(&cached)
            .map(|r| r.with_headers(cache_headers()));
    }
    
    // 4. Execute business logic
    let result = ${OPERATION}_logic(&db, &params).await?;
    
    // 5. Update cache
    kv.put(&cache_key, &result)?
        .expiration_ttl(CACHE_TTL)
        .execute()
        .await?;
    
    // 6. Return response
    Response::from_json(&result)
}
```

#### CRDT Operation Template
```rust
impl CRDTOperation for ${OperationType} {
    type State = ${StateType};
    
    fn apply(&self, state: &mut Self::State) -> Result<(), CRDTError> {
        // Validate preconditions
        self.validate_preconditions(state)?;
        
        // Apply operation atomically
        match self {
            ${OperationType}::Insert { key, value, timestamp } => {
                state.lww_map.insert(key.clone(), value.clone(), *timestamp);
            }
            ${OperationType}::Remove { key, timestamp } => {
                state.lww_map.remove(key, *timestamp);
            }
        }
        
        // Update vector clock
        state.vector_clock.increment(self.replica_id());
        
        // Recompute Merkle root
        state.merkle_root = self.compute_merkle_root(state);
        
        Ok(())
    }
    
    fn is_concurrent(&self, other: &Self) -> bool {
        !self.vector_clock.happens_before(&other.vector_clock) &&
        !other.vector_clock.happens_before(&self.vector_clock)
    }
}
```

### Performance Optimization Checklist

When implementing new features, verify:

- [ ] **Allocation Analysis**: Run `cargo bloat --release` to check binary size
- [ ] **Benchmark Suite**: Add microbenchmarks using `criterion`
- [ ] **Flame Graphs**: Generate flame graphs for hot paths
- [ ] **Memory Profiling**: Use `wasm-bindgen` memory profiler
- [ ] **Query Plans**: Analyze with `EXPLAIN QUERY PLAN` for all SQL

### Database Schema Migrations

Always use versioned migrations with rollback support:

```sql
-- Migration: V${VERSION}__${DESCRIPTION}.sql
-- Rollback: V${VERSION}__${DESCRIPTION}.rollback.sql

BEGIN TRANSACTION;

-- Create new structures
CREATE TABLE IF NOT EXISTS bookmarks_v2 (
    -- ... schema
);

-- Migrate data with validation
INSERT INTO bookmarks_v2 
SELECT * FROM bookmarks 
WHERE json_valid(metadata);

-- Atomic swap
ALTER TABLE bookmarks RENAME TO bookmarks_backup;
ALTER TABLE bookmarks_v2 RENAME TO bookmarks;

-- Add indexes AFTER data migration
CREATE INDEX idx_bookmarks_url_hash ON bookmarks(url_hash);

COMMIT;
```

### Vector Search Tuning

Optimal HNSW parameters based on dataset size:

| Dataset Size | M | ef_construction | ef_search | Index Build Time |
|--------------|---|-----------------|-----------|------------------|
| < 10K | 16 | 200 | 50 | < 1s |
| 10K - 100K | 32 | 400 | 100 | < 10s |
| 100K - 1M | 48 | 800 | 200 | < 2min |
| > 1M | 64 | 1600 | 400 | < 15min |

### Error Handling Hierarchy

```rust
#[derive(Error, Debug)]
pub enum ApiError {
    // Client errors (4xx)
    #[error("Invalid request: {0}")]
    BadRequest(String),
    
    #[error("Resource not found: {0}")]
    NotFound(String),
    
    #[error("Rate limit exceeded: {limit} requests per {window}")]
    RateLimited { limit: u32, window: String },
    
    // Server errors (5xx)
    #[error("Database error: {0}")]
    Database(#[from] D1Error),
    
    #[error("Vector index error: {0}")]
    VectorIndex(#[from] VectorizeError),
    
    // Always log these
    #[error("Internal error: {0}")]
    Internal(String),
}

impl ApiError {
    pub fn status_code(&self) -> u16 {
        match self {
            Self::BadRequest(_) => 400,
            Self::NotFound(_) => 404,
            Self::RateLimited { .. } => 429,
            Self::Database(_) | Self::VectorIndex(_) => 503,
            Self::Internal(_) => 500,
        }
    }
}
```

### Testing Strategies

#### Property-Based Testing
```rust
proptest! {
    #[test]
    fn bookmark_url_normalization_is_idempotent(url in url_regex()) {
        let normalized_once = normalize_url(&url);
        let normalized_twice = normalize_url(&normalized_once);
        prop_assert_eq!(normalized_once, normalized_twice);
    }
    
    #[test]
    fn vector_search_preserves_order_invariant(
        embeddings in prop::collection::vec(
            prop::array::uniform32(-1.0f32..1.0),
            10..100
        )
    ) {
        let index = build_index(embeddings);
        let results = index.search(&query, k);
        
        // Verify monotonic distance ordering
        for window in results.windows(2) {
            prop_assert!(window[0].distance <= window[1].distance);
        }
    }
}
```

#### Chaos Engineering Tests
```rust
#[tokio::test]
async fn test_sync_under_network_partition() {
    let mut simulation = NetworkSimulation::new();
    
    // Create partition
    simulation.partition_nodes(&["node1", "node2"], &["node3", "node4"]);
    
    // Perform operations on both sides
    let ops_partition_a = generate_random_ops(100);
    let ops_partition_b = generate_random_ops(100);
    
    simulation.apply_ops("node1", ops_partition_a).await;
    simulation.apply_ops("node3", ops_partition_b).await;
    
    // Heal partition
    simulation.heal_partition();
    
    // Verify eventual consistency
    simulation.wait_for_convergence(Duration::from_secs(10)).await;
    
    let states = simulation.get_all_states().await;
    assert!(states.windows(2).all(|w| w[0] == w[1]));
}
```

### Profiling Commands

```bash
# CPU profiling
perf record -F 99 -g -- cargo run --release
perf report

# Memory profiling
valgrind --tool=massif --massif-out-file=massif.out cargo run --release
ms_print massif.out

# WASM size analysis
wasm-opt -Oz -o optimized.wasm target/wasm32-unknown-unknown/release/bitmarks_api.wasm
wasm-strip optimized.wasm
twiggy top -n 20 optimized.wasm

# Benchmark regression detection
cargo bench -- --save-baseline main
git checkout feature-branch
cargo bench -- --baseline main
```

### CI/CD Pipeline Requirements

```yaml
# .github/workflows/api.yml
- name: Verify WASM compatibility
  run: |
    cargo check --target wasm32-unknown-unknown
    cargo clippy --target wasm32-unknown-unknown -- -D warnings

- name: Size regression check
  run: |
    SIZE=$(stat -f%z target/wasm32-unknown-unknown/release/bitmarks_api.wasm)
    if [ $SIZE -gt 5242880 ]; then # 5MB limit
      echo "WASM binary exceeds size limit: $SIZE bytes"
      exit 1
    fi

- name: Performance regression test
  run: |
    cargo bench -- --save-baseline pr-${{ github.event.pull_request.number }}
    cargo benchcmp main pr-${{ github.event.pull_request.number }} --threshold 5
```

### Security Audit Checklist

- [ ] No `unsafe` blocks without `SAFETY` comments
- [ ] All user inputs validated with `validator` crate
- [ ] SQL injection prevention via parameterized queries
- [ ] XSS prevention in HTML responses
- [ ] CORS headers properly configured
- [ ] Rate limiting implemented per endpoint
- [ ] Authentication tokens have expiration
- [ ] Sensitive data never logged
- [ ] Dependencies audited with `cargo audit`

### Common Pitfalls to Avoid

1. **Never use `block_on` in WASM context** - causes panic
2. **Avoid large stack allocations** - WASM stack limited to 1MB
3. **Don't use `std::thread`** - use `wasm_bindgen_futures::spawn_local`
4. **Cache invalidation must be atomic** - use two-phase commit
5. **Vector dimensions must match** - enforce at compile time with const generics