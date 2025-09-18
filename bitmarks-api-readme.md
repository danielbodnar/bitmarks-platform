# bitmarks-api

> High-performance WebAssembly-compiled Rust API leveraging Cloudflare's edge infrastructure for distributed bookmark management

## Architecture Overview

The `bitmarks-api` package implements a zero-copy, lock-free concurrent bookmark management system compiled to WebAssembly (WASM) for execution on Cloudflare Workers. The architecture employs a multi-tiered caching strategy with probabilistic data structures for optimal O(log n) query complexity.

### Core Components

#### 1. WASM Runtime Engine
```rust
#[wasm_bindgen]
pub struct ApiRuntime {
    executor: tokio::runtime::Runtime,
    memory_pool: MemPool<4096>,  // 4KB page-aligned allocation
    query_planner: AdaptiveQueryPlanner,
    vector_index: HNSWIndex<768>,  // 768-dimensional embeddings
}
```

**Memory Management**: Custom allocator with TLSF (Two-Level Segregated Fit) algorithm for O(1) allocation/deallocation complexity.

**Concurrency Model**: Lock-free MPMC (Multi-Producer Multi-Consumer) channels using atomic CAS operations for inter-component communication.

#### 2. Storage Abstraction Layer

```rust
trait StorageBackend: Send + Sync {
    type Error: std::error::Error;
    
    async fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>, Self::Error>;
    async fn put(&self, key: &[u8], value: &[u8]) -> Result<(), Self::Error>;
    async fn scan(&self, prefix: &[u8]) -> BoxStream<'_, Result<(Vec<u8>, Vec<u8>), Self::Error>>;
}
```

**Implementation Matrix**:
| Backend | Read Latency (p99) | Write Latency (p99) | Consistency Model |
|---------|-------------------|---------------------|-------------------|
| D1 | 5ms | 10ms | Strong (Serializable) |
| KV | 2ms | 8ms | Eventual (Read-after-write) |
| R2 | 15ms | 25ms | Strong (Linearizable) |

#### 3. Query Optimization Engine

The query planner employs cost-based optimization with cardinality estimation using HyperLogLog++ sketches:

```rust
pub struct QueryPlan {
    stages: Vec<ExecutionStage>,
    cost_estimate: f64,
    cardinality_estimate: u64,
}

impl QueryPlanner {
    pub fn optimize(&self, query: &ParsedQuery) -> QueryPlan {
        let candidates = self.generate_candidate_plans(query);
        candidates.into_iter()
            .map(|plan| (self.estimate_cost(&plan), plan))
            .min_by(|a, b| a.0.partial_cmp(&b.0).unwrap())
            .map(|(_, plan)| plan)
            .unwrap()
    }
}
```

### Vector Search Implementation

#### HNSW (Hierarchical Navigable Small World) Index

Mathematical foundation for similarity search with O(log n) complexity:

```rust
pub struct HNSWIndex<const D: usize> {
    layers: Vec<HashMap<NodeId, Vec<NodeId>>>,
    entry_point: Option<NodeId>,
    M: usize,  // Max connections per layer
    ef_construction: usize,  // Size of dynamic candidate list
    ml: f64,  // Level multiplier (1/ln(2.0))
}

impl<const D: usize> HNSWIndex<D> {
    pub fn search(&self, query: &[f32; D], k: usize, ef: usize) -> Vec<(NodeId, f32)> {
        let mut visited = BitVec::new();
        let mut candidates = BinaryHeap::new();
        let mut w = BinaryHeap::new();
        
        // Greedy search to find nearest neighbors
        for level in (0..=self.top_layer()).rev() {
            w = self.search_layer(query, w, level, ef, &mut visited);
        }
        
        w.into_sorted_vec()
            .into_iter()
            .take(k)
            .map(|node| (node.id, node.distance))
            .collect()
    }
    
    fn distance(&self, a: &[f32; D], b: &[f32; D]) -> f32 {
        // Cosine similarity: 1 - (a·b)/(||a||·||b||)
        let dot: f32 = a.iter().zip(b.iter()).map(|(x, y)| x * y).sum();
        let norm_a: f32 = a.iter().map(|x| x * x).sum::<f32>().sqrt();
        let norm_b: f32 = b.iter().map(|x| x * x).sum::<f32>().sqrt();
        1.0 - (dot / (norm_a * norm_b))
    }
}
```

### CRDT Synchronization Protocol

Implementation of δ-CRDTs (Delta CRDTs) for bandwidth-efficient synchronization:

```rust
pub struct DeltaCRDT<T: CRDTValue> {
    state: T,
    delta_buffer: VecDeque<Delta<T>>,
    version_vector: VersionVector,
    merkle_tree: MerkleTree<Blake3>,
}

impl<T: CRDTValue> DeltaCRDT<T> {
    pub fn merge_delta(&mut self, delta: Delta<T>) -> Result<(), CRDTError> {
        // Verify causality
        if !self.version_vector.dominates(&delta.dependencies) {
            return Err(CRDTError::CausalityViolation);
        }
        
        // Apply delta mutation
        self.state.apply_delta(&delta);
        self.version_vector.increment(delta.replica_id);
        
        // Update Merkle tree for efficient diff detection
        self.merkle_tree.update(delta.hash());
        
        Ok(())
    }
}
```

## Performance Characteristics

### Complexity Analysis

| Operation | Time Complexity | Space Complexity | Amortized Cost |
|-----------|----------------|------------------|----------------|
| Insert | O(log n) | O(1) | O(log n) |
| Search (exact) | O(1)* | O(1) | O(1) |
| Search (semantic) | O(log n) | O(k) | O(k log n) |
| Sync (δ-CRDT) | O(δ) | O(δ) | O(δ) |
| Full-text search | O(m + k) | O(k) | O(m + k) |

*With perfect hash function in KV store

### Memory Layout Optimization

```rust
#[repr(C, align(64))]  // Cache line alignment
pub struct Bookmark {
    // Hot fields (frequently accessed)
    id: [u8; 16],           // UUID as bytes
    url_hash: u64,          // XXH3 hash for fast lookup
    access_count: AtomicU32,
    
    // Warm fields (occasionally accessed)
    title: CompactString,   // Small string optimization
    tags: SmallVec<[u32; 8]>, // Tag IDs, stack-allocated for ≤8 tags
    
    // Cold fields (rarely accessed)
    content: Option<Box<str>>,
    metadata: Option<Box<HashMap<String, Value>>>,
}
```

## API Endpoints

### RESTful Interface Specification

```rust
pub fn configure_routes() -> Router {
    Router::new()
        // Bookmark CRUD
        .route("/bookmarks", get(list_bookmarks).post(create_bookmark))
        .route("/bookmarks/:id", get(get_bookmark)
            .put(update_bookmark)
            .delete(delete_bookmark))
        
        // Search endpoints
        .route("/search", post(hybrid_search))
        .route("/search/semantic", post(semantic_search))
        .route("/search/fulltext", post(fulltext_search))
        
        // Synchronization
        .route("/sync/pull", post(sync_pull))
        .route("/sync/push", post(sync_push))
        .route("/sync/merge", post(sync_merge))
        
        // Analytics
        .route("/analytics/usage", get(usage_analytics))
        .route("/analytics/trends", get(trend_analysis))
}
```

## Deployment Configuration

### Wrangler Configuration

```toml
name = "bitmarks-api"
main = "build/worker/shim.mjs"
compatibility_date = "2024-01-01"

[build]
command = "cargo install -q worker-build && worker-build --release"

[build.upload]
format = "modules"
main = "./build/worker/shim.mjs"

[[build.upload.rules]]
type = "CompiledWasm"
globs = ["build/worker/shim_bg.wasm"]

[env.production]
workers_dev = false
routes = [
    { pattern = "api.bitmarks.io/*", zone_name = "bitmarks.io" }
]

[[d1_databases]]
binding = "DB"
database_name = "bitmarks-prod"
database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

[[kv_namespaces]]
binding = "KV_CACHE"
id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

[[r2_buckets]]
binding = "R2_STORAGE"
bucket_name = "bitmarks-content"

[[vectorize]]
binding = "VECTOR_INDEX"
index_name = "bookmark-embeddings"
```

## Testing Strategy

### Unit Tests
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;
    
    proptest! {
        #[test]
        fn test_crdt_convergence(
            ops1 in prop::collection::vec(arb_operation(), 0..100),
            ops2 in prop::collection::vec(arb_operation(), 0..100)
        ) {
            let mut crdt1 = DeltaCRDT::new();
            let mut crdt2 = DeltaCRDT::new();
            
            // Apply operations in different orders
            for op in &ops1 { crdt1.apply(op); }
            for op in &ops2 { crdt2.apply(op); }
            
            // Sync both ways
            crdt1.merge(&crdt2);
            crdt2.merge(&crdt1);
            
            // Assert convergence
            prop_assert_eq!(crdt1.state, crdt2.state);
        }
    }
}
```

### Load Testing

```bash
# Using grafana/k6 for load testing
k6 run --vus 1000 --duration 30s load-test.js
```

## Monitoring & Observability

### Metrics Collection

```rust
lazy_static! {
    static ref REQUEST_DURATION: Histogram = register_histogram!(
        "api_request_duration_seconds",
        "API request duration in seconds",
        vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
    ).unwrap();
    
    static ref SEARCH_PRECISION: Gauge = register_gauge!(
        "search_precision_ratio",
        "Search result precision (relevant/total)"
    ).unwrap();
}
```

### Distributed Tracing

Integration with OpenTelemetry for distributed tracing:

```rust
#[tracing::instrument(skip(db))]
async fn search_handler(
    Query(params): Query<SearchParams>,
    Extension(db): Extension<Database>,
) -> Result<Json<SearchResponse>, ApiError> {
    let span = tracing::Span::current();
    span.record("query", &params.query);
    
    let results = db.search(&params).await?;
    
    span.record("result_count", results.len());
    span.record("search_latency_ms", elapsed.as_millis());
    
    Ok(Json(results))
}
```

## License

MIT