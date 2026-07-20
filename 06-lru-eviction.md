# Bab 6: LRU Eviction - Mengelola Memory dengan Cerdas

## 6.1 Pendahuluan: Mengapa Perlu Eviction?

Setelah kita berhasil membangun in-memory storage di Bab 5, muncul pertanyaan penting: Apa yang terjadi ketika cache penuh?

>    📂 Kode Lengkap Bab Ini:
>    Seluruh kode yang dibahas di bab ini tersedia di GitHub:
>
>    🔗 [github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/06-lru-eviction](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/06-lru-eviction)



### 6.1.1 Masalah: Cache Penuh

Cache server bekerja dengan menyimpan data di memory (RAM). Memory terbatas, sementara data yang disimpan bisa terus bertambah.

```text
┌─────────────────────────────────────────────────────────────┐
│                    Tanpa Eviction                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Memory: [████████████████████████████████████████████]    │
│           ↑                                                 │
│           │                                                 │
│   💥 Out of Memory! Server Crash!                           │
│                                                             │
│   ❌ Tidak ada mekanisme untuk menghapus data lama          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Dengan Eviction                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Memory: [██████████████████████████████░░░░░░░░░░░░░░]    │
│           ↑                         ↑                       │
│           │                         │                       │
│   Data baru masuk          Data lama dihapus (evicted)      │
│                                                             │
│   ✅ Memory tetap terkendali                                │
└─────────────────────────────────────────────────────────────┘
```

### 6.1.2 Solusi: Eviction Policy

Eviction Policy adalah aturan untuk menentukan data mana yang akan dihapus ketika memory penuh.

| Policy | Cara Kerja | Kelebihan | Kekurangan |
|--------|------------|-----------|------------|
| LRU | Hapus yang paling jarang digunakan | Sederhana, efektif | Overhead tracking |
| LFU | Hapus yang paling jarang diakses | Cocok untuk popularitas | Kompleks |
| TTL | Hapus yang sudah expired | Otomatis | Tidak untuk memory |
| Random | Hapus random | Sederhana | Tidak efektif |

Pendem menggunakan LRU (Least Recently Used) karena:
1. ✅ Sederhana dan cepat
2. ✅ Efektif untuk caching
3. ✅ Mendekati perilaku ideal (temporal locality)
4. ✅ Mudah diimplementasikan

### 6.1.3 Konsep LRU

LRU (Least Recently Used) menghapus item yang paling lama tidak diakses.

```text
┌─────────────────────────────────────────────────────────────┐
│                      LRU Concept                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Access Order:  A → B → C → D → E                          │
│                   ↑            ↑                            │
│              Most Recent   Least Recent                     │
│              (Keep)        (Evict)                          │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  STEP 1: Cache penuh [A, B, C, D]                   │   │
│   │                                                     │   │
│   │  STEP 2: Akses A → [A, B, C, D] (A jadi terbaru)    │   │
│   │                                                     │   │
│   │  STEP 3: Tambah E → Hapus yang paling lama (B)      │   │
│   │                                                     │   │
│   │  STEP 4: Hasil [A, C, D, E]                         │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   Item yang diakses → Pindah ke depan (most recent)         │
│   Item yang tidak diakses → Pergeser ke belakang            │
│   Item paling belakang → Dievict (dihapus)                  │
└─────────────────────────────────────────────────────────────┘
```

## 6.2 Desain LRU

### 6.2.1 Data Structure

LRU membutuhkan dua struktur data:
1. Map → O(1) akses berdasarkan key
2. Doubly Linked List → O(1) move to front, remove from back

```text
┌─────────────────────────────────────────────────────────────┐
│                    LRU Data Structure                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    MAP                              │   │
│   │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                 │   │
│   │  │ "A" │─▶│ "B" │─▶│ "C" │─▶│ "D" │                 │   │
│   │  └─────┘  └─────┘  └─────┘  └─────┘                 │   │
│   └─────────────────────────────────────────────────────┘   │
│                         │                                   │
│                         ▼                                   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              DOUBLY LINKED LIST                     │   │
│   │                                                     │   │
│   │   Front (Most Recent)              Back (Least)     │   │
│   │      ┌───┐    ┌───┐    ┌───┐    ┌───┐               │   │
│   │      │ A │ ←→ │ B │ ←→ │ C │ ←→ │ D │               │   │
│   │      └───┘    └───┘    └───┘    └───┘               │   │
│   │                                                     │   │
│   │   • Access A → Move to front                        │   │
│   │   • Add E → Evict D (back)                          │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 6.2.2 Kompleksitas

| Operasi | Map | Linked List | Total |
|---------|-----|-------------|-------|
| Get | O(1) | O(1) (move to front) | O(1) |
| Add | O(1) | O(1) (push front) | O(1) |
| Remove | O(1) | O(1) (remove) | O(1) |
| Evict | O(1) | O(1) (pop back) | O(1) |

### 6.2.3 Memory Tracking

Selain tracking access order, LRU juga perlu tracking memory usage:

```text
┌─────────────────────────────────────────────────────────────┐
│                  Memory Tracking                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   currentMem: 800 MB  ← Total memory used                   │
│   maxMemory:   1 GB   ← Memory limit                        │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  Add new item: 50 MB                                │   │
│   │  currentMem + 50 = 850 MB → Still OK                │   │
│   │                                                     │   │
│   │  Add new item: 200 MB                               │   │
│   │  currentMem + 200 = 1050 MB → OVER!                 │   │
│   │                                                     │   │
│   │  Evict until memory fits                            │   │
│   │  Remove oldest: 150 MB → currentMem = 900 MB        │   │
│   │  Remove oldest: 100 MB → currentMem = 800 MB        │   │
│   │  currentMem + 200 = 1000 MB → OK!                   │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 6.3 Implementasi Evictor Interface

### 6.3.1 Interface Design

Kita desain Evictor interface agar support multiple policies:

```go
// internal/engine/evictor.go
package engine

// Evictor defines the interface for eviction policies
type Evictor[V any] interface {
    // Add adds or updates an item
    Add(key string, value *Item[V])
    
    // Get retrieves an item and marks it as recently used
    Get(key string) (*Item[V], bool)
    
    // Remove deletes an item
    Remove(key string) bool
    
    // Len returns the number of items
    Len() int
    
    // MaxCapacity returns the maximum capacity
    MaxCapacity() int
    
    // MemoryUsage returns current memory usage in bytes
    MemoryUsage() int64
    
    // ForEach iterates over all items
    ForEach(fn func(key string, item *Item[V]) bool)
}
```

Keuntungan Interface:
- ✅ Strategy Pattern - mudah ganti policy
- ✅ Cache tidak tergantung implementasi spesifik
- ✅ Mudah testing
- ✅ Future ready (LFU, TTL)

## 6.4 Implementasi LRU

### 6.4.1 Struct LRU

```go
// internal/engine/lru.go
package engine

import (
    "container/list"
    "log"
)

// LRU implements Least Recently Used eviction policy
type LRU[V any] struct {
    capacity   int                    // Max items
    maxMemory  int64                  // Max memory in bytes
    currentMem int64                  // Current memory usage
    items      map[string]*list.Element // key → list element
    order      *list.List             // Doubly linked list
    logger     *log.Logger            // Optional logging
}

// lruEntry stores item data in the list
type lruEntry[V any] struct {
    key   string
    value *Item[V]
    size  int64 // Estimated memory size
}
```

Penjelasan:
- capacity: Maksimum jumlah item
- maxMemory: Maksimum memory dalam bytes
- currentMem: Memory yang sedang digunakan
- items: Map untuk O(1) lookup
- order: Linked list untuk tracking access order
- logger: Untuk logging eviction events

### 6.4.2 Constructor

```go
// NewLRU creates an LRU with capacity limit only
func NewLRU[V any](capacity int) *LRU[V] {
    return &LRU[V]{
        capacity:  capacity,
        maxMemory: 0,
        items:     make(map[string]*list.Element),
        order:     list.New(),
    }
}

// NewLRUWithMemory creates an LRU with both capacity and memory limits
func NewLRUWithMemory[V any](capacity int, maxMemory int64, logger *log.Logger) *LRU[V] {
    return &LRU[V]{
        capacity:   capacity,
        maxMemory:  maxMemory,
        currentMem: 0,
        items:      make(map[string]*list.Element),
        order:      list.New(),
        logger:     logger,
    }
}
```

### 6.4.3 Add - Menambah atau Update Item

```go
func (l *LRU[V]) Add(key string, value *Item[V]) {
    // Calculate item size
    itemSize := l.calculateSize(key, value.Value)

    // Check memory limit
    if l.maxMemory > 0 && l.currentMem+itemSize > l.maxMemory {
        // Evict until enough memory
        for l.currentMem+itemSize > l.maxMemory && l.order.Len() > 0 {
            l.removeOldest()
        }
    }

    // If key exists, update and move to front
    if elem, exists := l.items[key]; exists {
        l.order.MoveToFront(elem)
        entry := elem.Value.(*lruEntry[V])
        // Update memory usage
        l.currentMem -= entry.size
        entry.value = value
        entry.size = itemSize
        l.currentMem += itemSize
        return
    }

    // If capacity reached, remove oldest
    if l.capacity > 0 && l.order.Len() >= l.capacity {
        l.removeOldest()
    }

    // Add new item
    elem := l.order.PushFront(&lruEntry[V]{
        key:   key,
        value: value,
        size:  itemSize,
    })
    l.items[key] = elem
    l.currentMem += itemSize
}
```

Alur Add:

```text
1. Hitung ukuran item
2. Cek memory limit → evict jika perlu
3. Jika key sudah ada → update dan move to front
4. Jika capacity penuh → evict oldest
5. Tambah item baru di front
```

### 6.4.4 Get - Mengambil Item

```go
func (l *LRU[V]) Get(key string) (*Item[V], bool) {
    if elem, exists := l.items[key]; exists {
        // Move to front (most recently used)
        l.order.MoveToFront(elem)
        entry := elem.Value.(*lruEntry[V])
        return entry.value, true
    }
    return nil, false
}
```

Alur Get:

```text
1. Cari key di map
2. Jika ada → pindahkan ke front (most recent)
3. Return value
```

### 6.4.5 Remove - Menghapus Item

```go
func (l *LRU[V]) Remove(key string) bool {
    if elem, exists := l.items[key]; exists {
        entry := elem.Value.(*lruEntry[V])
        l.currentMem -= entry.size
        l.order.Remove(elem)
        delete(l.items, key)
        return true
    }
    return false
}
```

### 6.4.6 Evict - Menghapus Item Paling Tua

```go
func (l *LRU[V]) removeOldest() {
    elem := l.order.Back()
    if elem != nil {
        entry := elem.Value.(*lruEntry[V])
        l.currentMem -= entry.size
        l.order.Remove(elem)
        delete(l.items, entry.key)

        if l.logger != nil {
            l.logger.Printf("LRU evicted: key=%s, size=%d bytes",
                entry.key, entry.size)
        }
    }
}

func (l *LRU[V]) MaxCapacity() int {
    return l.capacity
}
```

### 6.4.7 Size Calculation

```go
func (l *LRU[V]) calculateSize(key string, value V) int64 {
    size := int64(len(key))

    // Type-based size estimation
    switch v := any(value).(type) {
    case string:
        size += int64(len(v))
    case int, int8, int16, int32, int64:
        size += 8
    case uint, uint8, uint16, uint32, uint64:
        size += 8
    case float32, float64:
        size += 8
    case bool:
        size += 1
    default:
        // Fallback estimation for complex types
        size += 64
    }

    return size
}
```

Kenapa perlu size calculation?
-  Memory limit harus di-track
- Berbeda tipe data, berbeda ukuran
- Estimasi cukup akurat untuk monitoring

## 6.5 Integrasi dengan Cache

### 6.5.1 Update Config

```go
// internal/config/config.go
type EngineConfig struct {
    MaxMemory       int64  // Max memory in bytes
    EvictionPolicy  string // "lru", "lfu", "ttl"
    EvictorCapacity int    // Max items
    DefaultTTL      time.Duration
}

func DefaultConfig() Config {
    return Config{
        Engine: EngineConfig{
            MaxMemory:       1024 * 1024 * 1024, // 1GB
            EvictionPolicy:  "lru",
            EvictorCapacity: 10000,
            DefaultTTL:      0,
        },
    }
}
```

### 6.5.2 Update Cache dengan Evictor

```go
// internal/engine/cache.go
package engine

import (
    "log"
    "pendem/internal/config"
    "sync"
    "time"
)

type Cache[V any] struct {
    evictor Evictor[V]
    policy  string
    mu      sync.RWMutex
}

func NewCache[V any](cfg config.EngineConfig, logger *log.Logger) *Cache[V] {
    var evictor Evictor[V]
    policy := cfg.EvictionPolicy

    if policy == "" {
        policy = "lru"
    }

    switch policy {
    case "lru":
        evictor = NewLRUWithMemory[V](cfg.EvictorCapacity, cfg.MaxMemory, logger)
    case "lfu":
        // Future: LFU implementation
        logger.Printf("LFU not implemented yet, falling back to LRU")
        evictor = NewLRUWithMemory[V](cfg.EvictorCapacity, cfg.MaxMemory, logger)
        policy = "lru"
    case "ttl":
        // Future: TTL eviction
        logger.Printf("TTL eviction not implemented yet, falling back to LRU")
        evictor = NewLRUWithMemory[V](cfg.EvictorCapacity, cfg.MaxMemory, logger)
        policy = "lru"
    default:
        logger.Printf("Unknown eviction policy '%s', using LRU", cfg.EvictionPolicy)
        evictor = NewLRUWithMemory[V](cfg.EvictorCapacity, cfg.MaxMemory, logger)
        policy = "lru"
    }

    cache := &Cache[V]{
        evictor: evictor,
        policy:  policy,
    }
    go cache.cleanupLoop()
    return cache
}
```

### 6.5.3 Cache Operations

```go
func (c *Cache[V]) Set(key string, value V, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    item := NewItem(value, ttl)
    c.evictor.Add(key, item)
}

func (c *Cache[V]) Get(key string) (V, bool) {
    var zero V

    c.mu.RLock()
    item, exists := c.evictor.Get(key)
    c.mu.RUnlock()

    if !exists {
        return zero, false
    }

    // Lazy deletion: remove expired items on access
    if item.IsExpired() {
        c.mu.Lock()
        c.evictor.Remove(key)
        c.mu.Unlock()
        return zero, false
    }

    return item.Value, true
}

func (c *Cache[V]) Delete(key string) bool {
    c.mu.Lock()
    defer c.mu.Unlock()

    return c.evictor.Remove(key)
}

func (c *Cache[V]) TTL(key string) int64 {
    c.mu.RLock()
    item, exists := c.evictor.Get(key)
    c.mu.RUnlock()

    if !exists {
        return -2
    }

    if item.IsExpired() {
        c.mu.Lock()
        c.evictor.Remove(key)
        c.mu.Unlock()
        return -2
    }

    ttl := item.TTL()
    if ttl < 0 {
        return -1
    }

    return int64(ttl.Seconds())
}
```

### 6.5.4 Additional Methods

```go
// Policy returns the current eviction policy
func (c *Cache[V]) Policy() string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.policy
}

// Size returns the number of items currently in cache
func (c *Cache[V]) Size() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.evictor.Len()
}

// MaxCapacity returns the maximum capacity
func (c *Cache[V]) MaxCapacity() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.evictor.MaxCapacity()
}

// MemoryUsage returns current memory usage in bytes
func (c *Cache[V]) MemoryUsage() int64 {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.evictor.MemoryUsage()
}
```

## 6.6 POLICY Command

### 6.6.1 Handler

```go
// internal/handler/handler.go
func (h *Handler) Policy(args []string) server.RESPValue {
    if len(args) > 0 {
        // Future: policy for specific key
        return server.RESPValue{
            Type: server.SimpleString,
            Str:  fmt.Sprintf("Policy for key '%s' not implemented yet", args[0]),
        }
    }

    return server.RESPValue{
        Type: server.SimpleString,
        Str: fmt.Sprintf("Eviction policy: %s (items: %d, max: %d)",
            h.cache.Policy(),
            h.cache.Size(),
            h.cache.MaxCapacity(),
        ),
    }
}
```

### 6.6.2 Register Command

```go
// cmd/main.go
srv.RegisterHandler("POLICY", h.Policy)
```

### 6.6.3 Testing

```bash
redis-cli -h host.docker.internal -p 6379

127.0.0.1:6379> SET key1 value1
OK
127.0.0.1:6379> SET key2 value2
OK
127.0.0.1:6379> SET key3 value3
OK

127.0.0.1:6379> POLICY
"Eviction policy: lru (items: 3, max: 10000)"

127.0.0.1:6379> GET key1
"value1"

127.0.0.1:6379> POLICY
"Eviction policy: lru (items: 3, max: 10000)"
```

## 6.7 Testing LRU

### 6.7.1 Test dengan redis-cli

```bash

# Start server dengan capacity kecil (testing)
# Ubah config: EvictorCapacity: 5

redis-cli -h host.docker.internal -p 6379

# 1. Set 5 items
127.0.0.1:6379> SET key1 value1
OK
127.0.0.1:6379> SET key2 value2
OK
127.0.0.1:6379> SET key3 value3
OK
127.0.0.1:6379> SET key4 value4
OK
127.0.0.1:6379> SET key5 value5
OK

# 2. Access key1 (most recent)
127.0.0.1:6379> GET key1
"value1"

# 3. Add key6 → should evict key2 (least recently used)
127.0.0.1:6379> SET key6 value6
OK

# 4. Check eviction
127.0.0.1:6379> GET key2
(nil)  # ← key2 evicted!

# 5. key1 still exists
127.0.0.1:6379> GET key1
"value1"

# 6. Check policy
127.0.0.1:6379> POLICY
"Eviction policy: lru (items: 5, max: 5)"
```

### 6.7.2 Server Logs

```bash
[SERVER] 2026/07/19 10:00:00 New connection from 127.0.0.1
[SERVER] 2026/07/19 10:00:01 Command from 127.0.0.1: SET key1 value1
[SERVER] 2026/07/19 10:00:02 Command from 127.0.0.1: SET key2 value2
[SERVER] 2026/07/19 10:00:03 Command from 127.0.0.1: SET key3 value3
[SERVER] 2026/07/19 10:00:04 Command from 127.0.0.1: SET key4 value4
[SERVER] 2026/07/19 10:00:05 Command from 127.0.0.1: SET key5 value5
[SERVER] 2026/07/19 10:00:06 Command from 127.0.0.1: GET key1
[SERVER] 2026/07/19 10:00:07 Command from 127.0.0.1: SET key6 value6
[SERVER] 2026/07/19 10:00:07 LRU evicted: key=key2, size=10 bytes
```

## 6.8 Performance Considerations

### 6.8.1 Time Complexity

| Operasi | Complexity | Keterangan |
|---------|------------|------------|
| Add | O(1) | Map + List push front |
| Get | O(1) | Map + List move to front |
| Remove | O(1) | Map + List remove |
| Evict | O(1) | List pop back |

### 6.8.2 Memory Overhead

```text
Per item overhead:
- Map entry: ~40 bytes
- List element: ~24 bytes
- lruEntry: ~16 bytes
- Key string: variable
- Value: variable

Total overhead per item: ~80-100 bytes
```

### 6.8.3 Optimasi yang Dilakukan

1. Map untuk O(1) lookup
2. Linked List untuk O(1) reorder
3. Memory tracking untuk limit
4. Generic type untuk type safety
5. Logger untuk debugging

## 6.9 Best Practices

### 6.9.1 Use Interface

```go
// ✅ DO: Use interface for flexibility
type Evictor[V any] interface { ... }

// ❌ DON'T: Hardcode implementation
type Cache[V any] struct {
    lru *LRU[V] // Hardcoded!
}
```

### 6.9.2 Track Memory

```go
// ✅ DO: Track memory usage
func (l *LRU[V]) Add(key string, value *Item[V]) {
    l.currentMem += itemSize
    // ...
}

// ❌ DON'T: Ignore memory
func (l *LRU[V]) Add(key string, value *Item[V]) {
    // No memory tracking!
}
```

### 6.9.3 Log Evictions

```go
// ✅ DO: Log evictions for debugging
if l.logger != nil {
    l.logger.Printf("LRU evicted: key=%s", entry.key)
}

// ❌ DON'T: Silent evictions
func (l *LRU[V]) removeOldest() {
    // No logging!
}
```

### 6.9.4 Set Reasonable Limits

```go
// ✅ DO: Set reasonable defaults
EngineConfig{
    MaxMemory:       1024 * 1024 * 1024, // 1GB
    EvictorCapacity: 10000,
}

// ❌ DON'T: Unlimited
EngineConfig{
    MaxMemory:       0,      // Unlimited = dangerous!
    EvictorCapacity: 0,      // Unlimited = dangerous!
}
```

## 6.10 Ringkasan

Yang Sudah Kita Pelajari:

1. Mengapa Eviction Penting
- Memory terbatas
- Prevent OOM
- Keep performance

2. Konsep LRU
- Least Recently Used
- Map + Doubly Linked List
- O(1) operations

3. Implementasi LRU
- Add, Get, Remove
- Memory tracking
- Eviction logging

4. Integrasi dengan Cache
- Evictor interface
- Strategy pattern
- POLICY command

Perbandingan Sebelum vs Sesudah

| Aspek | Bab 5 | Bab 6 |
|-------|-------|-------|
| Storage | Unlimited | Limited |
| Memory Limit | ❌ No | ✅ MaxMemory |
| Capacity Limit | ❌ No | ✅ EvictorCapacity |
| Eviction | ❌ No | ✅ LRU |
| Policy Command | ❌ No | ✅ POLICY |
| Memory Tracking | ❌ No | ✅ currentMem |
| Eviction Logs | ❌ No | ✅ Logger |

Dengan LRU eviction, kita bisa mengelola memory dengan baik. Selanjutnya kita akan meningkatkan performa dengan sharding.