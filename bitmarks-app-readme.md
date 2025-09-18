# bitmarks-app

> Native desktop application leveraging Tauri's multi-threaded Rust backend with hardware-accelerated WebView rendering for sub-millisecond UI response times

## Technical Architecture

The desktop application implements a dual-process architecture with IPC-based command pattern, utilizing Tauri's async runtime for non-blocking I/O operations and native OS integration.

### System Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Main Process (Rust)                 │
├──────────────────────┬──────────────────────────────┤
│   Tauri Runtime      │     Application Core         │
│   - Window Manager   │     - State Machine          │
│   - IPC Handler      │     - Database Engine        │
│   - Asset Pipeline   │     - Sync Controller        │
├──────────────────────┴──────────────────────────────┤
│              Native OS Integration Layer            │
├──────────────────────────────────────────────────────┤
│            WebView Process (Chromium/WebKit)        │
├──────────────────────────────────────────────────────┤
│              Frontend (Vue 3 + TypeScript)          │
└──────────────────────────────────────────────────────┘
```

### Memory-Mapped Database Implementation

```rust
use memmap2::MmapOptions;
use zerocopy::{AsBytes, FromBytes};

#[repr(C)]
#[derive(AsBytes, FromBytes, Copy, Clone)]
pub struct BookmarkRecord {
    pub id: [u8; 16],
    pub url_hash: u64,
    pub title: [u8; 256],
    pub tags: [u32; 16],  // Tag IDs
    pub created_at: i64,
    pub updated_at: i64,
    pub access_count: u32,
    pub flags: u32,
}

pub struct MmapDatabase {
    mmap: memmap2::MmapMut,
    header: *mut DatabaseHeader,
    records: *mut BookmarkRecord,
    capacity: usize,
}

unsafe impl Send for MmapDatabase {}
unsafe impl Sync for MmapDatabase {}

impl MmapDatabase {
    pub fn open(path: &Path) -> Result<Self> {
        let file = OpenOptions::new()
            .read(true)
            .write(true)
            .create(true)
            .open(path)?;
        
        // Allocate 1GB initially
        file.set_len(1 << 30)?;
        
        let mut mmap = unsafe { MmapOptions::new().map_mut(&file)? };
        
        let header = mmap.as_mut_ptr() as *mut DatabaseHeader;
        let records = unsafe {
            mmap.as_mut_ptr().add(size_of::<DatabaseHeader>())
        } as *mut BookmarkRecord;
        
        Ok(Self {
            mmap,
            header,
            records,
            capacity: (1 << 30 - size_of::<DatabaseHeader>()) / size_of::<BookmarkRecord>(),
        })
    }
    
    pub fn insert(&mut self, record: BookmarkRecord) -> Result<()> {
        unsafe {
            let header = &mut *self.header;
            if header.count >= self.capacity {
                return Err(Error::CapacityExceeded);
            }
            
            let index = header.count as usize;
            self.records.add(index).write(record);
            header.count += 1;
            
            // Ensure write visibility
            self.mmap.flush_async()?;
        }
        
        Ok(())
    }
    
    pub fn binary_search(&self, url_hash: u64) -> Option<&BookmarkRecord> {
        unsafe {
            let header = &*self.header;
            let slice = slice::from_raw_parts(self.records, header.count as usize);
            
            slice.binary_search_by_key(&url_hash, |r| r.url_hash)
                .ok()
                .map(|idx| &slice[idx])
        }
    }
}
```

### IPC Command Architecture

```rust
#[tauri::command]
async fn execute_search(
    state: State<'_, AppState>,
    query: SearchQuery,
) -> Result<Vec<SearchResult>, CommandError> {
    // Acquire read lock with timeout
    let db = state.db.read_timeout(Duration::from_millis(100))
        .map_err(|_| CommandError::LockTimeout)?;
    
    // Spawn blocking task for CPU-intensive search
    let results = spawn_blocking(move || {
        let searcher = TantivySearcher::new(&db);
        searcher.search(&query)
    }).await?;
    
    Ok(results)
}

#[derive(Clone, serde::Serialize)]
struct SearchProgress {
    current: usize,
    total: usize,
    phase: SearchPhase,
}

#[derive(Clone, serde::Serialize)]
enum SearchPhase {
    Indexing,
    Querying,
    Ranking,
    Filtering,
}

#[tauri::command]
async fn search_with_progress(
    window: Window,
    state: State<'_, AppState>,
    query: SearchQuery,
) -> Result<Vec<SearchResult>, CommandError> {
    let (tx, mut rx) = mpsc::channel::<SearchProgress>(100);
    
    // Spawn background search task
    let search_handle = spawn(async move {
        perform_search_with_progress(query, tx).await
    });
    
    // Forward progress updates to frontend
    spawn(async move {
        while let Some(progress) = rx.recv().await {
            window.emit("search-progress", progress).ok();
        }
    });
    
    search_handle.await?
}
```

## Frontend Implementation

### Vue 3 Composition API with TypeScript

```typescript
// stores/bookmarks.ts
import { defineStore } from 'pinia';
import { ref, computed, shallowRef } from 'vue';
import { invoke } from '@tauri-apps/api/tauri';

interface Bookmark {
  id: string;
  url: string;
  title: string;
  tags: string[];
  embedding?: Float32Array;
}

export const useBookmarksStore = defineStore('bookmarks', () => {
  // Use shallowRef for large arrays to avoid deep reactivity overhead
  const bookmarks = shallowRef<Bookmark[]>([]);
  const searchResults = shallowRef<SearchResult[]>([]);
  const isSearching = ref(false);
  
  // Virtual list for performance
  const virtualList = computed(() => {
    const PAGE_SIZE = 50;
    return {
      total: bookmarks.value.length,
      pages: Math.ceil(bookmarks.value.length / PAGE_SIZE),
      getPage: (page: number) => {
        const start = page * PAGE_SIZE;
        return bookmarks.value.slice(start, start + PAGE_SIZE);
      }
    };
  });
  
  // Debounced search with cancellation
  const searchController = ref<AbortController | null>(null);
  
  const search = useDebounceFn(async (query: string) => {
    // Cancel previous search
    searchController.value?.abort();
    searchController.value = new AbortController();
    
    isSearching.value = true;
    
    try {
      const results = await invoke<SearchResult[]>('execute_search', {
        query,
        signal: searchController.value.signal
      });
      
      searchResults.value = results;
    } catch (error) {
      if (error.name !== 'AbortError') {
        throw error;
      }
    } finally {
      isSearching.value = false;
    }
  }, 300);
  
  return {
    bookmarks,
    searchResults,
    isSearching,
    virtualList,
    search
  };
});
```

### Hardware Acceleration

```typescript
// GPU-accelerated rendering for large datasets
class GPURenderer {
  private gl: WebGL2RenderingContext;
  private program: WebGLProgram;
  private vao: WebGLVertexArrayObject;
  
  constructor(canvas: HTMLCanvasElement) {
    const gl = canvas.getContext('webgl2', {
      antialias: false,
      depth: false,
      preserveDrawingBuffer: false,
      powerPreference: 'high-performance'
    });
    
    if (!gl) throw new Error('WebGL2 not supported');
    
    this.gl = gl;
    this.program = this.createProgram();
    this.vao = this.createVAO();
  }
  
  private createProgram(): WebGLProgram {
    const vertexShader = `#version 300 es
      in vec2 a_position;
      in vec4 a_color;
      
      out vec4 v_color;
      
      uniform mat3 u_transform;
      
      void main() {
        vec3 transformed = u_transform * vec3(a_position, 1.0);
        gl_Position = vec4(transformed.xy, 0.0, 1.0);
        v_color = a_color;
      }
    `;
    
    const fragmentShader = `#version 300 es
      precision highp float;
      
      in vec4 v_color;
      out vec4 fragColor;
      
      void main() {
        fragColor = v_color;
      }
    `;
    
    return this.compileProgram(vertexShader, fragmentShader);
  }
  
  renderBookmarks(bookmarks: Bookmark[]): void {
    const { gl } = this;
    
    // Convert bookmarks to vertex data
    const vertices = new Float32Array(bookmarks.length * 6); // x, y, r, g, b, a
    
    bookmarks.forEach((bookmark, i) => {
      const offset = i * 6;
      // Position based on embedding
      vertices[offset] = bookmark.embedding?.[0] ?? 0;
      vertices[offset + 1] = bookmark.embedding?.[1] ?? 0;
      // Color based on tags
      vertices[offset + 2] = this.hashToColor(bookmark.tags[0])[0];
      vertices[offset + 3] = this.hashToColor(bookmark.tags[0])[1];
      vertices[offset + 4] = this.hashToColor(bookmark.tags[0])[2];
      vertices[offset + 5] = 1.0;
    });
    
    // Upload to GPU
    gl.bindVertexArray(this.vao);
    gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.DYNAMIC_DRAW);
    
    // Render
    gl.useProgram(this.program);
    gl.drawArrays(gl.POINTS, 0, bookmarks.length);
  }
}
```

## Native OS Integration

### macOS Integration

```rust
#[cfg(target_os = "macos")]
mod macos {
    use cocoa::base::{id, nil};
    use cocoa::appkit::{NSApplication, NSWindow};
    use objc::{msg_send, sel, sel_impl};
    
    pub fn setup_transparent_titlebar(window: &Window) {
        unsafe {
            let ns_window = window.ns_window().unwrap() as id;
            
            // Make titlebar transparent
            let _: () = msg_send![ns_window, setTitlebarAppearsTransparent: true];
            
            // Enable full size content view
            let _: () = msg_send![ns_window, setStyleMask: 
                NSWindowStyleMask::NSFullSizeContentViewWindowMask |
                NSWindowStyleMask::NSResizableWindowMask |
                NSWindowStyleMask::NSTitledWindowMask
            ];
            
            // Set vibrant background
            let visual_effect = NSVisualEffectView::init(nil);
            let _: () = msg_send![visual_effect, setMaterial: 
                NSVisualEffectMaterial::UnderWindowBackground];
            let _: () = msg_send![visual_effect, setBlendingMode:
                NSVisualEffectBlendingMode::BehindWindow];
            let _: () = msg_send![visual_effect, setState:
                NSVisualEffectState::Active];
        }
    }
    
    pub fn register_global_shortcut() {
        let event_monitor = NSEvent::addGlobalMonitorForEventsMatchingMask_handler(
            NSEventMask::KeyDown,
            |event: id| {
                let keycode = event.keyCode();
                let modifiers = event.modifierFlags();
                
                // Cmd+Shift+B for quick bookmark
                if modifiers.contains(NSEventModifierFlags::Command | 
                                     NSEventModifierFlags::Shift) && 
                   keycode == 11 { // 'B' key
                    emit_global_shortcut("quick_bookmark");
                }
            }
        );
    }
}
```

### Windows Integration

```rust
#[cfg(target_os = "windows")]
mod windows {
    use windows::{
        core::*,
        Win32::{
            Foundation::*,
            Graphics::Dwm::*,
            UI::WindowsAndMessaging::*,
        },
    };
    
    pub fn enable_acrylic_backdrop(hwnd: HWND) -> Result<()> {
        unsafe {
            // Enable dark mode
            let dark_mode = true as i32;
            DwmSetWindowAttribute(
                hwnd,
                DWMWA_USE_IMMERSIVE_DARK_MODE,
                &dark_mode as *const _ as *const _,
                size_of::<i32>() as u32,
            )?;
            
            // Enable acrylic backdrop
            let backdrop_type = DWM_SYSTEMBACKDROP_TYPE::DWMSBT_TRANSIENTWINDOW;
            DwmSetWindowAttribute(
                hwnd,
                DWMWA_SYSTEMBACKDROP_TYPE,
                &backdrop_type as *const _ as *const _,
                size_of::<DWM_SYSTEMBACKDROP_TYPE>() as u32,
            )?;
            
            // Extend frame into client area
            let margins = MARGINS {
                cxLeftWidth: -1,
                cxRightWidth: -1,
                cyTopHeight: -1,
                cyBottomHeight: -1,
            };
            DwmExtendFrameIntoClientArea(hwnd, &margins)?;
        }
        
        Ok(())
    }
    
    pub fn register_jumplist_tasks() -> Result<()> {
        let shell = ICustomDestinationList::new()?;
        shell.BeginList()?;
        
        // Add custom tasks
        let tasks = IObjectCollection::new()?;
        
        let quick_add = create_shell_link(
            "Quick Add",
            "bitmarks://quick-add",
            "Quickly add current clipboard URL"
        )?;
        tasks.AddObject(quick_add)?;
        
        shell.AddUserTasks(tasks)?;
        shell.CommitList()?;
        
        Ok(())
    }
}
```

### Linux Integration

```rust
#[cfg(target_os = "linux")]
mod linux {
    use freedesktop_desktop_entry::{DesktopEntry, Exec};
    use zbus::{Connection, dbus_proxy};
    
    #[dbus_proxy(
        interface = "org.freedesktop.portal.Settings",
        default_service = "org.freedesktop.portal.Desktop",
        default_path = "/org/freedesktop/portal/desktop"
    )]
    trait Settings {
        fn read(&self, namespace: &str, key: &str) -> zbus::Result<Variant>;
    }
    
    pub async fn detect_dark_mode() -> Result<bool> {
        let connection = Connection::session().await?;
        let proxy = SettingsProxy::new(&connection).await?;
        
        let color_scheme = proxy.read(
            "org.freedesktop.appearance",
            "color-scheme"
        ).await?;
        
        Ok(color_scheme.get::<u32>()? == 1) // 1 = dark, 0 = light
    }
    
    pub fn create_desktop_entry() -> Result<()> {
        let entry = DesktopEntry {
            name: "Bitmarks".into(),
            comment: Some("Bookmark Manager".into()),
            exec: Exec::new(vec!["bitmarks".into()]),
            icon: Some("bitmarks".into()),
            terminal: false,
            categories: vec!["Network".into(), "Utility".into()],
            mime_types: vec!["x-scheme-handler/bitmarks".into()],
            ..Default::default()
        };
        
        entry.save_to_file("~/.local/share/applications/bitmarks.desktop")?;
        
        Ok(())
    }
}
```

## Performance Optimizations

### Lazy Loading with Intersection Observer

```vue
<template>
  <div class="bookmark-grid">
    <div
      v-for="(bookmark, index) in visibleBookmarks"
      :key="bookmark.id"
      :ref="el => setItemRef(el, index)"
      class="bookmark-item"
    >
      <BookmarkCard :bookmark="bookmark" />
    </div>
    <div ref="sentinel" class="loading-sentinel" />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

const visibleBookmarks = ref<Bookmark[]>([]);
const sentinel = ref<HTMLElement>();
const observer = ref<IntersectionObserver>();

onMounted(() => {
  observer.value = new IntersectionObserver(
    (entries) => {
      if (entries[0].isIntersecting) {
        loadMoreBookmarks();
      }
    },
    {
      root: null,
      rootMargin: '100px',
      threshold: 0.1
    }
  );
  
  if (sentinel.value) {
    observer.value.observe(sentinel.value);
  }
});

const loadMoreBookmarks = async () => {
  const BATCH_SIZE = 50;
  const nextBatch = await invoke<Bookmark[]>('load_bookmarks', {
    offset: visibleBookmarks.value.length,
    limit: BATCH_SIZE
  });
  
  visibleBookmarks.value.push(...nextBatch);
};

onUnmounted(() => {
  observer.value?.disconnect();
});
</script>
```

## Build Configuration

### Tauri Configuration

```json
{
  "tauri": {
    "bundle": {
      "active": true,
      "targets": ["deb", "appimage", "nsis", "msi", "app", "dmg"],
      "identifier": "io.bitmarks.app",
      "icon": ["icons/32x32.png", "icons/128x128.png", "icons/icon.icns", "icons/icon.ico"],
      "resources": ["resources/*"],
      "externalBin": ["binaries/bitmarks-cli"],
      "copyright": "© 2025 Bitmarks",
      "category": "Productivity",
      "shortDescription": "Bookmark Manager",
      "longDescription": "A powerful bookmark management application",
      "deb": {
        "depends": ["libwebkit2gtk-4.0-37", "libssl3"]
      },
      "macOS": {
        "frameworks": [],
        "minimumSystemVersion": "10.15",
        "entitlements": "./entitlements.plist",
        "hardenedRuntime": true,
        "notarize": {
          "appleId": "your-apple-id",
          "password": "your-app-specific-password",
          "teamId": "your-team-id"
        }
      },
      "windows": {
        "certificateThumbprint": null,
        "digestAlgorithm": "sha256",
        "webviewInstallMode": {
          "type": "embedBootstrapper"
        },
        "wix": {
          "language": "en-US",
          "template": "./wix/main.wxs"
        }
      }
    },
    "allowlist": {
      "all": false,
      "fs": {
        "all": false,
        "readFile": true,
        "writeFile": true,
        "scope": ["$APPDATA/*", "$RESOURCE/*"]
      },
      "http": {
        "all": false,
        "request": true,
        "scope": ["https://api.bitmarks.io/*"]
      },
      "shell": {
        "all": false,
        "open": true
      },
      "protocol": {
        "all": false,
        "asset": true,
        "assetScope": ["$RESOURCE/*"]
      }
    },
    "security": {
      "csp": "default-src 'self'; script-src 'self' 'unsafe-eval'; style-src 'self' 'unsafe-inline'"
    }
  }
}
```

## License

MIT