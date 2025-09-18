# CLAUDE.md - bitmarks-cli

## CLI-Specific Development Guidelines

### Parser Implementation Constraints

The CLI parser employs a deterministic finite automaton (DFA) with O(n) parsing complexity. When extending the command grammar:

1. **Maintain LL(1) Grammar**: Ensure the grammar remains LL(1) parseable to avoid backtracking:
   ```rust
   // CORRECT: Unambiguous grammar
   command := verb object [flags]*
   verb := "add" | "remove" | "search"
   
   // INCORRECT: Requires lookahead > 1
   command := "add" "bookmark" | "add" "tag" "to" "bookmark"
   ```

2. **Command Signature Hashing**: All commands must produce deterministic hashes for caching:
   ```rust
   impl Command {
       fn signature(&self) -> u64 {
           let mut hasher = XxHash64::default();
           hasher.write(&self.verb.discriminant());
           hasher.write(&self.args.canonical_form());
           hasher.finish()
       }
   }
   ```

### State Machine Implementation

The CLI operates as a hierarchical state machine (HSM):

```rust
#[derive(State)]
enum CliState {
    Idle,
    Parsing { buffer: String },
    Executing { command: Command, cancel_token: CancellationToken },
    Syncing { progress: f32 },
    Interactive { context: ReplContext },
}

impl StateMachine for CliStateMachine {
    type State = CliState;
    type Event = CliEvent;
    
    fn transition(&mut self, event: Self::Event) -> Result<Self::State> {
        match (self.state, event) {
            (Idle, Input(s)) => Parsing { buffer: s },
            (Parsing { buffer }, ParseComplete(cmd)) => Executing { 
                command: cmd,
                cancel_token: CancellationToken::new()
            },
            (Executing { .. }, ExecutionComplete(result)) => {
                self.handle_result(result);
                Idle
            }
            // ... other transitions
        }
    }
}
```

### Caching Strategy Implementation

#### Three-Tier Cache Architecture

```rust
pub struct CacheHierarchy {
    l1_memory: Arc<DashMap<CacheKey, CacheEntry>>,     // In-memory, < 100µs
    l2_disk: Arc<Mutex<sled::Db>>,                     // On-disk, < 1ms  
    l3_remote: Arc<CloudflareKV>,                      // Remote, < 50ms
}

impl CacheHierarchy {
    pub async fn get(&self, key: &CacheKey) -> Option<Vec<u8>> {
        // L1: Check memory cache
        if let Some(entry) = self.l1_memory.get(key) {
            if !entry.is_expired() {
                metrics::increment_counter!("cache.l1.hit");
                return Some(entry.data.clone());
            }
        }
        
        // L2: Check disk cache
        if let Ok(Some(data)) = self.l2_disk.lock().await.get(key.as_bytes()) {
            metrics::increment_counter!("cache.l2.hit");
            // Promote to L1
            self.l1_memory.insert(key.clone(), CacheEntry::new(data.to_vec()));
            return Some(data.to_vec());
        }
        
        // L3: Check remote cache
        if let Ok(Some(data)) = self.l3_remote.get(key.to_string()).await {
            metrics::increment_counter!("cache.l3.hit");
            // Promote to L1 and L2
            self.promote(key, &data).await;
            return Some(data);
        }
        
        metrics::increment_counter!("cache.miss");
        None
    }
}
```

### Offline-First Synchronization

#### Operation Log Structure

```rust
#[derive(Serialize, Deserialize)]
pub struct OperationLog {
    operations: Vec<Operation>,
    clock: HybridLogicalClock,
    checkpoint: Option<Checkpoint>,
}

#[derive(Serialize, Deserialize)]
pub struct Operation {
    id: Uuid,
    timestamp: HLCTimestamp,
    op_type: OperationType,
    payload: serde_json::Value,
    device_id: DeviceId,
    dependencies: Vec<Uuid>,  // For causal consistency
}

impl OperationLog {
    pub fn append(&mut self, op: Operation) -> Result<()> {
        // Ensure causal consistency
        for dep_id in &op.dependencies {
            if !self.has_operation(dep_id) {
                return Err(Error::MissingDependency(*dep_id));
            }
        }
        
        // Update HLC
        self.clock.update(op.timestamp);
        
        // Append to log
        self.operations.push(op);
        
        // Trigger compaction if needed
        if self.should_compact() {
            self.compact()?;
        }
        
        Ok(())
    }
    
    fn compact(&mut self) -> Result<()> {
        // Create snapshot at current state
        let snapshot = self.create_snapshot()?;
        
        // Remove operations before snapshot
        self.operations.retain(|op| op.timestamp > snapshot.timestamp);
        
        // Update checkpoint
        self.checkpoint = Some(snapshot);
        
        Ok(())
    }
}
```

### Command Prediction Model

#### Markov Chain Implementation

```rust
pub struct CommandPredictor {
    transition_matrix: HashMap<CommandSignature, HashMap<CommandSignature, f64>>,
    context_window: VecDeque<CommandSignature>,
    window_size: usize,
}

impl CommandPredictor {
    pub fn predict(&self, current: CommandSignature, k: usize) -> Vec<(CommandSignature, f64)> {
        let transitions = self.transition_matrix.get(&current)?;
        
        // Apply context weighting
        let weighted = transitions.iter()
            .map(|(next, base_prob)| {
                let context_boost = self.calculate_context_boost(next);
                let temporal_decay = self.calculate_temporal_decay(next);
                let final_prob = base_prob * context_boost * temporal_decay;
                (*next, final_prob)
            })
            .collect::<Vec<_>>();
        
        // Return top-k predictions
        let mut heap = BinaryHeap::with_capacity(k);
        for (cmd, prob) in weighted {
            heap.push(Reverse((OrderedFloat(prob), cmd)));
            if heap.len() > k {
                heap.pop();
            }
        }
        
        heap.into_sorted_vec()
            .into_iter()
            .map(|Reverse((OrderedFloat(p), c))| (c, p))
            .collect()
    }
}
```

### Output Formatting Pipeline

```typescript
interface FormatterPipeline {
  formatters: Formatter[];
  transform(data: any): string;
}

class OutputPipeline implements FormatterPipeline {
  formatters: Formatter[] = [];
  
  addFormatter(formatter: Formatter): void {
    this.formatters.push(formatter);
  }
  
  transform(data: any): string {
    return this.formatters.reduce((acc, formatter) => {
      return formatter.format(acc);
    }, data);
  }
}

// Formatters
class JsonFormatter implements Formatter {
  format(data: any): string {
    return JSON.stringify(data, null, 2);
  }
}

class TableFormatter implements Formatter {
  format(data: any[]): string {
    const headers = Object.keys(data[0] || {});
    const rows = data.map(item => 
      headers.map(h => String(item[h] ?? ''))
    );
    
    // Calculate column widths
    const widths = headers.map((h, i) => 
      Math.max(h.length, ...rows.map(r => r[i].length))
    );
    
    // Format table
    const separator = '+' + widths.map(w => '-'.repeat(w + 2)).join('+') + '+';
    const headerRow = '|' + headers.map((h, i) => 
      ` ${h.padEnd(widths[i])} `
    ).join('|') + '|';
    
    return [
      separator,
      headerRow,
      separator,
      ...rows.map(row => 
        '|' + row.map((cell, i) => 
          ` ${cell.padEnd(widths[i])} `
        ).join('|') + '|'
      ),
      separator
    ].join('\n');
  }
}
```

### Testing Strategies

#### Integration Test Harness

```rust
#[tokio::test]
async fn test_offline_sync_convergence() {
    // Setup two CLI instances
    let mut cli1 = CliInstance::new_temp().await;
    let mut cli2 = CliInstance::new_temp().await;
    
    // Operate offline
    cli1.set_offline(true);
    cli2.set_offline(true);
    
    // Perform divergent operations
    cli1.execute("add https://example1.com --tags=test").await?;
    cli2.execute("add https://example2.com --tags=demo").await?;
    
    // Verify local state
    assert_eq!(cli1.execute("list --count").await?, "1");
    assert_eq!(cli2.execute("list --count").await?, "1");
    
    // Go online and sync
    cli1.set_offline(false);
    cli2.set_offline(false);
    
    cli1.execute("sync").await?;
    cli2.execute("sync").await?;
    
    // Verify convergence
    let list1 = cli1.execute("list --format=json").await?;
    let list2 = cli2.execute("list --format=json").await?;
    
    assert_eq!(list1, list2);
    assert_eq!(serde_json::from_str::<Vec<Bookmark>>(&list1)?.len(), 2);
}
```

#### Benchmark Suite

```rust
#[bench]
fn bench_search_performance(b: &mut Bencher) {
    let cli = setup_cli_with_bookmarks(10_000);
    
    b.iter(|| {
        black_box(cli.execute("search 'rust async' --limit=100"))
    });
}

#[bench]
fn bench_add_with_caching(b: &mut Bencher) {
    let cli = setup_cli();
    let urls = generate_urls(1000);
    
    b.iter(|| {
        for url in &urls {
            black_box(cli.execute(&format!("add {}", url)));
        }
    });
}
```

### Performance Profiling

```bash
# Generate flame graph
cargo flamegraph --bin bitmarks -- search "test query"

# Memory profiling with Valgrind
valgrind --tool=massif \
         --pages-as-heap=yes \
         --massif-out-file=massif.out \
         target/release/bitmarks search "test"

# CPU profiling with perf
perf record -F 99 -g target/release/bitmarks search "test"
perf report

# Analyze binary size
cargo bloat --release --crates
cargo bloat --release --filter bitmarks
```

### Common Antipatterns to Avoid

1. **Synchronous I/O in async context**
   ```rust
   // BAD: Blocks the executor
   let data = std::fs::read_to_string(path)?;
   
   // GOOD: Use async I/O
   let data = tokio::fs::read_to_string(path).await?;
   ```

2. **Unbounded cache growth**
   ```rust
   // BAD: Memory leak potential
   cache.insert(key, value);
   
   // GOOD: Use bounded cache with eviction
   if cache.len() >= MAX_CACHE_SIZE {
       cache.evict_lru();
   }
   cache.insert(key, value);
   ```

3. **Inefficient string concatenation**
   ```rust
   // BAD: O(n²) complexity
   let mut result = String::new();
   for item in items {
       result = result + &item.to_string();
   }
   
   // GOOD: O(n) with pre-allocation
   let mut result = String::with_capacity(
       items.iter().map(|i| i.len()).sum()
   );
   for item in items {
       result.push_str(&item.to_string());
   }
   ```

### Security Considerations

1. **Command injection prevention**
   ```rust
   // Validate and sanitize all user input
   fn sanitize_command(input: &str) -> Result<String> {
       // Remove shell metacharacters
       let sanitized = input
           .chars()
           .filter(|c| !matches!(c, ';' | '&' | '|' | '>' | '<' | '`' | '$'))
           .collect();
       
       Ok(sanitized)
   }
   ```

2. **Secure credential storage**
   ```rust
   // Use OS keychain for sensitive data
   let keyring = keyring::Entry::new("bitmarks", "api_key");
   keyring.set_password(&api_key)?;
   ```

3. **Rate limiting for API calls**
   ```rust
   let rate_limiter = RateLimiter::new(
       NonZeroU32::new(10).unwrap(),  // 10 requests
       Duration::from_secs(1),         // per second
   );
   ```