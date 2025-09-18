# CLAUDE.md - bitmarks-app

## Tauri-Specific Development Constraints

### Memory Safety in IPC Bridge

The Tauri IPC bridge requires careful lifetime management to prevent use-after-free vulnerabilities:

```rust
// INCORRECT: Potential UAF
#[tauri::command]
async fn dangerous_command(state: State<'_, AppState>) -> &str {
    let data = state.get_data();
    &data.as_str() // Lifetime extends beyond function scope
}

// CORRECT: Owned return type
#[tauri::command]
async fn safe_command(state: State<'_, AppState>) -> String {
    state.get_data().to_string()
}

// For zero-copy optimization with large payloads
#[tauri::command]
async fn optimized_command(state: State<'_, Arc<AppState>>) -> Arc<[u8]> {
    Arc::clone(&state.large_data)
}
```

### State Management Pattern

Implement lock-free state management using atomic operations and hazard pointers:

```rust
use crossbeam_epoch::{self as epoch, Atomic, Owned, Shared};

pub struct LockFreeState<T: Send + Sync> {
    head: Atomic<Node<T>>,
}

struct Node<T> {
    data: T,
    next: Atomic<Node<T>>,
}

impl<T: Send + Sync> LockFreeState<T> {
    pub fn update<F>(&self, updater: F) 
    where
        F: Fn(&T) -> T,
    {
        let guard = &epoch::pin();
        
        loop {
            let head = self.head.load(Ordering::Acquire, guard);
            let current_data = unsafe { head.deref() }.data;
            let new_data = updater(&current_data);
            
            let new_node = Owned::new(Node {
                data: new_data,
                next: Atomic::null(),
            });
            
            match self.head.compare_exchange(
                head,
                new_node,
                Ordering::Release,
                Ordering::Acquire,
                guard,
            ) {
                Ok(_) => {
                    unsafe { guard.defer_destroy(head) };
                    break;
                }
                Err(e) => drop(e.new),
            }
        }
    }
}
```

### WebView Thread Synchronization

Ensure proper synchronization between Rust backend and WebView frontend:

```rust
use parking_lot::RwLock;
use dashmap::DashMap;

pub struct EventBus {
    listeners: DashMap<String, Vec<EventHandler>>,
    pending_events: RwLock<VecDeque<Event>>,
}

impl EventBus {
    pub fn emit(&self, event: Event) -> Result<()> {
        // Non-blocking event dispatch
        let mut pending = self.pending_events.write();
        pending.push_back(event.clone());
        
        // Wake up event processor
        self.notify_processor();
        
        // Immediate dispatch for high-priority events
        if event.priority == Priority::Immediate {
            self.dispatch_immediate(event)?;
        }
        
        Ok(())
    }
    
    async fn process_events(&self) {
        loop {
            let events = {
                let mut pending = self.pending_events.write();
                mem::take(&mut *pending)
            };
            
            for event in events {
                if let Some(handlers) = self.listeners.get(&event.name) {
                    // Parallel execution for independent handlers
                    let futures: Vec<_> = handlers
                        .iter()
                        .map(|h| h.handle(event.clone()))
                        .collect();
                    
                    join_all(futures).await;
                }
            }
            
            // Yield to prevent starvation
            yield_now().await;
        }
    }
}
```

### Frontend Performance Patterns

#### Virtual DOM Optimization

```typescript
// Use Vue's shallowRef for large datasets
import { shallowRef, triggerRef, customRef } from 'vue';

function useLargeDataset<T>(initialData: T[]) {
  const data = shallowRef(initialData);
  const version = ref(0);
  
  // Custom ref with batched updates
  const batchedData = customRef((track, trigger) => ({
    get() {
      track();
      return data.value;
    },
    set(newValue: T[]) {
      data.value = newValue;
      
      // Batch DOM updates
      requestIdleCallback(() => {
        version.value++;
        trigger();
      });
    }
  }));
  
  // Incremental update without full re-render
  const updateItem = (index: number, updater: (item: T) => T) => {
    const items = data.value;
    items[index] = updater(items[index]);
    
    // Manually trigger update for specific index
    triggerRef(data);
  };
  
  return { batchedData, updateItem, version };
}
```

#### Web Workers for Heavy Computation

```typescript
// search-worker.ts
import * as Comlink from 'comlink';
import { SearchEngine } from './search-engine';

class SearchWorker {
  private engine: SearchEngine;
  
  constructor() {
    this.engine = new SearchEngine();
  }
  
  async initialize(data: ArrayBuffer) {
    // Load WASM module in worker
    const module = await WebAssembly.instantiate(data);
    this.engine.setWasmModule(module.instance);
  }
  
  async search(query: string, options: SearchOptions): Promise<SearchResult[]> {
    // CPU-intensive search runs off main thread
    return this.engine.search(query, options);
  }
}

Comlink.expose(SearchWorker);

// main.ts
import * as Comlink from 'comlink';

const SearchWorker = Comlink.wrap<typeof SearchWorker>(
  new Worker(new URL('./search-worker.ts', import.meta.url))
);

const worker = await new SearchWorker();
const results = await worker.search('query', { limit: 100 });
```

### Database Access Patterns

#### Connection Pool Management

```rust
use deadpool_sqlite::{Config, Pool, Runtime};
use rusqlite::params;

pub struct DatabasePool {
    pool: Pool,
    write_semaphore: Arc<Semaphore>,
}

impl DatabasePool {
    pub fn new(path: &Path) -> Result<Self> {
        let config = Config::new(path);
        let pool = config.create_pool(Runtime::Tokio1)?;
        
        Ok(Self {
            pool,
            write_semaphore: Arc::new(Semaphore::new(1)), // Single writer
        })
    }
    
    pub async fn read<F, R>(&self, f: F) -> Result<R>
    where
        F: FnOnce(&Connection) -> Result<R> + Send + 'static,
        R: Send + 'static,
    {
        let conn = self.pool.get().await?;
        
        // Run in blocking thread pool
        spawn_blocking(move || {
            conn.pragma_update(None, "query_only", &true)?;
            f(&conn)
        }).await?
    }
    
    pub async fn write<F, R>(&self, f: F) -> Result<R>
    where
        F: FnOnce(&mut Connection) -> Result<R> + Send + 'static,
        R: Send + 'static,
    {
        // Ensure single writer
        let _permit = self.write_semaphore.acquire().await?;
        let mut conn = self.pool.get().await?;
        
        spawn_blocking(move || {
            let tx = conn.transaction()?;
            let result = f(&tx)?;
            tx.commit()?;
            Ok(result)
        }).await?
    }
}
```

#### Query Optimization

```rust
// Prepared statement cache
lazy_static! {
    static ref STATEMENT_CACHE: DashMap<String, String> = {
        let cache = DashMap::new();
        
        // Pre-compile common queries
        cache.insert(
            "search_bookmarks".to_string(),
            r#"
            WITH ranked_results AS (
                SELECT 
                    b.*,
                    bm25(bookmarks_fts) as score,
                    snippet(bookmarks_fts, 0, '<b>', '</b>', '...', 30) as snippet
                FROM bookmarks b
                INNER JOIN bookmarks_fts ON b.id = bookmarks_fts.docid
                WHERE bookmarks_fts MATCH ?1
                ORDER BY score DESC
                LIMIT ?2 OFFSET ?3
            )
            SELECT * FROM ranked_results;
            "#.to_string()
        );
        
        cache
    };
}
```

### Build Optimization

#### Profile-Guided Optimization (PGO)

```bash
# Step 1: Build with instrumentation
RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" \
    cargo build --release

# Step 2: Run representative workload
./target/release/bitmarks-app --benchmark

# Step 3: Build with profile data
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data" \
    cargo build --release

# Step 4: Further optimize with BOLT
perf record -e cycles:u -j any,u -o perf.data -- ./target/release/bitmarks-app
perf2bolt -p perf.data -o perf.fdata ./target/release/bitmarks-app
llvm-bolt ./target/release/bitmarks-app -o bitmarks-app.bolt -data=perf.fdata \
    -reorder-blocks=ext-tsp -reorder-functions=hfsort -split-functions \
    -icf=1 -use-gnu-stack -plt=all
```

#### Binary Size Optimization

```toml
[profile.release]
opt-level = "z"     # Optimize for size
lto = true          # Link-time optimization
codegen-units = 1   # Single codegen unit
strip = true        # Strip symbols
panic = "abort"     # Smaller panic handler

[profile.release.package."*"]
opt-level = "z"     # Optimize all dependencies for size

[dependencies]
# Use features to minimize dependency size
tokio = { version = "1", default-features = false, features = ["rt-multi-thread", "macros"] }
serde = { version = "1", default-features = false, features = ["derive"] }
```

### Security Hardening

#### Content Security Policy

```rust
fn setup_csp(app: &mut App) -> Result<(), Box<dyn Error>> {
    app.setup(|app| {
        let window = app.get_window("main").unwrap();
        
        // Inject CSP meta tag
        window.eval(&format!(r#"
            const meta = document.createElement('meta');
            meta.httpEquiv = 'Content-Security-Policy';
            meta.content = "{}";
            document.head.appendChild(meta);
        "#, CSP_POLICY))?;
        
        Ok(())
    });
    
    Ok(())
}

const CSP_POLICY: &str = r#"
    default-src 'self';
    script-src 'self' 'unsafe-inline' 'unsafe-eval';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    connect-src 'self' https://api.bitmarks.io;
    font-src 'self' data:;
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
"#;
```

### Platform-Specific Optimizations

#### macOS Performance

```rust
#[cfg(target_os = "macos")]
fn optimize_for_macos(window: &Window) {
    unsafe {
        // Enable Metal rendering
        let view = window.ns_view() as id;
        let layer: id = msg_send![view, layer];
        let _: () = msg_send![layer, setContentsScale: 2.0]; // Retina support
        
        // Optimize for ProMotion displays (120Hz)
        let _: () = msg_send![layer, setMaximumDrawableCount: 3];
        let _: () = msg_send![layer, setDisplaySyncEnabled: true];
    }
}
```

#### Windows Performance

```rust
#[cfg(target_os = "windows")]
fn optimize_for_windows(hwnd: HWND) {
    unsafe {
        // Enable GPU acceleration
        SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);
        
        // Optimize for high refresh rate displays
        let mut dxgi_factory: *mut IDXGIFactory1 = null_mut();
        CreateDXGIFactory1(&IDXGIFactory1::uuidof(), &mut dxgi_factory);
        
        // Set frame latency for reduced input lag
        (*dxgi_factory).MakeWindowAssociation(hwnd, DXGI_MWA_NO_WINDOW_CHANGES);
    }
}
```

### Testing Infrastructure

#### Integration Tests

```rust
#[cfg(test)]
mod tests {
    use tauri::test::{mock_builder, MockRuntime};
    
    #[test]
    fn test_command_execution() {
        let app = mock_builder()
            .invoke_handler(tauri::generate_handler![search_bookmarks])
            .build(tauri::generate_context!())
            .expect("Failed to build app");
        
        let window = app.get_window("main").unwrap();
        
        // Test IPC command
        let result: Vec<Bookmark> = tauri::test::get_ipc_response(
            &window,
            tauri::InvokeRequest {
                cmd: "search_bookmarks".into(),
                callback: tauri::CallbackFn(0),
                error: tauri::CallbackFn(1),
                inner: serde_json::json!({
                    "query": "rust"
                }),
            },
        );
        
        assert!(!result.is_empty());
    }
}
```

### Performance Monitoring

```rust
#[tauri::command]
async fn get_performance_metrics() -> PerformanceMetrics {
    PerformanceMetrics {
        memory: {
            let mut system = System::new();
            system.refresh_memory();
            MemoryMetrics {
                used: system.used_memory(),
                total: system.total_memory(),
                available: system.available_memory(),
            }
        },
        cpu: {
            let mut system = System::new();
            system.refresh_cpu();
            CpuMetrics {
                usage: system.global_cpu_info().cpu_usage(),
                cores: system.cpus().len(),
            }
        },
        disk: {
            let stats = fs4::statvfs(".").unwrap();
            DiskMetrics {
                used: stats.blocks() - stats.blocks_available(),
                total: stats.blocks(),
                available: stats.blocks_available(),
            }
        },
    }
}
```