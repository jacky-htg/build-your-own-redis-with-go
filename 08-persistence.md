# Bab 8: Persistence - Menjaga Data Tetap Aman

## 8.1 Pendahuluan: Mengapa Data Perlu Disimpan?

Setelah kita berhasil membangun cache server dengan sharding dan LRU eviction, ada satu masalah besar: data hilang saat server restart!

>    📂 Kode Lengkap Bab Ini:
>    Seluruh kode yang dibahas di bab ini tersedia di GitHub:
>
>    🔗 [github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/08-persistence](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/08-persistence)


### 8.1.1 Masalah: Data Volatile

```text
┌─────────────────────────────────────────────────────────────────┐
│              MASALAH: DATA HILANG SAAT RESTART                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Server Running:                                               │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  SET user:1 "John"  ✅                                  │   │
│   │  SET user:2 "Jane"  ✅                                  │   │
│   │  GET user:1         "John"  ✅                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   💥 Server Crash / Restart                                     │
│                                                                 │
│   Server Restart:                                               │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  GET user:1         (nil)  ❌ Data hilang!              │   │
│   │  GET user:2         (nil)  ❌ Data hilang!              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ❌ Semua data di memory hilang!                               │
└─────────────────────────────────────────────────────────────────┘
```

### 8.1.2 Solusi: Persistence

Persistence adalah mekanisme untuk menyimpan data ke disk agar tetap aman meskipun server restart.

```text
┌─────────────────────────────────────────────────────────────────┐
│                    SOLUSI: PERSISTENCE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Memory (RAM)          →    Disk (Persistent)                  │
│   ┌─────────────────┐         ┌─────────────────────────────┐   │
│   │  user:1 "John"  │  ────▶  │  pendem.aof                 │   │
│   │  user:2 "Jane"  │         │  SET user:1 "John"          │   │
│   │  user:3 "Jack"  │         │  SET user:2 "Jane"          │   │
│   └─────────────────┘         │  SET user:3 "Jack"          │   │
│                               └─────────────────────────────┘   │
│                                                                 │
│   ✅ Data tetap aman walau server restart!                      │
└─────────────────────────────────────────────────────────────────┘
```

## 8.2 Strategi Persistence

Ada dua strategi utama untuk persistence di cache server:

### 8.2.1 Snapshot (Full Backup)

Snapshot adalah menyimpan seluruh data ke disk secara periodik.

```text
┌─────────────────────────────────────────────────────────────────┐
│              SNAPSHOT STRATEGY                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ⏰ Every 1 Hour: Save ALL data to disk                        │
│                                                                 │
│   10:00 ──── 11:00 ──── 12:00 ──── 13:00                        │
│    │          │          │          │                           │
│    🏺         🏺         🏺         🏺                           │
│  Snapshot   Snapshot   Snapshot   Snapshot                      │
│  (Full)     (Full)     (Full)     (Full)                        │
│                                                                 │
│   ✅ Keuntungan: Recovery cepat                                 │
│   ❌ Kekurangan: Data loss up to 1 jam                          │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2.2 AOF (Append Only File)

AOF adalah mencatat setiap operasi write ke disk secara real-time.

```text
┌─────────────────────────────────────────────────────────────────┐
│              AOF STRATEGY                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Every write operation → Append to file                        │
│                                                                 │
│   10:00 ──── 10:05 ──── 10:10 ──── 10:15                        │
│    │          │          │          │                           │
│    📝         📝         📝         📝                           │
│   SET user:1 SET user:2 DEL user:1 SET user:3                   │
│                                                                 │
│   ✅ Keuntungan: Data loss minimal                              │
│   ❌ Kekurangan: Recovery lebih lambat                          │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2.3 Kombinasi Snapshot + AOF (Best of Both)

```text
┌─────────────────────────────────────────────────────────────────┐
│              KOMBINASI SNAPSHOT + AOF                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   10:00 ──── 10:05 ──── 10:10 ──── 10:15 ──── 10:20             │
│    │          │          │          │          │                │
│    🏺         📝         📝         📝         📝                 │
│  Snapshot   AOF        AOF        AOF        AOF                │
│  (Full)     (Log)      (Log)      (Log)      (Log)              │
│                                                                 │
│   Recovery: Snapshot 10:00 + AOF 10:05 → 10:20                  │
│   Data Loss: Max 5 menit (hanya AOF yang belum di-flush)        │
└─────────────────────────────────────────────────────────────────┘
```

Pendem menggunakan kombinasi Snapshot + AOF dengan tiga format:
1. JSON Snapshot - Human readable (untuk debugging)
2. RDB - Binary format (cepat, compact)
3. AOF - Incremental log (data safety)

## 8.3 Desain Persistence

### 8.3.1 Arsitektur

```text
┌─────────────────────────────────────────────────────────────────┐
│              PERSISTENCE ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Cache Engine                         │   │
│   │  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐     │   │
│   │  │ S0   │ S1   │ S2   │ S3   │ S4   │ S5   │ S6   │     │   │
│   │  │ LRU  │ LRU  │ LRU  │ LRU  │ LRU  │ LRU  │ LRU  │     │   │
│   │  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                Persistence Manager                      │   │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────┐            │   │
│   │  │   JSON    │  │   RDB     │  │   AOF     │            │   │
│   │  │ Snapshot  │  │  Binary   │  │   Log     │            │   │
│   │  └───────────┘  └───────────┘  └───────────┘            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                     Disk                                │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │  pendem.snapshot.json  (human readable)           │  │   │
│   │  │  pendem.rdb            (binary, fast)             │  │   │
│   │  │  pendem.aof            (incremental log)          │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3.2 Config Design

```go
// internal/config/config.go
type JSONConfig struct {
    Enabled      bool          // Enable/disable JSON snapshot
    Path         string        // File path
    Interval     time.Duration // Snapshot interval
    MaxSnapshots int           // Keep N versions
}

type RDBConfig struct {
    Enabled  bool          // Enable/disable RDB
    Path     string        // File path
    Interval time.Duration // Snapshot interval
}

type AOFConfig struct {
    Enabled       bool          // Enable/disable AOF
    FilePath      string        // File path
    FlushInterval time.Duration // Flush to disk interval
    SyncOnWrite   bool          // fsync on every write
}

type PersistenceConfig struct {
    JSON JSONConfig
    RDB  RDBConfig
    AOF  AOFConfig
}
```

## 8.4 Implementasi JSON Snapshot

### 8.4.1 Struktur Data

```go
// internal/persistence/json.go
type JSONManager[V any] struct {
    config    config.JSONConfig
    cache     *engine.Cache[string]
    mu        sync.RWMutex
    logger    *log.Logger
    stopChan  chan struct{}
    wg        sync.WaitGroup
    isRunning bool
}

// JSONData adalah struktur data yang disimpan ke JSON
type JSONData[V any] struct {
    Version    string              `json:"version"`
    Timestamp  int64               `json:"timestamp"`
    NumShards  int                 `json:"num_shards"`
    TotalItems int                 `json:"total_items"`
    Shards     []JSONShard[string] `json:"shards"`
}

type JSONShard[V any] struct {
    ID    int                       `json:"id"`
    Items map[string]engine.Item[V] `json:"items"`
}
```

### 8.4.2 Save - Menyimpan Snapshot

```go
func (sm *JSONManager[V]) Save() error {
    sm.mu.Lock()
    defer sm.mu.Unlock()

    sm.logger.Println("Starting JSON snapshot save...")
    startTime := time.Now()

    // 1. Collect data from all shards
    snapshotData, err := sm.collectData()
    if err != nil {
        return fmt.Errorf("failed to collect data: %w", err)
    }

    // 2. Marshal to JSON
    jsonData, err := json.MarshalIndent(snapshotData, "", "  ")
    if err != nil {
        return fmt.Errorf("failed to marshal JSON: %w", err)
    }

    // 3. Write to temp file (atomic write)
    tempFile := sm.config.Path + ".tmp"
    if err := os.WriteFile(tempFile, jsonData, 0644); err != nil {
        return fmt.Errorf("failed to write temp file: %w", err)
    }

    // 4. Rename (atomic operation)
    if err := os.Rename(tempFile, sm.config.Path); err != nil {
        return fmt.Errorf("failed to rename file: %w", err)
    }

    // 5. Rotate old snapshots
    if sm.config.MaxSnapshots > 0 {
        sm.rotateSnapshots()
    }

    elapsed := time.Since(startTime)
    sm.logger.Printf("Snapshot saved successfully in %v (%d items)",
        elapsed, snapshotData.TotalItems)

    return nil
}
```

### 8.4.3 Load - Memuat Snapshot

```go
func (sm *JSONManager[V]) Load() error {
    sm.mu.Lock()
    defer sm.mu.Unlock()

    if _, err := os.Stat(sm.config.Path); os.IsNotExist(err) {
        sm.logger.Println("No snapshot file found, starting with empty cache")
        return nil
    }

    sm.logger.Printf("Loading snapshot from %s...", sm.config.Path)
    startTime := time.Now()

    // 1. Read file
    jsonData, err := os.ReadFile(sm.config.Path)
    if err != nil {
        return fmt.Errorf("failed to read snapshot file: %w", err)
    }

    // 2. Parse JSON
    var snapshotData JSONData[V]
    if err := json.Unmarshal(jsonData, &snapshotData); err != nil {
        return fmt.Errorf("failed to parse JSON: %w", err)
    }

    // 3. Restore data
    if err := sm.restoreData(&snapshotData); err != nil {
        return fmt.Errorf("failed to restore data: %w", err)
    }

    elapsed := time.Since(startTime)
    sm.logger.Printf("Snapshot loaded successfully in %v (%d items)",
        elapsed, snapshotData.TotalItems)

    return nil
}
```

### 8.4.4 Rotasi Snapshot

```go
func (sm *JSONManager[V]) rotateSnapshots() {
    pattern := sm.config.Path + ".*"
    files, err := filepath.Glob(pattern)
    if err != nil {
        sm.logger.Printf("Error finding snapshot files: %v", err)
        return
    }

    files = append(files, sm.config.Path)

    // Sort by modification time (oldest first)
    sort.Slice(files, func(i, j int) bool {
        fi, _ := os.Stat(files[i])
        fj, _ := os.Stat(files[j])
        return fi.ModTime().Before(fj.ModTime())
    })

    // Remove oldest if exceed limit
    if len(files) > sm.config.MaxSnapshots {
        for i := 0; i < len(files)-sm.config.MaxSnapshots; i++ {
            os.Remove(files[i])
        }
    }
}
```

## 8.5 Implementasi RDB (Binary Format)

### 8.5.1 Format RDB

```text
+------------------+
| MAGIC: "PENDEM"  | 6 bytes
+------------------+
| VERSION: 1       | 1 byte
+------------------+
| NUM_SHARDS: 16   | 2 bytes
+------------------+
| For each shard:  |
|   SHARD_ID: 0    | 2 bytes
|   NUM_ITEMS: 10  | 4 bytes
|   For each item: |
|     KEY_LEN: 5   | 2 bytes
|     KEY: "user"  | KEY_LEN bytes
|     VAL_LEN: 10  | 4 bytes
|     VAL: "john"  | VAL_LEN bytes
|     TTL: 3600    | 8 bytes (0 = no TTL)
|     EXPIRE: ...  | 8 bytes
+------------------+
```

### 8.5.2 Implementasi Save

```go
// internal/persistence/rdb.go
const RDBMagic = "PENDEM"
const RDBVersion = 1

func (r *RDB) Save() error {
    r.mu.Lock()
    defer r.mu.Unlock()

    r.logger.Println("Starting RDB save...")
    startTime := time.Now()

    tempFile := r.config.Path + ".tmp"
    f, err := os.Create(tempFile)
    if err != nil {
        return fmt.Errorf("failed to create temp file: %w", err)
    }
    defer f.Close()

    w := bufio.NewWriter(f)

    // 1. Write magic + version
    if _, err := w.Write([]byte(RDBMagic)); err != nil {
        return err
    }
    if err := w.WriteByte(RDBVersion); err != nil {
        return err
    }

    // 2. Write number of shards
    numShards := r.cache.NumShards()
    if err := binary.Write(w, binary.LittleEndian, uint16(numShards)); err != nil {
        return err
    }

    totalItems := 0

    // 3. Write each shard
    for shardID := 0; shardID < numShards; shardID++ {
        shard := r.cache.GetShard(shardID)
        if shard == nil {
            continue
        }

        items := shard.GetItems()
        if len(items) == 0 {
            continue
        }

        // Write shard ID
        if err := binary.Write(w, binary.LittleEndian, uint16(shardID)); err != nil {
            return err
        }

        // Write number of items
        if err := binary.Write(w, binary.LittleEndian, uint32(len(items))); err != nil {
            return err
        }

        for key, item := range items {
            // Write key
            keyBytes := []byte(key)
            binary.Write(w, binary.LittleEndian, uint16(len(keyBytes)))
            w.Write(keyBytes)

            // Write value
            valBytes := []byte(item.Value)
            binary.Write(w, binary.LittleEndian, uint32(len(valBytes)))
            w.Write(valBytes)

            // Write TTL (0 = no TTL)
            ttl := item.TTL()
            if ttl < 0 {
                ttl = 0
            }
            binary.Write(w, binary.LittleEndian, int64(ttl.Seconds()))

            // Write expiration timestamp
            binary.Write(w, binary.LittleEndian, item.Expiration)

            totalItems++
        }
    }

    // 4. Flush and rename
    if err := w.Flush(); err != nil {
        return err
    }
    f.Close()

    if err := os.Rename(tempFile, r.config.Path); err != nil {
        return err
    }

    elapsed := time.Since(startTime)
    r.logger.Printf("RDB saved successfully in %v (%d items, %d shards)",
        elapsed, totalItems, numShards)

    return nil
}
```

### 8.5.3 Implementasi Load

```go
func (r *RDB) Load() error {
    r.mu.Lock()
    defer r.mu.Unlock()

    if _, err := os.Stat(r.config.Path); os.IsNotExist(err) {
        r.logger.Println("No RDB file found, starting with empty cache")
        return nil
    }

    r.logger.Printf("Loading RDB from %s...", r.config.Path)
    startTime := time.Now()

    f, err := os.Open(r.config.Path)
    if err != nil {
        return fmt.Errorf("failed to open RDB file: %w", err)
    }
    defer f.Close()

    reader := bufio.NewReader(f)

    // 1. Read magic
    magic := make([]byte, len(RDBMagic))
    io.ReadFull(reader, magic)
    if string(magic) != RDBMagic {
        return fmt.Errorf("invalid RDB magic: %s", string(magic))
    }

    // 2. Read version
    version, _ := reader.ReadByte()
    if version != RDBVersion {
        return fmt.Errorf("unsupported RDB version: %d", version)
    }

    // 3. Read number of shards
    var numShards uint16
    binary.Read(reader, binary.LittleEndian, &numShards)

    totalItems := 0

    // 4. Read each shard
    for {
        var shardID uint16
        err := binary.Read(reader, binary.LittleEndian, &shardID)
        if err == io.EOF {
            break
        }

        var numItems uint32
        binary.Read(reader, binary.LittleEndian, &numItems)

        shard := r.cache.GetShard(int(shardID))
        if shard == nil {
            // Skip items if shard not found
            for i := 0; i < int(numItems); i++ {
                // Skip key, value, TTL, expiration
                var keyLen uint16
                binary.Read(reader, binary.LittleEndian, &keyLen)
                reader.Discard(int(keyLen))

                var valLen uint32
                binary.Read(reader, binary.LittleEndian, &valLen)
                reader.Discard(int(valLen))

                var ttl int64
                binary.Read(reader, binary.LittleEndian, &ttl)

                var exp int64
                binary.Read(reader, binary.LittleEndian, &exp)
            }
            continue
        }

        items := make(map[string]engine.Item[string])
        for i := 0; i < int(numItems); i++ {
            // Read key
            var keyLen uint16
            binary.Read(reader, binary.LittleEndian, &keyLen)
            keyBytes := make([]byte, keyLen)
            io.ReadFull(reader, keyBytes)
            key := string(keyBytes)

            // Read value
            var valLen uint32
            binary.Read(reader, binary.LittleEndian, &valLen)
            valBytes := make([]byte, valLen)
            io.ReadFull(reader, valBytes)
            value := string(valBytes)

            // Read TTL and expiration
            var ttlSec int64
            binary.Read(reader, binary.LittleEndian, &ttlSec)

            var exp int64
            binary.Read(reader, binary.LittleEndian, &exp)

            // Skip expired items
            if exp > 0 && time.Now().UnixNano() > exp {
                continue
            }

            items[key] = engine.Item[string]{
                Value:      value,
                Expiration: exp,
            }
            totalItems++
        }

        shard.Restore(items)
        r.logger.Printf("Restored shard %d: %d items", shardID, len(items))
    }

    elapsed := time.Since(startTime)
    r.logger.Printf("RDB loaded successfully in %v (%d items)", elapsed, totalItems)

    return nil
}
```

## 8.6 Implementasi AOF (Append Only File)

### 8.6.1 Struktur AOF

```go
// internal/persistence/aof.go
type AOF struct {
    file          *os.File
    writer        *bufio.Writer
    mu            sync.Mutex
    logger        *log.Logger
    cache         *engine.Cache[string]
    flushInterval time.Duration
    stopChan      chan struct{}
    wg            sync.WaitGroup
}

func NewAOF(logger *log.Logger, cache *engine.Cache[string], flushInterval time.Duration) *AOF {
    return &AOF{
        logger:        logger,
        cache:         cache,
        flushInterval: flushInterval,
        stopChan:      make(chan struct{}),
    }
}
```

### 8.6.2 Log Command

```go
func (a *AOF) LogCommand(cmd string, args ...string) error {
    a.mu.Lock()
    defer a.mu.Unlock()

    // Format RESP: *<count>\r\n$<len>\r\n<cmd>\r\n...
    resp := fmt.Sprintf("*%d\r\n", 1+len(args))
    resp += fmt.Sprintf("$%d\r\n%s\r\n", len(cmd), cmd)
    for _, arg := range args {
        resp += fmt.Sprintf("$%d\r\n%s\r\n", len(arg), arg)
    }

    _, err := a.writer.WriteString(resp)
    return err
}
```

### 8.6.3 Flush dan Flush Loop

```go
func (a *AOF) Flush() {
    a.mu.Lock()
    defer a.mu.Unlock()

    if err := a.writer.Flush(); err != nil {
        a.logger.Printf("Failed to flush AOF: %v", err)
    }
}

func (a *AOF) flushLoop() {
    ticker := time.NewTicker(a.flushInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            a.Flush()
        case <-a.stopChan:
            a.Flush()
            return
        }
    }
}
```

### 8.6.4 Load dan Replay

```go
func (a *AOF) Load() error {
    file, err := os.Open("pendem.aof")
    if err != nil {
        return err
    }
    defer file.Close()

    startTime := time.Now()
    parser := server.NewRESPParser(file)
    commandCount := 0

    for {
        val, err := parser.Read()
        if err != nil {
            break
        }

        if err := a.replayCommand(val); err != nil {
            a.logger.Printf("Error replaying command: %v", err)
            continue
        }
        commandCount++
    }

    elapsed := time.Since(startTime)
    a.logger.Printf("AOF loaded successfully in %v (%d commands replayed)",
        elapsed, commandCount)

    return nil
}

func (a *AOF) replayCommand(val *server.RESPValue) error {
    if val.Type != server.Array || len(val.Array) < 1 {
        return fmt.Errorf("invalid AOF command format")
    }

    cmd := strings.ToUpper(val.Array[0].Str)
    args := make([]string, 0)

    for i := 1; i < len(val.Array); i++ {
        if val.Array[i].Type == server.BulkString {
            args = append(args, val.Array[i].Str)
        }
    }

    switch cmd {
    case "SET":
        if len(args) < 2 {
            return fmt.Errorf("invalid SET command: %v", args)
        }
        key := args[0]
        value := args[1]
        ttl := time.Duration(0)

        if len(args) >= 4 && strings.ToUpper(args[2]) == "EX" {
            if secs, err := strconv.Atoi(args[3]); err == nil {
                ttl = time.Duration(secs) * time.Second
            }
        }

        a.cache.Set(key, value, ttl)
        a.logger.Printf("AOF replay: SET %s = %s (TTL: %v)", key, value, ttl)

    case "DEL":
        if len(args) < 1 {
            return fmt.Errorf("invalid DEL command: %v", args)
        }
        for _, key := range args {
            a.cache.Delete(key)
            a.logger.Printf("AOF replay: DEL %s", key)
        }

    default:
        a.logger.Printf("AOF replay: unknown command '%s', skipping", cmd)
    }

    return nil
}
```

## 8.7 Persistence Manager

### 8.7.1 Struktur Manager

```go
// internal/persistence/manager.go
type Manager[V any] struct {
    json   *JSONManager[V]
    rdb    *RDB
    aof    *AOF
    logger *log.Logger
    cache  *engine.Cache[string]
    config config.PersistenceConfig
}

func NewManager[V any](logger *log.Logger, cache *engine.Cache[string], cfg config.PersistenceConfig) *Manager[V] {
    mgr := &Manager[V]{
        logger: logger,
        cache:  cache,
        config: cfg,
    }

    if cfg.JSON.Enabled {
        mgr.json = NewJSONManager[V](logger, cache, cfg.JSON)
    }
    if cfg.RDB.Enabled {
        mgr.rdb = NewRDB(logger, cache, cfg.RDB)
    }
    if cfg.AOF.Enabled {
        mgr.aof = NewAOF(logger, cache, cfg.AOF.FlushInterval)
    }

    return mgr
}
```

### 8.7.2 Start - Smart Recovery

```go
func (pm *Manager[V]) Start() error {
    // 1. Load data (Smart recovery)
    if err := pm.loadData(); err != nil {
        return err
    }

    // 2. Start background processes
    if pm.config.JSON.Enabled && pm.json != nil {
        pm.json.Start()
    }
    if pm.config.RDB.Enabled && pm.rdb != nil {
        pm.rdb.Start()
    }
    if pm.config.AOF.Enabled && pm.aof != nil {
        if err := pm.aof.Start(); err != nil {
            pm.logger.Printf("Warning: Failed to start AOF: %v", err)
        }
    }

    return nil
}

func (pm *Manager[V]) loadData() error {
    // Priority 1: AOF (most up-to-date)
    if pm.config.AOF.Enabled && pm.aof != nil {
        if err := pm.aof.Load(); err == nil {
            pm.logger.Println("✅ Data restored from AOF")
            return nil
        }
        pm.logger.Printf("⚠️ AOF load failed: %v, trying next...", err)
    }

    // Priority 2: RDB (fast binary)
    if pm.config.RDB.Enabled && pm.rdb != nil {
        if err := pm.rdb.Load(); err == nil {
            pm.logger.Println("✅ Data restored from RDB")
            return nil
        }
        pm.logger.Printf("⚠️ RDB load failed: %v, trying next...", err)
    }

    // Priority 3: JSON Snapshot (human readable)
    if pm.config.JSON.Enabled && pm.json != nil {
        if err := pm.json.Load(); err == nil {
            pm.logger.Println("✅ Data restored from JSON Snapshot")
            return nil
        }
        pm.logger.Printf("⚠️ JSON snapshot load failed: %v", err)
    }

    pm.logger.Println("ℹ️ No valid persistence found, starting with empty cache")
    return nil
}
```

### 8.7.3 Log Command dari Handler

```go
// internal/handler/handler.go
func (h *Handler[V]) Set(args []string) server.RESPValue {
    // ... existing logic

    h.cache.Set(key, value, ttl)

    // ✅ Log ke AOF
    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("SET", args...)
    }

    return server.RESPValue{Type: server.SimpleString, Str: "OK"}
}
```

## 8.8 Config File Support

### 8.8.1 Config File Format

```ini

# pendem.conf
[server]
max_connections = 50000
read_timeout = 2m
write_timeout = 30s
idle_timeout = 5m

[engine]
max_memory = 2GB
eviction_policy = lru
evictor_capacity = 20000
shard_count = 16
default_ttl = 0

[persistence.rdb]
enabled = true
path = pendem.rdb
interval = 1h

[persistence.aof]
enabled = true
path = pendem.aof
flush_interval = 5m
sync_on_write = false

[persistence.json]
enabled = false
path = pendem.snapshot.json
interval = 1h
max_snapshots = 5
```

### 8.8.2 Load Config dari File

```go
// internal/config/config.go
func LoadConfig(path string) (Config, error) {
    parser := NewConfigParser()
    if err := parser.Parse(path); err != nil {
        return DefaultConfig(), fmt.Errorf("failed to parse config: %w", err)
    }

    cfg := DefaultConfig()

    // [server] section
    if v, ok := parser.GetInt("server", "max_connections"); ok {
        cfg.Server.MaxConnections = v
    }
    if v, ok := parser.GetDuration("server", "read_timeout"); ok {
        cfg.Server.ReadTimeout = v
    }
    // ... other sections

    return cfg, nil
}
```

### 8.8.3 Multi-Lokasi Config File

```go
// internal/config/path.go
func DefaultConfigPaths() []string {
    paths := []string{}

    // 1. Current directory
    paths = append(paths, "pendem.conf")
    paths = append(paths, "config/pendem.conf")

    // 2. User home
    if home := os.Getenv("HOME"); home != "" {
        paths = append(paths, filepath.Join(home, ".pendem", "pendem.conf"))
    }

    // 3. System directory
    paths = append(paths, "/etc/pendem/pendem.conf")

    return paths
}
```

## 8.9 Testing Persistence

### 8.9.1 Test dengan redis-cli

```bash
# 1. Start server
go run cmd/main.go

# 2. Set data
redis-cli -h localhost -p 6379
127.0.0.1:6379> SET user:1 "John"
OK
127.0.0.1:6379> SET user:2 "Jane" EX 3600
OK

# 3. Check AOF file
cat pendem.aof
*3\r\n$3\r\nSET\r\n$5\r\nuser:1\r\n$4\r\nJohn\r\n
*4\r\n$3\r\nSET\r\n$5\r\nuser:2\r\n$4\r\nJane\r\n$2\r\nEX\r\n$4\r\n3600\r\n

# 4. Restart server
^C
go run cmd/main.go

# 5. Data should still exist
127.0.0.1:6379> GET user:1
"John"
127.0.0.1:6379> GET user:2
"Jane"
```

### 8.9.2 Log Output

```bash
[PENDEM] Persistence: JSON=false, RDB=true, AOF=true
[PENDEM] AOF replay: SET user:1 = John (TTL: 0s)
[PENDEM] AOF replay: SET user:2 = Jane (TTL: 1h0m0s)
[PENDEM] AOF loaded successfully in 352µs (2 commands replayed)
[PENDEM] ✅ Data restored from AOF
[PENDEM] RDB: Auto-snapshot started (interval: 1m0s)
[PENDEM] Server listening on :6379
```

## 8.10 Best Practices

### 8.10.1 Interval Rekomendasi

| Use Case | RDB Interval | AOF Flush | Data Loss |
|----------|--------------|-----------|-----------|
| Development | 1 jam | 5 menit | 5 menit |
| Production | 1 jam | 1 menit | 1 menit |
| Critical Data | 30 menit | 1 detik | 1 detik |

### 8.10.2 Security

```go
// ✅ Gunakan atomic write (temp file + rename)
tempFile := path + ".tmp"
os.WriteFile(tempFile, data, 0644)
os.Rename(tempFile, path)

// ✅ Gunakan mutex untuk operasi file
sm.mu.Lock()
defer sm.mu.Unlock()

// ✅ Jangan pernah commit file data ke git
// .gitignore: *.aof, *.rdb, *.snapshot.json
```

### 8.10.3 Performance

```go
// ✅ Gunakan bufio.Writer untuk buffering
writer := bufio.NewWriter(file)

// ✅ Flush periodik (bukan setiap write)
ticker := time.NewTicker(flushInterval)

// ✅ Snapshot di background
go func() {
    sm.Save()
}()
```

## 8.11 Ringkasan

Yang Sudah Kita Pelajari:

1. Snapshot (JSON)
- Human readable
- Mudah debugging
- Atomic write dengan temp file
- Rotasi snapshot

2. RDB (Binary)
- Compact dan cepat
- Binary format
- Fast recovery

3. AOF (Append Only File)
- Mencatat setiap write
- Minimal data loss
- Replay command

4. Persistence Manager
- Smart recovery (AOF > RDB > JSON)
- Background processes
- Clean integration

5. Config File
- pendem.conf
- Multi-lokasi
- Environment override

**Perbandingan Format:**

| Aspek | JSON | RDB | AOF |
|-------|------|-----|-----|
| Human Readable | ✅ | ❌ | ⚠️ (RESP format) |
| File Size | ❌ | ✅ | ⚠️ |
| Save Speed | ❌ | ✅ | ✅ |
| Load Speed | ❌ | ✅ | ⚠️ |
| Data Loss | 1 jam | 1 jam | 5 menit |
| Debugging | ✅ | ❌ | ✅ |

Dengan persistence yang sudah solid, kita bisa menambahkan fitur batch operations untuk meningkatkan performa.