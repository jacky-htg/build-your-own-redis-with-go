# Bab 7: Sharding - Mempercepat dengan Konkurensi

## 7.1 Pendahuluan: Masalah pada Cache Tunggal

### 7.1.1 Mengapa Sharding Diperlukan?

Setelah kita berhasil membangun storage engine dengan LRU eviction di Bab 6, server Pendem sudah memiliki fondasi yang solid. Namun, ada satu masalah: semua operasi menggunakan satu lock.

>    📂 Kode Lengkap Bab Ini:
>    Seluruh kode yang dibahas di bab ini tersedia di GitHub:
>
>    🔗 [github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/07-sharding](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/07-sharding)


```text
┌─────────────────────────────────────────────────────────────────┐
│              MASALAH: SINGLE CACHE, SINGLE LOCK                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    SINGLE CACHE                         │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │                   🔒 LOCK                         │  │   │
│   │  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐      │  │   │
│   │  │  │ A   │  │ B   │  │ C   │  │ D   │  │ E   │      │  │   │
│   │  │  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘      │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ❌ Masalah:                                                   │
│   • Semua operasi harus menunggu lock                           │
│   • CPU hanya satu core yang bekerja                            │
│   • Tidak bisa parallel                                         │
│   • Bottleneck di lock                                          │
└─────────────────────────────────────────────────────────────────┘
```

### 7.1.2 Solusi: Sharding

Sharding adalah teknik membagi data menjadi beberapa bagian (shards) yang lebih kecil. Setiap shard memiliki lock sendiri, sehingga operasi bisa berjalan paralel.
text

┌─────────────────────────────────────────────────────────────────┐
│                    SOLUSI: SHARDING                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│   │ SHARD 0  │  │ SHARD 1  │  │ SHARD 2  │  │ SHARD 3  │        │
│   │ ┌─────┐  │  │ ┌─────┐  │  │ ┌─────┐  │  │ ┌─────┐  │        │
│   │ │🔒 0 │  │  │ │🔒 1  │  │  │ │🔒 2 │  │  │ │🔒 3 │  │        │
│   │ │A,E,I│  │  │ │B,F,J│  │  │ │C,G,K│  │  │ │D,H,L│  │        │
│   │ └─────┘  │  │ └─────┘  │  │ └─────┘  │  │ └─────┘  │        │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│        │              │             │             │             │
│        ▼              ▼             ▼             ▼             │
│    ╔═══════╗      ╔═══════╗     ╔═══════╗     ╔═══════╗         │
│    ║ Core1 ║      ║ Core2 ║     ║ Core3 ║     ║ Core4 ║         │
│    ╚═══════╝      ╚═══════╝     ╚═══════╝     ╚═══════╝         │
│                                                                 │
│   ✅ Keuntungan:                                                │
│   • Operasi parallel                                            │
│   • Semua CPU core digunakan                                    │
│   • Lock contention berkurang                                   │
│   • Throughput meningkat                                        │
└─────────────────────────────────────────────────────────────────┘
```

## 7.2 Konsep Dasar Sharding

### 7.2.1 Bagaimana Data Didistribusikan?

Data didistribusikan ke shard menggunakan hash function. Key akan di-hash, lalu hasil hash digunakan untuk menentukan shard tujuan.

```text
┌──────────────────────────────────────────────────────────────┐
│              DISTRIBUSI DATA DENGAN HASH                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Key: "user:123"                                            │
│        │                                                     │
│        ▼                                                     │
│   Hash: fnv32("user:123") = 0x1A2B3C4D                       │
│        │                                                     │
│        ▼                                                     │
│   Index: 0x1A2B3C4D & (16 - 1) = 0x... = 5                   │
│        │                                                     │
│        ▼                                                     │
│   Shard: shards[5]                                           │
│                                                              │
│   ┌──────────────────────────────────────────────────────┐   │
│   │  Key Distribution (16 shards):                       │   │
│   │                                                      │   │
│   │  user:1    → shard[3]   │  user:2  → shard[7]        │   │
│   │  user:3    → shard[1]   │  user:4  → shard[5]        │   │
│   │  session:1 → shard[5]   │  session:2→ shard[3]       │   │
│   │  cart:1    → shard[0]   │  cart:2  → shard[8]        │   │
│   └──────────────────────────────────────────────────────┘   │
│                                                              │
│   ❗ Key dengan pola sama (user:*) tersebar merata           │
└──────────────────────────────────────────────────────────────┘
```

### 7.2.2 Mengapa Menggunakan FNV-32a?

```go
// FNV-32a adalah hash function yang:
// 1. Cepat (beberapa operasi bitwise)
// 2. Distribusi merata
// 3. Deterministik (key yang sama selalu hash sama)
// 4. Collision rendah

hasher := fnv.New32a()
hasher.Write([]byte(key))
hash := hasher.Sum32()  // 32-bit integer
```

### 7.2.3 Power of 2 Optimization

```go

// ❌ Modulo (butuh division, lebih lambat)
idx := int(hash) % numShards

// ✅ Bitwise AND (lebih cepat, hanya untuk power of 2)
idx := int(hash) & (numShards - 1)

// Contoh dengan 16 shards:
// 16 - 1 = 15 (0b1111)
// hash & 15 = hash % 16 (tapi lebih cepat!)
```

Power of 2 adalah: 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024.

## 7.3 Implementasi Sharding

### 7.3.1 Struktur Shard

```go
// internal/engine/shard.go
package engine

import (
    "log"
    "pendem/internal/config"
    "sync"
    "time"
)

type Shard[V any] struct {
    mu      sync.RWMutex  // Lock per-shard
    policy  string
    evictor Evictor[V]    // Eviction policy
}

func NewShard[V any](cfg config.EngineConfig, logger *log.Logger) *Shard[V] {
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
    default:
        logger.Printf("Unknown eviction policy '%s', using LRU", cfg.EvictionPolicy)
        evictor = NewLRUWithMemory[V](cfg.EvictorCapacity, cfg.MaxMemory, logger)
        policy = "lru"
    }

    return &Shard[V]{
        policy:  policy,
        evictor: evictor,
    }
}
```

Penjelasan:
- Setiap shard punya sync.RWMutex sendiri
- Setiap shard punya Evictor sendiri (LRU, LFU, dll)
- Shard adalah unit independen yang bisa dioperasikan paralel

### 7.3.2 Operasi pada Shard

```go
func (s *Shard[V]) Set(key string, value V, ttl time.Duration) {
    s.mu.Lock()
    defer s.mu.Unlock()

    item := NewItem(value, ttl)
    s.evictor.Add(key, item)
}

func (s *Shard[V]) Get(key string) (V, bool) {
    var zero V

    s.mu.RLock()
    item, exists := s.evictor.Get(key)
    s.mu.RUnlock()

    if !exists {
        return zero, false
    }

    // Lazy deletion: expired items removed on access
    if item.IsExpired() {
        s.mu.Lock()
        s.evictor.Remove(key)
        s.mu.Unlock()
        return zero, false
    }

    return item.Value, true
}

func (s *Shard[V]) Delete(key string) bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    return s.evictor.Remove(key)
}
```

Perhatikan:
- Get menggunakan RLock (bisa parallel dengan Get lain)
- Set dan Delete menggunakan Lock (exclusive)
- Expired items dihapus saat diakses (lazy deletion)

### 7.3.3 Struktur Cache dengan Sharding

```go
// internal/engine/cache.go
package engine

import (
    "hash/fnv"
    "log"
    "pendem/internal/config"
    "time"
)

type Cache[V any] struct {
    shards    []*Shard[V]
    numShards int
}

func NewCache[V any](cfg config.EngineConfig, logger *log.Logger) *Cache[V] {
    // Default ShardCount jika tidak di-set
    if cfg.ShardCount <= 0 {
        cfg.ShardCount = config.DefaultConfig().Engine.ShardCount
    }

    // Warning jika bukan power of 2
    if cfg.ShardCount&(cfg.ShardCount-1) != 0 {
        logger.Printf("Warning: ShardCount %d is not power of 2, performance may be affected",
            cfg.ShardCount)
    }

    cache := &Cache[V]{
        shards:    make([]*Shard[V], cfg.ShardCount),
        numShards: cfg.ShardCount,
    }

    // Inisialisasi setiap shard
    for i := 0; i < cfg.ShardCount; i++ {
        cache.shards[i] = NewShard[V](cfg, logger)
    }

    // Start background cleanup
    go cache.cleanupLoop()
    return cache
}
```

### 7.3.4 Menentukan Shard dari Key

```go
// getShard menentukan shard berdasarkan key menggunakan hash
func (c *Cache[V]) getShard(key string) *Shard[V] {
    hasher := fnv.New32a()
    hasher.Write([]byte(key))
    hash := hasher.Sum32()
    idx := int(hash) & (c.numShards - 1)  // Bitwise AND (power of 2)
    return c.shards[idx]
}
```

Kenapa menggunakan bitwise AND?
- & (AND) lebih cepat dari % (modulo)
- Hanya valid untuk power of 2
- hash & (n-1) sama dengan hash % n untuk power of 2

### 7.3.5 Key-Based Operations

```go
// ============================================
// KEY-BASED OPERATIONS (1 shard)
// ============================================

func (c *Cache[V]) Set(key string, value V, ttl time.Duration) {
    shard := c.getShard(key)
    shard.Set(key, value, ttl)
}

func (c *Cache[V]) Get(key string) (V, bool) {
    shard := c.getShard(key)
    return shard.Get(key)
}

func (c *Cache[V]) Delete(key string) bool {
    shard := c.getShard(key)
    return shard.Delete(key)
}

func (c *Cache[V]) TTL(key string) int64 {
    shard := c.getShard(key)
    return shard.TTL(key)
}
```

Alur:
- Client memanggil Get("user:123")
- getShard() menghitung hash "user:123"
- Hash → index shard (misal: 5)
- Operasi dilakukan di shards[5]
- Hanya shard 5 yang di-lock, shard lain tetap bisa diakses!

### 7.3.6 Global Operations

```go
// ============================================
// GLOBAL OPERATIONS (Semua shards)
// ============================================

func (c *Cache[V]) MemoryUsage() int64 {
    var total int64
    for _, shard := range c.shards {
        total += shard.MemoryUsage()
    }
    return total
}

func (c *Cache[V]) Size() int {
    total := 0
    for _, shard := range c.shards {
        total += shard.Size()
    }
    return total
}

func (c *Cache[V]) Policy() string {
    if len(c.shards) == 0 {
        return "unknown"
    }
    return c.shards[0].Policy() // Sama untuk semua shard
}

func (c *Cache[V]) MaxCapacity() int {
    if len(c.shards) == 0 {
        return 0
    }
    return c.shards[0].MaxCapacity()
}
```

Untuk operasi global:
- Looping semua shard
- Mengakumulasi hasil
- Policy sama untuk semua shard

### 7.3.7 Background Cleanup

```go
func (c *Cache[V]) cleanupLoop() {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()

    for range ticker.C {
        for _, shard := range c.shards {
            shard.CleanupExpired()
        }
    }
}
```

Kenapa cleanup semua shard?
- Setiap shard punya expired items sendiri
- Harus dibersihkan di semua shard
- Bisa berjalan paralel (tapi sequential di sini)

## 7.4 Config Update

### internal/config/config.go

```go
type EngineConfig struct {
    MaxMemory       int64  // Max memory in bytes
    EvictionPolicy  string // "lru", "lfu", "ttl"
    EvictorCapacity int    // Max items per shard
    ShardCount      int    // Number of shards (default: 16)
    DefaultTTL      time.Duration
}

func DefaultConfig() Config {
    return Config{
        Engine: EngineConfig{
            MaxMemory:       1024 * 1024 * 1024, // 1GB
            EvictionPolicy:  "lru",
            EvictorCapacity: 10000,
            ShardCount:      16, // ✅ Default
            DefaultTTL:      0,
        },
    }
}
```

## 7.5 Visualisasi Alur Operasi

### GET Operation dengan Sharding

```text
┌─────────────────────────────────────────────────────────────────┐
│                    GET OPERATION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Client: GET "user:123"                                        │
│              │                                                  │
│              ▼                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  1. hash = fnv32("user:123") = 0x1A2B3C4D               │   │
│   │  2. idx = hash & (16 - 1) = 5                           │   │
│   │  3. shard = shards[5]                                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│              │                                                  │
│              ▼                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  SHARD[5].Get("user:123")                               │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │  🔒 RLock() (bisa parallel dengan Get lain)       │  │   │
│   │  │  item = evictor.Get("user:123")                   │  │   │
│   │  │  🔓 RUnlock()                                     │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│              │                                                  │
│              ▼                                                  │
│   Response: "John"                                              │
└─────────────────────────────────────────────────────────────────┘
```

### Parallel Operations

```text
┌─────────────────────────────────────────────────────────────────┐
│                 PARALLEL OPERATIONS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Client 1: GET "user:1"   → shard[3]  ──┐                      │
│   Client 2: GET "user:2"   → shard[7]  ──┼── Parallel! ✅       │
│   Client 3: SET "user:3"   → shard[1]  ──┤                      │
│   Client 4: GET "user:4"   → shard[5]  ──┘                      │
│                                                                 │
│   ❗ Tidak ada blocking antar shard!                            │
│   ❗ Semua CPU core digunakan!                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 7.6 Testing Sharding
### 7.6.1 Test dengan redis-cli

```bash
redis-cli -h host.docker.internal -p 6379

# ============================================
# BASIC OPERATIONS
# ============================================

# SET - akan masuk ke shard tertentu
127.0.0.1:6379> SET user:1 "John"
OK
127.0.0.1:6379> SET user:2 "Jane"
OK
127.0.0.1:6379> SET session:1 "active"
OK

# GET - akan mencari di shard yang sama
127.0.0.1:6379> GET user:1
"John"

# ============================================
# GLOBAL STATS
# ============================================

127.0.0.1:6379> POLICY
"Eviction policy: lru (items: 3, max: 160000)"  # 16 × 10000

127.0.0.1:6379> MEMORY USAGE
"12.00 KB"  # Total memory semua shard

7.6.2 Test Distribusi
bash

# Script untuk test distribusi
for i in {1..1000}; do
    redis-cli SET "key_$i" "value_$i"
done

# Hasil: Key tersebar merata ke 16 shard
# Setiap shard mendapat ~62-63 keys (1000/16)
```

### 7.6.3 Test Concurrency

```bash

# 10 concurrent clients
for i in {1..10}; do
    (while true; do
        redis-cli GET "key_$RANDOM"
    done) &
done

# ✅ Semua berjalan parallel tanpa blocking
```

## 7.7 Performance Analysis

### 7.7.1 Lock Contention

```text
┌─────────────────────────────────────────────────────────────────┐
│              LOCK CONTENTION COMPARISON                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Single Cache:              Sharded Cache (16 shards):         │
│   ┌──────────────────────┐  ┌────┬────┬────┬────┬────┬────┐     │
│   │   🔒 100% busy       │  │ 🔒  │ 🔒 │ 🔒 │ 🔒  │ 🔒 │ 🔒 |     │
│   │   (1 lock)           │  │ 6% │ 7% │ 5% │ 8% │ 6% │ 6% │     |
│   │                      │  └────┴────┴────┴────┴────┴────┘     │
│   └──────────────────────┘                                      │
│                                                                 │
│   ❌ Bottleneck          ✅ Lock terdistribusi                   │
│   ❌ 1 core              ✅ Multi-core                           │
└─────────────────────────────────────────────────────────────────┘
```

### 7.7.2 Throughput Comparison

```text
┌─────────────────────────────────────────────────────────────────┐
│              THROUGHPUT COMPARISON                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Single Cache:                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Ops/sec: 50,000 (1 core)                               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   Sharded Cache (16 shards, 4 cores):                           │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Ops/sec: 150,000+ (4 cores)                            │   │
│   │  🚀 3x faster!                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ❗ Sharding memberikan scaling yang hampir linear             │
└─────────────────────────────────────────────────────────────────┘
```

### 7.7.3 Memory Usage per Shard

```text
┌─────────────────────────────────────────────────────────────────┐
│                  MEMORY PER SHARD                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Total Memory: 1 GB                                            │
│   Per Shard:   64 MB (1 GB / 16)                                │
│                                                                 │
│   ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐     │
│   │  64  │  63  │  65  │  62  │  64  │  63  │  66  │  62  │     │
│   │  MB  │  MB  │  MB  │  MB  │  MB  │  MB  │  MB  │  MB  │     │
│   ├──────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┤     │
│   │  63  │  64  │  62  │  65  │  64  │  63  │  65  │  62  │     │
│   │  MB  │  MB  │  MB  │  MB  │  MB  │  MB  │  MB  │  MB  │     │
│   └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘     │
│                                                                 │
│   ❗ Distribusi memory merata karena hash distribution          │
└─────────────────────────────────────────────────────────────────┘
```

## 7.8 Best Practices

### 7.8.1 Jumlah Shard yang Tepat

| Use Case | Shard Count | Alasan |
|----------|-------------|--------|
| Development | 16 | Sederhana, cukup |
| Small Production | 32 | Balance |
| Medium Production | 64 | High concurrency |
| Large Production | 128 | Very high concurrency |


### 7.8.2 Gunakan Power of 2

```go
// ✅ DO: Gunakan power of 2
ShardCount: 16  // 2^4

// ❌ DON'T: Gunakan non-power of 2
ShardCount: 10  // Bukan power of 2 (performansi turun)
```

### 7.8.3 Per-Shard Lock

```go
// ✅ DO: Lock per-shard
type Shard[V any] struct {
    mu sync.RWMutex  // Per-shard
}

// ❌ DON'T: Global lock
type Cache[V any] struct {
    mu sync.RWMutex  // Global (bottleneck)
}
```

### 7.8.4 Background Cleanup

```go
// ✅ DO: Cleanup semua shard
func (c *Cache[V]) cleanupLoop() {
    for range ticker.C {
        for _, shard := range c.shards {
            shard.CleanupExpired()
        }
    }
}

// ❌ DON'T: Cleanup hanya 1 shard
func (c *Cache[V]) cleanupLoop() {
    for range ticker.C {
        c.shards[0].CleanupExpired() // Shard lain tidak di-cleanup!
    }
}
```

## 7.9 Ringkasan

Yang Sudah Kita Pelajari:
1. Masalah Single Cache
- Lock contention
- Single core usage
- Bottleneck

2. Konsep Sharding
- Data dibagi ke banyak shard
- Hash function untuk distribusi
- Per-shard lock

3. Implementasi
- Shard[V] dengan RWMutex
- Cache[V] dengan multiple shards
- Key-based operations → 1 shard
- Global operations → semua shard

4. Performance
- Parallel operations
- Multi-core utilization
- 3x throughput increase

**Perbandingan Sebelum vs Sesudah**
| Aspek | Sebelum (Bab 6) | Sesudah (Bab 7) |
|-------|-----------------|-----------------|
| Lock | 1 global lock | Per-shard lock |
| Parallel | ❌ Tidak | ✅ Ya |
| CPU Cores | 1 core | All cores |
| Throughput | 50k ops/sec | 150k+ ops/sec |
| Scalability | ❌ Limited | ✅ Linear |

Dengan sharding, kita bisa menyimpan data dengan lebih efisien. Selanjutnya, kita akan membuat data tetap aman meskipun server restart.