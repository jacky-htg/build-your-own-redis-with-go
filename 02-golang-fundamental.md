# Bab 2: Golang Fundamental

## 2.1 Pendahuluan

Sebelum kita memulai perjalanan membangun Pendem, ada baiknya kita memahami teknik-teknik Go yang akan sering digunakan. Bab ini bukan pengenalan Go dari dasar, melainkan **panduan praktis** tentang konsep-konsep yang menjadi tulang punggung project ini.

### 2.1.1 Prasyarat

Pembaca diharapkan sudah memahami dasar-dasar Go. Jika belum, silakan pelajari terlebih dahulu:
- [Basic Golang](https://golang-microservices.rijalasepnugroho.com/golang-fundamental/01-basic) — tipe data, variabel, fungsi, flow control, array, slice, map.
- [Struktur Data](https://golang-microservices.rijalasepnugroho.com/golang-fundamental/02-struktur-data) — struct, method, interface, object composition.
- [Konkurensi](https://golang-microservices.rijalasepnugroho.com/golang-fundamental/03-konkurensi) — goroutine, channel, sync.Mutex, sync.WaitGroup, errgroup, context.

Bab ini akan fokus pada **penerapan praktis** dari konsep-konsep tersebut di dalam kode Pendem.

## 2.2 Map: Fondasi Penyimpanan Data

### 2.2.1 Mengapa Map?

Map adalah struktur data yang paling penting di Pendem. Semua data cache disimpan dalam map.

```go

// internal/engine/cache.go
type Cache[V any] struct {
    items map[string]*Item[V]  // ← Map untuk menyimpan data
    mu    sync.RWMutex
}
```

**Karakteristik Map di Go:**
- O(1) untuk operasi get dan set.
- Tidak aman untuk konkurensi — harus dilindungi dengan mutex.
- Key harus comparable — string, int, dll.

### 2.2.2 Operasi Dasar Map

```go

// Membuat map
items := make(map[string]*Item[string])

// Menyimpan data
items["user:1"] = NewItem("John", 0)

// Mengambil data
item, exists := items["user:1"]
if !exists {
    // Key tidak ditemukan
}

// Menghapus data
delete(items, "user:1")

// Iterasi
for key, item := range items {
    fmt.Printf("%s: %v\n", key, item.Value)
}
```

### 2.2.3 Map di Pendem

Di Pendem, map digunakan di berbagai tempat:

```go

// internal/engine/hash.go
type Hash struct {
    fields map[string]string  // ← Hash fields
    mu     sync.RWMutex
}

// internal/engine/set.go
type Set struct {
    members map[string]struct{}  // ← Set members (struct{} untuk hemat memory)
    mu      sync.RWMutex
}

// internal/engine/lru.go
type LRU[V any] struct {
    items map[string]*list.Element  // ← Map untuk O(1) lookup
    // ...
}
```

**Tips:** Gunakan `map[string]struct{}` untuk set karena `struct{}` tidak memakan memory.

## 2.3 Type Parameter (Generics)

### 2.3.1 Mengapa Generics?

Generics memungkinkan kita menulis kode yang **reusable** untuk berbagai tipe data.

```go

// internal/engine/item.go
type Item[T any] struct {
    Value      T      // ← T adalah generic type
    Expiration int64
}

func NewItem[T any](value T, ttl time.Duration) *Item[T] {
    return &Item[T]{
        Value:      value,
        Expiration: exp,
    }
}
```

### 2.3.2 Cara Menggunakan Generics

Untuk menggunakan fungsi generic, kita bisa:

**1. Biarkan Go menginfer tipe:**

```go

// Go akan infer bahwa T adalah string
item := NewItem("John", 0)
// item bertipe *Item[string]
```

**2. Eksplisit dengan type parameter:**

```go

// Eksplisit menyebutkan tipe
item := NewItem[string]("John", 0)

// Untuk tipe lain
intItem := NewItem[int](100, 0)
boolItem := NewItem[bool](true, 0)
```

**3. Dengan variabel:**

```go

var cache *Cache[string] = NewCache[string]()
cache.Set("key", "value", 0)

// Atau dengan inferensi
cache := NewCache[string]()  // Tipe dieksplisitkan di sini
```

### 2.3.3 Generic di Pendem

Pendem menggunakan generics di banyak tempat:

```go

// internal/engine/cache.go
type Cache[V any] struct {
    items map[string]*Item[V]  // ← Cache bisa menyimpan tipe apapun
    mu    sync.RWMutex
}

func NewCache[V any]() *Cache[V] {
    return &Cache[V]{
        items: make(map[string]*Item[V]),
    }
}

// internal/engine/lru.go
type LRU[V any] struct {
    // ...
}

// internal/engine/shard.go
type Shard[V any] struct {
    // ...
}
```

**Keuntungan:**
- ✅ **Type safety** — compiler menjamin tipe data konsisten.
- ✅ **Reusability** — satu kode untuk semua tipe.
- ✅ **Performance** — tidak perlu casting (type assertion).

## 2.4 container/list: Doubly Linked List untuk LRU

### 2.4.1 Mengapa container/list?

LRU (Least Recently Used) membutuhkan struktur data yang bisa:
1. Menambahkan elemen di depan (O(1)).
2. Memindahkan elemen ke depan (O(1)).
3. Menghapus elemen dari belakang (O(1)).

`container/list` adalah **doubly linked list** yang menyediakan semua operasi tersebut.

```go

// internal/engine/lru.go
import "container/list"

type LRU[V any] struct {
    items map[string]*list.Element  // Map untuk O(1) lookup
    order *list.List                // List untuk tracking order
}

type lruEntry[V any] struct {
    key   string
    value *Item[V]
}
```

### 2.4.2 Operasi container/list

```go

// Membuat list
order := list.New()

// Menambahkan di depan
elem := order.PushFront(&lruEntry[string]{key: "user:1", value: item})

// Memindahkan ke depan
order.MoveToFront(elem)

// Menghapus dari belakang (LRU)
oldest := order.Back()
if oldest != nil {
    order.Remove(oldest)
    entry := oldest.Value.(*lruEntry[string])
    delete(items, entry.key)
}

// Iterasi
for elem := order.Front(); elem != nil; elem = elem.Next() {
    entry := elem.Value.(*lruEntry[string])
    fmt.Println(entry.key)
}
```

### 2.4.3 LRU dengan container/list

```go

func (l *LRU[V]) Add(key string, value *Item[V]) {
    // Jika key sudah ada, update dan pindahkan ke depan
    if elem, exists := l.items[key]; exists {
        l.order.MoveToFront(elem)
        elem.Value.(*lruEntry[V]).value = value
        return
    }

    // Jika penuh, hapus yang paling belakang
    if l.order.Len() >= l.capacity {
        l.removeOldest()
    }

    // Tambahkan di depan
    elem := l.order.PushFront(&lruEntry[V]{key: key, value: value})
    l.items[key] = elem
}
```

## 2.5 sync: Concurrency Control

### 2.5.1 Mutex vs RWMutex

Keduanya digunakan untuk mengamankan akses ke data bersama, tapi dengan karakteristik berbeda:

**sync.Mutex — Mutual Exclusion**

```go

type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Inc() {
    c.mu.Lock()         // ← Hanya satu goroutine yang bisa akses
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()         // ← Bahkan untuk baca, harus lock
    defer c.mu.Unlock()
    return c.value
}
```

**Karakteristik:**
- ✅ Sederhana — hanya Lock() dan Unlock()
- ❌ **Semua operasi (baca & tulis) saling blocking**
- ❌ Performa lebih rendah untuk banyak operasi baca

**sync.RWMutex — Reader/Writer Mutual Exclusion**

```go

type Counter struct {
    mu    sync.RWMutex
    value int
}

func (c *Counter) Inc() {
    c.mu.Lock()          // ← Write lock (exclusive)
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.RLock()         // ← Read lock (boleh banyak)
    defer c.mu.RUnlock()
    return c.value
}
```

**Karakteristik:**
- ✅ **Multiple readers** boleh akses bersamaan
- ✅ **Writers exclusive** — hanya satu writer
- ✅ Performa lebih baik untuk **banyak baca, sedikit tulis**
- ⚠️ Sedikit lebih kompleks (RLock/RUnlock)

**Perbandingan Visual**

```text

┌─────────────────────────────────────────────────────────────────┐
│              MUTEX vs RWMUTEX                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ❌ sync.Mutex:                                                │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Reader 1: ████████████████░░░░░░░░░░░░░░░░░░░░░░░░     │   │
│   │  Reader 2: ░░░░░░░████████████████░░░░░░░░░░░░░░░░      │   │
│   │  Writer:   ░░░░░░░░░░░░░░░░░░░░░░███████████████        │   │
│   │  Reader 3: ░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████        │   │
│   │  ❗ Semua saling tunggu (termasuk baca)                 │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ✅ sync.RWMutex:                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Reader 1: ████████████████████████████░░░░░░░░░░░      │   │
│   │  Reader 2: ████████████████████████████░░░░░░░░░░░      │   │
│   │  Writer:   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████       │   │
│   │  Reader 3: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░███████       │   │
│   │  ❗ Reader bisa jalan bersamaan!                        │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Kapan Pakai Apa?**

| Skenario | Rekomendasi |
|----------|-------------|
| Banyak baca, sedikit tulis | ✅ RWMutex (cache, config) |
| Banyak tulis | ⚠️ Mutex (lebih simpel) |
| Sederhana | ✅ Mutex |
| Performance critical | ✅ RWMutex untuk read-heavy |

### 2.5.1 sync.RWMutex

`sync.RWMutex` digunakan untuk mengamankan akses ke data bersama.

```go

// internal/engine/shard.go
type Shard[V any] struct {
    mu      sync.RWMutex  // ← Proteksi akses concurrent
    evictor Evictor[V]
}

func (s *Shard[V]) Get(key string) (V, bool) {
    s.mu.RLock()         // ← Read lock (boleh banyak)
    defer s.mu.RUnlock()
    // ... baca data
}

func (s *Shard[V]) Set(key string, value V, ttl time.Duration) {
    s.mu.Lock()          // ← Write lock (hanya satu)
    defer s.mu.Unlock()
    // ... tulis data
}
```

**Mengapa RWMutex di Shard?**
- Cache didominasi oleh operasi baca (GET, EXISTS, TTL)
- Hanya sesekali operasi tulis (SET, DEL)
- RWMutex memungkinkan banyak GET berjalan paralel
    
**Kapan pakai RLock vs Lock:**
- RLock — untuk operasi baca (GET, EXISTS, TTL).
- Lock — untuk operasi tulis (SET, DEL, HSET, dll).

### 2.5.2 sync.WaitGroup

`sync.WaitGroup` digunakan untuk menunggu goroutine selesai.

```go

// internal/server/server.go
func (s *Server) ShutdownWithTimeout(timeout time.Duration) error {
    close(s.quit)
    s.listener.Close()

    done := make(chan struct{})
    go func() {
        s.wg.Wait()  // ← Tunggu semua goroutine selesai
        close(done)
    }()

    select {
    case <-done:
        return nil
    case <-time.After(timeout):
        return fmt.Errorf("shutdown timeout")
    }
}

func (s *Server) handleConnection(conn net.Conn) {
    s.wg.Add(1)          // ← Tambah counter
    defer s.wg.Done()    // ← Kurangi counter saat selesai
    // ... handle connection
}
```

## 2.6 Goroutine: Concurrency di Pendem

### 2.6.1 Goroutine untuk Setiap Koneksi

Setiap koneksi client di-handle oleh goroutine terpisah:

```go

// internal/server/server.go
func (s *Server) Start() error {
    for {
        conn, err := s.listener.Accept()
        if err != nil {
            continue
        }
        go s.handleConnection(conn)  // ← Setiap koneksi = 1 goroutine
    }
}
```

### 2.6.2 Goroutine untuk Background Task

```go

// internal/engine/cache.go
func NewCache[V any]() *Cache[V] {
    cache := &Cache[V]{items: make(map[string]*Item[V])}
    go cache.cleanupLoop()  // ← Background cleanup
    return cache
}

func (c *Cache[V]) cleanupLoop() {
    ticker := time.NewTicker(5 * time.Minute)
    for range ticker.C {
        c.cleanupExpired()
    }
}
```

### 2.6.3 Goroutine untuk Idle Monitor

```go

// internal/server/connection.go
func (c *Connection) MonitorIdle() {
    ticker := time.NewTicker(1 * time.Second)
    for {
        select {
        case <-ticker.C:
            if c.IsIdle() {
                c.Close()
                return
            }
        case <-c.done:
            return
        }
    }
}

// Dipanggil di handleConnection
go c.MonitorIdle()  // ← Background idle monitor
```

## 2.7 Channel: Komunikasi Antar Goroutine

### 2.7.1 Channel untuk Shutdown Signal

```go

// internal/server/server.go
type Server struct {
    quit chan struct{}  // ← Channel untuk signal shutdown
}

func (s *Server) Shutdown() {
    close(s.quit)  // ← Kirim signal ke semua goroutine
}

func (s *Server) handleConnection(conn net.Conn) {
    for {
        select {
        case <-s.quit:  // ← Terima signal shutdown
            return
        default:
            // ... proses command
        }
    }
}
```

### 2.7.2 Channel untuk Activity Signal

```go

// internal/server/connection.go
func (c *Connection) MonitorIdle() {
    idleTimer := time.NewTimer(c.idleTimeout)
    activity := make(chan bool, 1)  // ← Buffered channel

    go func() {
        for {
            select {
            case <-idleTimer.C:
                c.Close()
                return
            case <-activity:  // ← Reset idle timer
                idleTimer.Reset(c.idleTimeout)
            }
        }
    }()
}
```

### 2.7.3 Channel untuk Read Result

```go

// internal/server/server.go
type readResult struct {
    respVal *RESPValue
    err     error
}
readCh := make(chan readResult, 1)

go func() {
    respVal, err := parser.Read()
    readCh <- readResult{respVal, err}
}()

select {
case <-shutdown:
    return
case res := <-readCh:  // ← Terima hasil baca
    // ... proses response
}
```

## 2.8 Interface: Abstraksi di Pendem

### 2.8.1 Evictor Interface

```go

// internal/engine/evictor.go
type Evictor[V any] interface {
    Add(key string, value *Item[V])
    Get(key string) (*Item[V], bool)
    Has(key string) bool
    Remove(key string) bool
    Len() int
    MaxCapacity() int
    MemoryUsage() int64
    ForEach(fn func(key string, item *Item[V]) bool)
    GetItems() map[string]Item[V]
}
```

**Kenapa pakai interface?**
- **Flexibility** — bisa ganti eviction policy (LRU, LFU, TTL).
- **Testability** — mudah di-mock untuk testing.
- **Separation of concerns** — Cache tidak bergantung pada implementasi spesifik.

### 8.2.2 CommandHandler Type

```go

// internal/server/server.go
type CommandHandler func(args []string) RESPValue

type Server struct {
    handlers map[string]CommandHandler  // ← Map of handlers
}

func (s *Server) RegisterHandler(cmd string, handler CommandHandler) {
    s.handlers[strings.ToUpper(cmd)] = handler
}
```

### 8.2.3 Interface Kosong (any)

```go

// internal/engine/value.go
type Value struct {
    Type ValueType
    Data any  // ← any = interface{}
    TTL  time.Duration
    Exp  int64
}
```

Catatan: `any` adalah alias untuk `interface{}` (Go 1.18+). Gunakan dengan hati-hati karena kehilangan type safety.

## 2.9 Struct: Organisasi Data

### 2.9.1 Struct untuk Konfigurasi

```go

// internal/config/config.go
type ServerConfig struct {
    MaxConnections int
    ReadTimeout    time.Duration
    WriteTimeout   time.Duration
    IdleTimeout    time.Duration
}

type EngineConfig struct {
    MaxMemory       int64
    EvictionPolicy  string
    EvictorCapacity int
    ShardCount      int
    DefaultTTL      time.Duration
}

type Config struct {
    Server ServerConfig
    Engine EngineConfig
}
```

### 2.9.2 Struct dengan Embedded Fields

```go

// internal/server/connection.go
type Connection struct {
    conn         net.Conn
    remoteAddr   string
    idleTimeout  time.Duration
    logger       *log.Logger
    done         chan struct{}
    mu           sync.Mutex  // ← Embedded field
    closed       bool
}
```

### 2.9.3 Object Composition

Pendem menggunakan **composition** bukan inheritance:

```go

// internal/engine/shard.go
type Shard[V any] struct {
    mu      sync.RWMutex
    policy  string
    evictor Evictor[V]  // ← Composition: Shard "has a" Evictor
    hashes  map[string]*Hash
    lists   map[string]*List
    sets    map[string]*Set
    sortedSets map[string]*SortedSet
}
```

## 2.10 Error Handling

### 2.10.1 Custom Error Types

```go

// internal/server/resp.go
var (
    ErrInvalidRESP  = errors.New("invalid RESP format")
    ErrRESPTooDeep  = errors.New("RESP nesting too deep")
    ErrRESPTooLarge = errors.New("RESP value too large")
)
```

### 2.10.2 Error Wrapping

```go

func (p *RESPParser) readBulkString() (*RESPValue, error) {
    if length > MaxBulkStringSize {
        return nil, fmt.Errorf("%w: bulk string too large (%d bytes)",
            ErrRESPTooLarge, length)
    }
    // ...
}
```

### 2.10.3 Defer untuk Cleanup

```go

func (s *Server) handleConnection(conn net.Conn) {
    defer func() {
        conn.Close()
        atomic.AddInt32(&s.activeConn, -1)
    }()
    // ...
}
```

## 2.11 Ringkasan

| Konsep | Penggunaan di Pendem |
|--------|----------------------|
| Map | Penyimpanan data cache, hash fields, set members |
| Generics | Cache, Item, LRU, Shard (reusable untuk semua tipe) |
| container/list | LRU eviction (doubly linked list) |
| sync.RWMutex | Proteksi akses concurrent ke map dan shard |
| sync.WaitGroup | Graceful shutdown, tracking goroutine |
| Goroutine | Setiap koneksi, background cleanup, idle monitor |
| Channel | Shutdown signal, activity signal, read result |
| Interface | Evictor abstraction, CommandHandler |
| Struct | Config, Connection, Item, Hash, List, Set, SortedSet |
| Error Handling | Custom errors, error wrapping, defer cleanup |

Dengan pemahaman tentang teknik-teknik Go di atas, kita siap membangun cache server. Bab berikutnya kita mulai dengan membahas socket programming.