# Bab 5: In-Memory Storage - Inti dari Cache Server

## 5.1 Pendahuluan: Memahami Storage Engine

Setelah kita berhasil membangun server dengan RESP protocol, sekarang saatnya membangun jantung dari cache server: storage engine. Storage engine adalah komponen yang bertanggung jawab untuk menyimpan, mengambil, dan mengelola data di dalam memory.

>    📂 Kode Lengkap Bab Ini:
>    Seluruh kode yang dibahas di bab ini tersedia di GitHub:
>
>    🔗 [github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/05-in-memory-storage](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/05-in-memory-storage)


## 5.1.1 Mengapa In-Memory Storage?

Cache server menggunakan in-memory storage karena beberapa alasan:

| Karakteristik | In-Memory | Disk-Based |
|---------------|-----------|------------|
| Kecepatan | ⚡ Nano/microseconds | 🐢 Milliseconds |
| Latency | < 1ms | 5-10ms |
| Throughput | > 1M ops/sec | ~100K ops/sec |
| Data Persistence | ❌ Volatile | ✅ Persistent |
| Use Case | Caching, Session | Database, File |

Redis dan Pendem menggunakan in-memory storage karena:
1. Kecepatan Ekstrem - Data di RAM 1000x lebih cepat dari disk
2. Throughput Tinggi - Bisa handle jutaan operasi per detik
3. Latency Rendah - Response dalam microseconds
4. Sederhana - Tidak perlu I/O overhead

### 5.1.2 Desain Storage Engine

Storage engine di Pendem memiliki beberapa komponen utama:

```text
┌─────────────────────────────────────────────────────────────┐
│                     Storage Engine                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    Cache                            │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│   │  │    Item     │  │    Item     │  │    Item     │  │   │
│   │  │  key: "a"   │  │  key: "b"   │  │  key: "c"   │  │   │
│   │  │  value: 1   │  │  value: 2   │  │  value: 3   │  │   │
│   │  │  TTL: 10s   │  │  TTL: 0     │  │  TTL: 5s    │  │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│   │                                                     │   │
│   │  Operations: SET, GET, DEL, TTL                     │   │
│   │  Concurrent: RWMutex                                │   │
│   │  Cleanup: Background goroutine                      │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.1.3 Struktur Folder

```text
pendem/
├── cmd/
│   └── main.go
├── internal/
│   ├── engine/           # 📁 Storage Engine
│   │   ├── cache.go      # Cache implementation
│   │   └── item.go       # Cache item
│   ├── server/           # Server & protocol
│   ├── handler/          # Command handlers
│   └── config/           # Configuration
├── test/
│   └── integration/      # Integration tests
└── go.mod
```

## 5.2 Desain Cache Item

Cache item adalah unit terkecil dalam storage engine. Setiap item menyimpan value dan informasi TTL.

### 5.2.1 Struktur Item

```go
// internal/engine/item.go
package engine

import "time"

// Item menyimpan data cache dengan expiration
type Item[T any] struct {
    Value      T     // Nilai yang disimpan
    Expiration int64 // Unix timestamp (nanosecond) untuk expiration
    CreatedAt  int64 // Unix timestamp saat item dibuat
}

// NewItem membuat item baru dengan TTL
func NewItem[T any](value T, ttl time.Duration) *Item[T] {
    now := time.Now()
    var exp int64
    
    // Jika TTL > 0, set expiration
    if ttl > 0 {
        exp = now.Add(ttl).UnixNano()
    }
    
    return &Item[T]{
        Value:      value,
        Expiration: exp,
        CreatedAt:  now.UnixNano(),
    }
}

// IsExpired mengecek apakah item sudah expired
func (i *Item[T]) IsExpired() bool {
    // Expiration == 0 berarti no TTL (never expire)
    if i.Expiration == 0 {
        return false
    }
    return time.Now().UnixNano() > i.Expiration
}

// TTL mengembalikan sisa waktu hidup item
func (i *Item[T]) TTL() time.Duration {
    if i.Expiration == 0 {
        return -1 // No TTL
    }
    
    remaining := i.Expiration - time.Now().UnixNano()
    if remaining < 0 {
        return 0
    }
    return time.Duration(remaining)
}
```

Penjelasan:
- Generic Type [T any]: Item bisa menyimpan tipe data apapun
- Expiration: Disimpan dalam nanosecond untuk presisi tinggi
- No TTL: Expiration = 0 berarti item tidak pernah expired
- CreatedAt: Untuk debugging dan monitoring

### 5.2.2 Visualisasi TTL

```text
Timeline: Item dengan TTL 10 detik

Created at: 10:00:00.000
Expires at: 10:00:10.000

10:00:00 ──── 10:00:05 ──── 10:00:10 ──── 10:00:15
   │              │              │              │
   │              │              │              │
 Created       Check          Expired       Check
              (valid)          ⏰            (expired)
```

## 5.3 Implementasi Cache

Cache adalah container yang menyimpan semua items dengan akses concurrent-safe.

### 5.3.1 Struktur Cache

```go
// internal/engine/cache.go
package engine

import (
    "sync"
    "time"
)

// Cache adalah storage engine utama
type Cache[V any] struct {
    items map[string]*Item[V] // key → item
    mu    sync.RWMutex        // Concurrent access protection
}

// NewCache membuat instance cache baru
func NewCache[V any]() *Cache[V] {
    cache := &Cache[V]{
        items: make(map[string]*Item[V]),
    }
    
    // Start background cleanup
    go cache.cleanupLoop()
    
    return cache
}
```

Penjelasan:
- Map: Penyimpanan utama dengan O(1) access time
- RWMutex: Multiple readers, single writer untuk performance
- Background Cleanup: Periodic cleanup goroutine

### 5.3.2 Cache Operations

#### a. Set - Menyimpan Data

```go
// Set menyimpan value dengan TTL
func (c *Cache[V]) Set(key string, value V, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    item := NewItem(value, ttl)
    c.items[key] = item
}
```

**Alur SET:**

```text
1. Client mengirim SET key value
2. Cache membuat Item dengan TTL
3. Item disimpan di map
4. Response OK
```

#### b. Get - Mengambil Data

```go
// Get mengambil value berdasarkan key
func (c *Cache[V]) Get(key string) (V, bool) {
    var zero V
    
    // Read lock untuk concurrent reads
    c.mu.RLock()
    item, exists := c.items[key]
    c.mu.RUnlock()
    
    if !exists {
        return zero, false
    }
    
    // Cek expired
    if item.IsExpired() {
        // Lazy deletion: hapus saat diakses
        c.mu.Lock()
        delete(c.items, key)
        c.mu.Unlock()
        return zero, false
    }
    
    return item.Value, true
}
```

**Alur GET:**

```text
1. Client mengirim GET key
2. Cache cek key di map
3. Jika ada, cek expiration
4. Jika expired → hapus dan return nil
5. Jika valid → return value
```

#### c. Delete - Menghapus Data

```go
// Delete menghapus key dari cache
func (c *Cache[V]) Delete(key string) bool {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if _, exists := c.items[key]; exists {
        delete(c.items, key)
        return true
    }
    return false
}
```

#### d. TTL - Cek Sisa Waktu

```go
// TTL mengembalikan sisa waktu hidup key dalam detik
func (c *Cache[V]) TTL(key string) int64 {
    c.mu.RLock()
    item, exists := c.items[key]
    c.mu.RUnlock()
    
    if !exists {
        return -2 // Redis: key doesn't exist
    }
    
    if item.IsExpired() {
        // Lazy deletion
        c.mu.Lock()
        delete(c.items, key)
        c.mu.Unlock()
        return -2
    }
    
    ttl := item.TTL()
    if ttl < 0 {
        return -1 // No TTL
    }
    
    return int64(ttl.Seconds())
}
```

### 5.3.3 Concurrent Access dengan RWMutex

Kenapa perlu Mutex?

```go
// ❌ TANPA MUTEX - RACE CONDITION!
func (c *Cache) Set(key string, value string) {
    c.items[key] = value  // Goroutine 1
}

func (c *Cache) Get(key string) string {
    return c.items[key]   // Goroutine 2 (bersamaan!)
}
// 💥 Data race! Map write dan read bersamaan

// ✅ DENGAN RWMUTEX - AMAN!
func (c *Cache) Set(key string, value string) {
    c.mu.Lock()           // Goroutine 1 acquire lock
    defer c.mu.Unlock()
    c.items[key] = value
}

func (c *Cache) Get(key string) string {
    c.mu.RLock()          // Goroutine 2 bisa read bersama
    defer c.mu.RUnlock()
    return c.items[key]
}
```

Perbandingan Lock:
| Lock Type | Readers | Writers | Performance |
|-----------|---------|---------|-------------|
| RWMutex | Multiple | Single | High (reads) |
| Mutex | Single | Single | Lower |
| No Lock | ❌ Race | ❌ Race | ❌ Unsafe |

### 5.3.4 Background Cleanup

```go
// cleanupLoop periodik menghapus item expired
func (c *Cache[V]) cleanupLoop() {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()
    
    for range ticker.C {
        c.cleanupExpired()
    }
}

// cleanupExpired menghapus semua item expired
func (c *Cache[V]) cleanupExpired() {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    for key, item := range c.items {
        if item.IsExpired() {
            delete(c.items, key)
        }
    }
}
```

Dua Strategi Cleanup:
1. Lazy Deletion: Hapus saat diakses (GET, TTL)
2. Periodic Cleanup: Hapus setiap 5 menit

Kenapa dua strategi?
- Lazy: Hemat CPU, tapi memory bisa penuh
- Periodic: Jaga memory, tapi ada overhead

Kombinasi = Efisien & Memory-safe!

## 5.4 Command Handlers

Setelah storage engine siap, kita integrasikan dengan handler commands.

### 5.4.1 Handler Structure

```go
// internal/handler/handler.go
package handler

import (
    "pendem/internal/engine"
    "pendem/internal/server"
    "strconv"
    "strings"
    "time"
)

type Handler struct {
    server *server.Server
    cache  *engine.Cache[string] // Cache with string values
}

func NewHandler(server *server.Server, cache *engine.Cache[string]) *Handler {
    return &Handler{
        server: server,
        cache:  cache,
    }
}
```

### 5.4.2 GET Command

```go
func (h *Handler) Get(args []string) server.RESPValue {
    // Validasi argumen
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'get' command",
        }
    }
    
    key := args[0]
    val, found := h.cache.Get(key)
    
    // Redis: key tidak ditemukan → null bulk string
    if !found {
        return server.RESPValue{
            Type:   server.BulkString,
            Str:    "",
            IsNull: true,
        }
    }
    
    return server.RESPValue{
        Type: server.BulkString,
        Str:  val,
    }
}
```

Contoh Interaksi:

```bash
# Key tidak ada
127.0.0.1:6379> GET nonexistent
(nil)

# Key ada
127.0.0.1:6379> SET key value
OK
127.0.0.1:6379> GET key
"value"
```

### 5.4.3 SET Command

```go
func (h *Handler) Set(args []string) server.RESPValue {
    // Validasi argumen minimum
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'set' command",
        }
    }
    
    key := args[0]
    value := args[1]
    ttl := time.Duration(0)
    
    // Optional TTL
    if len(args) >= 4 {
        switch strings.ToUpper(args[2]) {
        case "EX": // Seconds
            if secs, err := strconv.Atoi(args[3]); err == nil {
                ttl = time.Duration(secs) * time.Second
            }
        case "PX": // Milliseconds
            if ms, err := strconv.Atoi(args[3]); err == nil {
                ttl = time.Duration(ms) * time.Millisecond
            }
        }
    }
    
    h.cache.Set(key, value, ttl)
    
    return server.RESPValue{
        Type: server.SimpleString,
        Str:  "OK",
    }
}
```

Contoh Interaksi:

```bash
# SET tanpa TTL
127.0.0.1:6379> SET key value
OK

# SET dengan TTL 10 detik
127.0.0.1:6379> SET key value EX 10
OK

# SET dengan TTL 500ms
127.0.0.1:6379> SET key value PX 500
OK
```

### 5.4.4 DEL Command

```go
func (h *Handler) Delete(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'del' command",
        }
    }
    
    count := 0
    for _, key := range args {
        if h.cache.Delete(key) {
            count++
        }
    }
    
    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(count),
    }
}
```

Contoh Interaksi:

```bash
# Delete 1 key
127.0.0.1:6379> DEL key
(integer) 1

# Delete multiple keys
127.0.0.1:6379> DEL key1 key2 key3
(integer) 3

# Delete non-existent key
127.0.0.1:6379> DEL nonexistent
(integer) 0
```

### 5.4.5 TTL Command

```go
func (h *Handler) TTL(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'ttl' command",
        }
    }
    
    key := args[0]
    ttl := h.cache.TTL(key)
    
    return server.RESPValue{
        Type: server.Integer,
        Int:  ttl,
    }
}
```

Contoh Interaksi:

```bash
# Key dengan TTL
127.0.0.1:6379> SET key value EX 10
OK
127.0.0.1:6379> TTL key
(integer) 8

# Key tanpa TTL
127.0.0.1:6379> SET permanent value
OK
127.0.0.1:6379> TTL permanent
(integer) -1

# Key tidak ada
127.0.0.1:6379> TTL nonexistent
(integer) -2
```

## 5.5 Integrasi dengan Server

### 5.5.1 Update Main

```go
// cmd/main.go
package main

import (
    "fmt"
    "log"
    "os"
    "os/signal"
    "pendem/internal/config"
    "pendem/internal/engine"
    "pendem/internal/handler"
    "pendem/internal/server"
    "syscall"
    "time"
)

func main() {
    // Config
    cfg := config.Config{
        Server: config.ServerConfig{
            MaxConnections: 50_000,
            ReadTimeout:    30 * time.Second,
            WriteTimeout:   10 * time.Second,
            IdleTimeout:    60 * time.Second,
        },
    }
    
    // Create components
    cache := engine.NewCache[string]()  // ✅ Storage engine
    srv := server.NewServerWithConfig(":6379", cfg.Server)
    h := handler.NewHandler(srv, cache) // ✅ Handler with cache
    
    // Register handlers
    srv.RegisterHandler("PING", h.Ping)
    srv.RegisterHandler("MEMORY", h.Memory)
    srv.RegisterHandler("GET", h.Get)
    srv.RegisterHandler("SET", h.Set)
    srv.RegisterHandler("DEL", h.Delete)
    srv.RegisterHandler("TTL", h.TTL)
    
    // ... start server & graceful shutdown
}
```

### 5.5.2 Component Interaction

```text
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│   Server    │────▶│   Handler   │
│  (redis-cli)│     │  (TCP+RESP) │     │  (Commands) │
└─────────────┘     └─────────────┘     └─────────────┘
                                                    │
                                                    ▼
                                         ┌─────────────────────┐
                                         │   Cache Engine      │
                                         │  ┌───────────────┐  │
                                         │  │  Map + Mutex  │  │
                                         │  └───────────────┘  │
                                         └─────────────────────┘
```

## 5.6 Testing

### 5.6.1 Testing dengan redis-cli

```bash
# Start server
go run cmd/main.go

# Connect with redis-cli
redis-cli -h host.docker.internal -p 6379

# Test basic operations
127.0.0.1:6379> PING
PONG

127.0.0.1:6379> SET user:1 "John Doe"
OK

127.0.0.1:6379> GET user:1
"John Doe"

127.0.0.1:6379> TTL user:1
(integer) -1

127.0.0.1:6379> SET session:1 "active" EX 10
OK

127.0.0.1:6379> TTL session:1
(integer) 8

# Wait 10 seconds
127.0.0.1:6379> GET session:1
(nil)

127.0.0.1:6379> DEL user:1
(integer) 1

127.0.0.1:6379> GET user:1
(nil)
```

### 5.6.2 Integration Test

```go
// test/integration/concurrency_test.go
package integration

import (
    "pendem/internal/config"
    "pendem/internal/engine"
    "pendem/internal/handler"
    "pendem/internal/server"
    "sync"
    "testing"
    "time"
)

func TestConcurrentReadWrite(t *testing.T) {
    // Setup
    cache := engine.NewCache[string]()
    srv := server.NewServerWithConfig(":16379", config.DefaultConfig().Server)
    h := handler.NewHandler(srv, cache)
    srv.RegisterHandler("SET", h.Set)
    srv.RegisterHandler("GET", h.Get)
    
    go srv.Start()
    time.Sleep(100 * time.Millisecond)
    defer srv.Shutdown()
    
    // Concurrent operations
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            key := fmt.Sprintf("key_%d", id)
            sendCommand("SET", key, "value")
            sendCommand("GET", key)
        }(i)
    }
    wg.Wait()
    // No panic = success ✅
}
```

## 5.7 Performance Considerations

### 5.7.1 Memory Usage

| Komponen | Approx Memory |
|----------|---------------|
| Map entry | ~40 bytes |
| Item struct | ~24 bytes |
| Value string | Variable |
| Total per key | ~100 bytes |

Contoh:

```text
1,000,000 keys × ~100 bytes = ~100 MB memory
10,000,000 keys × ~100 bytes = ~1 GB memory
```

### 5.7.2 Optimasi Memory

```go
// ❌ Inefficient: String copying
func (c *Cache) Set(key string, value string) {
    c.items[key] = value // Copies string
}

// ✅ Efficient: Use pointer if needed
func (c *Cache) Set(key string, value *string) {
    c.items[key] = value // Only copies pointer
}
```

### 5.7.3 Lock Contention

```text
High contention: Banyak writer
    → RWMutex menjadi bottleneck
    → Solution: Sharding (Bab 7)

Low contention: Banyak reader
    → RWMutex optimal
    → Multiple readers can proceed
```

## 5.8 Best Practices

### 5.8.1 Concurrent Access

```go
// ✅ DO: Always use mutex
func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    // ...
}

// ❌ DON'T: Access map without lock
func (c *Cache) Get(key string) (string, bool) {
    return c.items[key] // Race condition!
}
```

### 5.8.2 Cleanup Strategy

```go
// ✅ DO: Lazy deletion + periodic cleanup
// 1. Delete expired on access
// 2. Periodic cleanup every 5 minutes

// ❌ DON'T: Only lazy deletion
// Memory leak! Expired items stay forever

// ❌ DON'T: Only periodic cleanup
// Stale data returned before cleanup
```

### 5.8.3 TTL Design

```go
// ✅ DO: Use nanoseconds for precision
Expiration: time.Now().Add(ttl).UnixNano()

// ❌ DON'T: Use seconds
Expiration: time.Now().Unix() + int64(ttl.Seconds())
// Loss of precision for sub-second TTL
```

### 5.8.4 Error Handling

```go
// ✅ DO: Return appropriate null values
if !found {
    return RESPValue{
        Type: BulkString,
        Str:  "",
        IsNull: true,  // Redis null
    }
}

// ❌ DON'T: Return empty string
return RESPValue{
    Type: BulkString,
    Str:  "",  // Ambiguous: empty vs null
}
```

## 5.9 Ringkasan

Yang Sudah Kita Pelajari:

1. Storage Engine Design
- Item dengan TTL
- Map-based cache
- RWMutex for concurrency

2. Cache Operations
- SET, GET, DEL, TTL
- Lazy deletion
- Periodic cleanup

3. Command Integration
- RESP handlers
- Redis compatibility
- Error handling

4. Testing & Performance
- redis-cli testing
- Integration testing
- Memory considerations

### Sebelum vs Sesudah 

| Aspek | Sebelum (Bab 4) | Sesudah (Bab 5) |
|-------|-----------------|-----------------|
| Storage | ❌ No storage | ✅ In-memory cache |
| Commands | PING, MEMORY | + GET, SET, DEL, TTL |
| TTL | ❌ No | ✅ With expiration |
| Concurrency | ❌ Not safe | ✅ RWMutex |
| Cleanup | ❌ No | ✅ Background cleanup |
| Redis Compatible | ⚠️ Partial | ✅ Full |

## Next Steps

Dengan In-Memory Storage, server Pendem sekarang memiliki storage engine yang:
- ✅ Menyimpan data di memory
- ✅ Mendukung TTL
- ✅ Aman untuk concurrent access
- ✅ Kompatibel dengan Redis commands

Kita sudah memiliki storage engine yang solid, namun saat ini operasi penyimpana data akan gagal ketika storage penuh. Bab selanjutnya, kita akan menambahkan eviction policy, strategi untuk memastikan operasi penyimpanan data sukses walalupun storage penuh. Eviction policy yang akan dibahas adalah LRU, yaitu saat cache penuh, otomatis akan menghapus item yang paling jarang digunakan.