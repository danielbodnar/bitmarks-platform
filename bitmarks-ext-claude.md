# CLAUDE.md - bitmarks-ext

## Extension Development Constraints

### Manifest V3 Migration Imperatives

The extension MUST adhere to Manifest V3 constraints, eliminating persistent background pages in favor of event-driven service workers with ephemeral lifecycle:

```typescript
// Service Worker Lifecycle Management
class ServiceWorkerLifecycle {
  private static readonly IDLE_TIMEOUT = 30_000; // 30 seconds
  private static readonly STATE_PERSISTENCE_KEY = 'sw_state';
  
  static async preserveState<T>(state: T): Promise<void> {
    // Serialize state before suspension
    const serialized = msgpack.encode(state);
    await chrome.storage.session.set({
      [this.STATE_PERSISTENCE_KEY]: serialized
    });
  }
  
  static async restoreState<T>(): Promise<T | null> {
    const stored = await chrome.storage.session.get(this.STATE_PERSISTENCE_KEY);
    if (!stored[this.STATE_PERSISTENCE_KEY]) return null;
    
    return msgpack.decode(stored[this.STATE_PERSISTENCE_KEY]);
  }
}

// Usage in service worker
chrome.runtime.onSuspend.addListener(async () => {
  await ServiceWorkerLifecycle.preserveState(globalState);
});

chrome.runtime.onStartup.addListener(async () => {
  globalState = await ServiceWorkerLifecycle.restoreState() || initialState;
});
```

### Content Security Policy Enforcement

Strict CSP compliance with nonce-based script execution:

```typescript
// Dynamic script injection with nonce
class SecureScriptInjector {
  private generateNonce(): string {
    const array = new Uint8Array(16);
    crypto.getRandomValues(array);
    return btoa(String.fromCharCode(...array));
  }
  
  injectScript(code: string): void {
    const nonce = this.generateNonce();
    
    // Update CSP meta tag
    const meta = document.createElement('meta');
    meta.httpEquiv = 'Content-Security-Policy';
    meta.content = `script-src 'nonce-${nonce}' 'strict-dynamic'; object-src 'none';`;
    document.head.appendChild(meta);
    
    // Inject script with nonce
    const script = document.createElement('script');
    script.nonce = nonce;
    script.textContent = code;
    document.head.appendChild(script);
  }
}
```

### Memory Management in Long-Running Contexts

Implement aggressive garbage collection strategies for content scripts:

```typescript
class MemoryManager {
  private weakRefs: WeakRef<any>[] = [];
  private registry: FinalizationRegistry<string>;
  
  constructor() {
    this.registry = new FinalizationRegistry((heldValue) => {
      console.log(`Object ${heldValue} was garbage collected`);
      this.cleanupResources(heldValue);
    });
    
    // Periodic cleanup
    setInterval(() => this.performGarbageCollection(), 60_000);
  }
  
  track<T extends object>(obj: T, identifier: string): T {
    const weakRef = new WeakRef(obj);
    this.weakRefs.push(weakRef);
    this.registry.register(obj, identifier);
    return obj;
  }
  
  performGarbageCollection(): void {
    // Remove dead references
    this.weakRefs = this.weakRefs.filter(ref => ref.deref() !== undefined);
    
    // Force GC if available (Chrome DevTools)
    if (typeof gc === 'function') {
      gc();
    }
  }
}
```

### Cross-Origin Communication Protocol

Implement secure message passing with type safety and validation:

```typescript
// Message protocol with Zod validation
import { z } from 'zod';

const MessageSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('BOOKMARK_CREATE'),
    payload: z.object({
      url: z.string().url(),
      title: z.string().max(200),
      tags: z.array(z.string()).max(20),
      metadata: z.record(z.unknown()).optional()
    })
  }),
  z.object({
    type: z.literal('SYNC_REQUEST'),
    payload: z.object({
      since: z.number().int().positive(),
      deviceId: z.string().uuid()
    })
  })
]);

type Message = z.infer<typeof MessageSchema>;

class SecureMessaging {
  private readonly origin = new URL(chrome.runtime.getURL('')).origin;
  
  async sendMessage(message: Message): Promise<any> {
    // Validate message
    const validated = MessageSchema.parse(message);
    
    // Sign message for integrity
    const signature = await this.signMessage(validated);
    
    return chrome.runtime.sendMessage({
      ...validated,
      signature,
      timestamp: Date.now(),
      nonce: crypto.randomUUID()
    });
  }
  
  private async signMessage(message: any): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(JSON.stringify(message));
    
    const key = await crypto.subtle.generateKey(
      { name: 'HMAC', hash: 'SHA-256' },
      true,
      ['sign', 'verify']
    );
    
    const signature = await crypto.subtle.sign('HMAC', key, data);
    return btoa(String.fromCharCode(...new Uint8Array(signature)));
  }
}
```

### DOM Mutation Optimization

Efficient DOM observation with debouncing and batching:

```typescript
class OptimizedDOMObserver {
  private observer: MutationObserver;
  private mutationQueue: MutationRecord[] = [];
  private rafId: number | null = null;
  private processedNodes = new WeakSet<Node>();
  
  constructor(private callback: (mutations: MutationRecord[]) => void) {
    this.observer = new MutationObserver(this.handleMutations.bind(this));
  }
  
  observe(target: Node, options: MutationObserverInit): void {
    // Use most efficient observation strategy
    this.observer.observe(target, {
      ...options,
      // Minimize observation scope
      attributeOldValue: false,
      characterDataOldValue: false,
      // Use attribute filter when possible
      attributeFilter: options.attributes ? ['href', 'src', 'data-bookmark'] : undefined
    });
  }
  
  private handleMutations(mutations: MutationRecord[]): void {
    // Filter already processed nodes
    const newMutations = mutations.filter(m => {
      if (m.type === 'childList') {
        return Array.from(m.addedNodes).some(n => !this.processedNodes.has(n));
      }
      return !this.processedNodes.has(m.target);
    });
    
    if (newMutations.length === 0) return;
    
    // Mark as processed
    newMutations.forEach(m => {
      if (m.type === 'childList') {
        m.addedNodes.forEach(n => this.processedNodes.add(n));
      } else {
        this.processedNodes.add(m.target);
      }
    });
    
    // Batch mutations
    this.mutationQueue.push(...newMutations);
    
    // Debounce processing
    if (this.rafId === null) {
      this.rafId = requestAnimationFrame(() => {
        this.processBatch();
        this.rafId = null;
      });
    }
  }
  
  private processBatch(): void {
    if (this.mutationQueue.length === 0) return;
    
    const batch = this.mutationQueue.splice(0);
    this.callback(batch);
  }
}
```

### WebAssembly Integration Pattern

Optimal WASM loading and caching strategy:

```typescript
class WasmLoader {
  private static cache = new Map<string, WebAssembly.Module>();
  
  static async loadModule(url: string): Promise<WebAssembly.Instance> {
    // Check cache
    let module = this.cache.get(url);
    
    if (!module) {
      // Stream compilation for optimal performance
      const response = await fetch(url);
      module = await WebAssembly.compileStreaming(response);
      this.cache.set(url, module);
    }
    
    // Instantiate with imports
    const imports = {
      env: {
        memory: new WebAssembly.Memory({ initial: 256, maximum: 512 }),
        abort: (msg: number, file: number, line: number, col: number) => {
          console.error(`WASM abort at ${file}:${line}:${col}`);
        }
      },
      wasi_snapshot_preview1: {
        // WASI imports for Rust stdlib
        environ_get: () => 0,
        environ_sizes_get: () => 0,
        proc_exit: (code: number) => { throw new Error(`Exit: ${code}`); }
      }
    };
    
    return WebAssembly.instantiate(module, imports);
  }
}
```

### State Machine for UI Components

Finite state machine implementation for predictable UI behavior:

```typescript
type PopupState = 
  | { type: 'IDLE' }
  | { type: 'LOADING' }
  | { type: 'BOOKMARKING'; url: string }
  | { type: 'SUCCESS'; bookmarkId: string }
  | { type: 'ERROR'; error: Error };

type PopupEvent =
  | { type: 'BOOKMARK_CLICKED' }
  | { type: 'BOOKMARK_SUCCESS'; bookmarkId: string }
  | { type: 'BOOKMARK_FAILURE'; error: Error }
  | { type: 'RESET' };

class PopupStateMachine {
  private state: PopupState = { type: 'IDLE' };
  
  transition(event: PopupEvent): PopupState {
    switch (this.state.type) {
      case 'IDLE':
        if (event.type === 'BOOKMARK_CLICKED') {
          return { type: 'LOADING' };
        }
        break;
        
      case 'LOADING':
        if (event.type === 'BOOKMARK_SUCCESS') {
          return { type: 'SUCCESS', bookmarkId: event.bookmarkId };
        }
        if (event.type === 'BOOKMARK_FAILURE') {
          return { type: 'ERROR', error: event.error };
        }
        break;
        
      case 'SUCCESS':
      case 'ERROR':
        if (event.type === 'RESET') {
          return { type: 'IDLE' };
        }
        break;
    }
    
    // Invalid transition
    console.warn(`Invalid transition: ${this.state.type} + ${event.type}`);
    return this.state;
  }
}
```

### Performance Monitoring

Real User Monitoring (RUM) implementation:

```typescript
class PerformanceMonitor {
  private metrics: Map<string, number[]> = new Map();
  
  measureAsync<T>(
    name: string,
    fn: () => Promise<T>
  ): Promise<T> {
    const mark = `${name}-${Date.now()}`;
    performance.mark(`${mark}-start`);
    
    return fn().finally(() => {
      performance.mark(`${mark}-end`);
      performance.measure(name, `${mark}-start`, `${mark}-end`);
      
      const measure = performance.getEntriesByName(name, 'measure')[0];
      this.recordMetric(name, measure.duration);
      
      // Clean up marks
      performance.clearMarks(`${mark}-start`);
      performance.clearMarks(`${mark}-end`);
      performance.clearMeasures(name);
    });
  }
  
  private recordMetric(name: string, duration: number): void {
    const metrics = this.metrics.get(name) || [];
    metrics.push(duration);
    
    // Keep only last 100 measurements
    if (metrics.length > 100) {
      metrics.shift();
    }
    
    this.metrics.set(name, metrics);
    
    // Calculate statistics
    if (metrics.length >= 10) {
      const stats = this.calculateStats(metrics);
      console.log(`Performance [${name}]:`, stats);
      
      // Send to analytics if threshold exceeded
      if (stats.p95 > 1000) {
        this.sendToAnalytics(name, stats);
      }
    }
  }
  
  private calculateStats(values: number[]): PerformanceStats {
    const sorted = [...values].sort((a, b) => a - b);
    return {
      mean: values.reduce((a, b) => a + b, 0) / values.length,
      median: sorted[Math.floor(sorted.length / 2)],
      p95: sorted[Math.floor(sorted.length * 0.95)],
      p99: sorted[Math.floor(sorted.length * 0.99)]
    };
  }
}
```

### Testing Harness

Comprehensive testing setup for extension components:

```typescript
// Mock Chrome APIs for testing
class ChromeAPIMock {
  private storage = new Map<string, any>();
  private listeners = new Map<string, Function[]>();
  
  runtime = {
    sendMessage: jest.fn().mockResolvedValue(undefined),
    onMessage: {
      addListener: (fn: Function) => {
        const listeners = this.listeners.get('message') || [];
        listeners.push(fn);
        this.listeners.set('message', listeners);
      }
    }
  };
  
  storage = {
    local: {
      get: jest.fn().mockImplementation((keys) => {
        const result: any = {};
        for (const key of keys) {
          if (this.storage.has(key)) {
            result[key] = this.storage.get(key);
          }
        }
        return Promise.resolve(result);
      }),
      set: jest.fn().mockImplementation((items) => {
        Object.entries(items).forEach(([key, value]) => {
          this.storage.set(key, value);
        });
        return Promise.resolve();
      })
    }
  };
  
  triggerMessage(message: any, sender: any): void {
    const listeners = this.listeners.get('message') || [];
    listeners.forEach(fn => fn(message, sender));
  }
}

// Usage in tests
describe('Extension Message Handler', () => {
  let chromeMock: ChromeAPIMock;
  
  beforeEach(() => {
    chromeMock = new ChromeAPIMock();
    (global as any).chrome = chromeMock;
  });
  
  test('handles bookmark creation', async () => {
    const handler = new MessageHandler();
    
    chromeMock.triggerMessage(
      { type: 'BOOKMARK_CREATE', payload: { url: 'https://example.com' } },
      { tab: { id: 1 } }
    );
    
    await waitFor(() => {
      expect(chromeMock.storage.local.set).toHaveBeenCalledWith(
        expect.objectContaining({
          'bookmark-https://example.com': expect.any(Object)
        })
      );
    });
  });
});
```

### Common Pitfalls

1. **Service Worker termination**: Always use `chrome.storage` for persistence
2. **Content script isolation**: Cannot access page's JavaScript variables directly
3. **CSP violations**: Inline scripts/styles blocked without proper nonce
4. **Memory leaks**: Remove all listeners when cleaning up
5. **Race conditions**: Service worker may restart mid-operation
6. **CORS issues**: Cannot fetch cross-origin resources without permissions