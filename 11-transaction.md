# Bab 11: Transaction - Atomic Operations dengan MULTI, EXEC, dan WATCH

## 11.1 Pendahuluan: Mengapa Transaction Diperlukan?

### 11.1.1 Masalah: Operasi Non-Atomic

Dalam aplikasi nyata, seringkali kita perlu melakukan beberapa operasi yang harus berjalan secara atomic (semua berhasil atau semua gagal).

>    📂 Kode Lengkap Bab Ini:
>    Seluruh kode yang dibahas di bab ini tersedia di GitHub:
>
>    🔗 [github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/11-transaction](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/11-transaction)


```text
Kasus : Memindahkan balance alice ke bob sebanyak 10
┌─────────────────────────────────────────────────────────────────┐
│              MASALAH: OPERASI NON-ATOMIC                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ❌ Tanpa Transaction:                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Client A:                                              │   │
│   │  1. GET balance:alice → 100                             │   │
│   │  2. SET balance:alice 90                                │   │
│   │  3. GET balance:bob → 50                                │   │
│   │  4. SET balance:bob 60                                  │   │
│   │                                                         │   │
│   │  💥 Jika step 3 gagal, data tidak konsisten!            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ✅ Dengan Transaction (MULTI/EXEC):                           │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Client A:                                              │   │
│   │  MULTI                                                  │   │
│   │  GET balance:alice                                      │   │
│   │  SET balance:alice 90                                   │   │
│   │  GET balance:bob                                        │   │
│   │  SET balance:bob 60                                     │   │
│   │  EXEC                                                   │   │
│   │                                                         │   │
│   │  ✅ Semua berhasil atau semua gagal!                    │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 11.1.2 Solusi: Transaction

Transaction di Redis memungkinkan kita mengelompokkan beberapa perintah menjadi satu unit atomic.

```text

┌─────────────────────────────────────────────────────────────────┐
│              TRANSACTION DIAGRAM                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Client ──▶ MULTI                                              │
│   Server ──▶ OK (mulai transaksi)                               │
│                                                                 │
│   Client ──▶ SET key1 value1                                    │
│   Server ──▶ QUEUED                                             │
│                                                                 │
│   Client ──▶ SET key2 value2                                    │
│   Server ──▶ QUEUED                                             │
│                                                                 │
│   Client ──▶ GET key1                                           │
│   Server ──▶ QUEUED                                             │
│                                                                 │
│   Client ──▶ EXEC                                               │
│   Server ──▶ [OK, OK, "value1"]  ← Eksekusi semua!              │
│                                                                 │
│   ❗ Perintah di-queue, baru dieksekusi saat EXEC               │
└─────────────────────────────────────────────────────────────────┘
```

### 11.1.3 Karakteristik Transaction di Redis

| Karakteristik | Keterangan |
|---------------|------------|
| Atomic | Semua perintah dieksekusi atau tidak ada |
| Isolated | Tidak ada perintah client lain yang disisipkan |
| Queued | Perintah di-queue, bukan langsung dieksekusi |
| No Rollback | Jika ada error saat eksekusi, tidak ada rollback |

## 11.2 Desain Transaction

### 11.2.1 Arsitektur

```text

┌─────────────────────────────────────────────────────────────────┐
│              TRANSACTION ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Client Connection                    │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │              Transaction State                    │  │   │
│   │  │  • InTransaction: bool                            │  │   │
│   │  │  • QueuedCommands: []Command                      │  │   │
│   │  │  • WatchedKeys: map[string]bool                   │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Command Flow                         │   │
│   │                                                         │   │
│   │  MULTI     → Set InTransaction = true                   │   │
│   │  SET key   → Queue command                              │   │
│   │  GET key   → Queue command                              │   │
│   │  EXEC      → Execute all queued commands                │   │
│   │  DISCARD   → Clear queue, cancel transaction            │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2.2 Komponen Transaction

```go

// internal/server/connection.go
package server

type Connection struct {
	// ... existing field

	// Transaction state
	inTransaction  bool
	queuedCommands []*RESPValue
	watchedKeys    map[string]bool
	dirty          bool
}
```

## 11.3 Implementasi MULTI, EXEC, DISCARD

### 11.3.1 Transaction State per Connection

Kita perlu menyimpan state transaction per connection:

```go
// internal/server/connection.go (modified)

type Connection struct {
	conn         net.Conn
	lastActivity int64
	idleTimeout  time.Duration
	logger       *log.Logger
	remoteAddr   string
	done         chan struct{}
	mu           sync.Mutex
	closed       bool

	// Transaction state
	inTransaction  bool
	queuedCommands []*RESPValue
	watchedKeys    map[string]bool
	dirty          bool
}

func NewConnection(conn net.Conn, idleTimeout time.Duration, logger *log.Logger) *Connection {
	return &Connection{
		conn:           conn,
		remoteAddr:     conn.RemoteAddr().String(),
		idleTimeout:    idleTimeout,
		logger:         logger,
		done:           make(chan struct{}),
		inTransaction:  false,
		queuedCommands: make([]*RESPValue, 0),
		watchedKeys:    make(map[string]bool),
		dirty:          false,
	}
}

// ... existing code

// ResetTransaction clears transaction state
func (c *Connection) ResetTransaction() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.inTransaction = false
	c.queuedCommands = make([]*RESPValue, 0)
	c.watchedKeys = make(map[string]bool)
	c.dirty = false
}

// WatchKey adds a key to watch list
func (c *Connection) WatchKey(key string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.watchedKeys[key] = true
}

// IsWatched checks if a key is being watched
func (c *Connection) IsWatched(key string) bool {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.watchedKeys[key]
}

// MarkDirty marks transaction as dirty (watched key changed)
func (c *Connection) MarkDirty() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.dirty = true
}

// IsDirty returns true if transaction is dirty
func (c *Connection) IsDirty() bool {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.dirty
}
```

### 11.3.2 MULTI Implementation

```go
// internal/server/server.go

func (s *Server) handleMulti(conn *Connection, args []string) RESPValue {
	if len(args) > 0 {
		return RESPValue{
			Type: Error,
			Str:  "ERR wrong number of arguments for 'multi' command",
		}
	}

	// If already in transaction, ignore (Redis behavior)
	if conn.inTransaction {
		return RESPValue{
			Type: SimpleString,
			Str:  "OK",
		}
	}

	conn.mu.Lock()
	conn.inTransaction = true
	conn.queuedCommands = make([]*RESPValue, 0)
	conn.mu.Unlock()

	return RESPValue{
		Type: SimpleString,
		Str:  "OK",
	}
}
```

### 11.3.3 Transaction Command Router

```go
// internal/server/transaction.go

func (s *Server) handleTxCmd(conn *Connection, value *RESPValue) (RESPValue, string, []string, bool) {
	if len(value.Array) < 1 {
		return *value, "", nil, false
	}

	cmd, args := s.parseCmd(value)

	switch cmd {
	case "MULTI":
		return s.handleMulti(conn, args), cmd, args, true
	case "EXEC":
		return s.handleExec(conn, args), cmd, args, true
	case "DISCARD":
		return s.handleDiscard(conn, args), cmd, args, true
	}

	if conn.inTransaction {
		// Queue command
		conn.mu.Lock()
		conn.queuedCommands = append(conn.queuedCommands, value)
		conn.mu.Unlock()

		return RESPValue{
			Type: SimpleString,
			Str:  "QUEUED",
		}, cmd, args, true
	}

	return *value, "", nil, false
}
```

### 11.3.3 Queue Command dalam Transaction

```go
// internal/server/server.go

// handleConnection menangani satu koneksi
func (s *Server) handleConnection(conn net.Conn) {
	// Tambahkan ke WaitGroup untuk gracefull shutdown
	s.wg.Add(1)
	defer s.wg.Done()

	c := NewConnection(conn, s.config.IdleTimeout, s.logger)

    // ... existing code
	
    // Loop membaca perintah dari client
	for {
        // .... existng code
		
		// Wait for read OR shutdown
		select {
		case <-shutdown:
			s.logger.Printf("Shutdown interrupt during read from %s", c.remoteAddr)
			return
		case res := <-readCh:
            // .... existing code
			

			// Proses command
			response := s.processCommandWithTx(c, res.respVal)

			// Set write timeout
			if s.config.WriteTimeout > 0 {
				conn.SetWriteDeadline(time.Now().Add(s.config.WriteTimeout))
			}

			conn.Write([]byte(EncodeRESP(response)))
			c.UpdateActivity()
		}
	}
}

func (s *Server) processCommandWithTx(c *Connection, value *RESPValue) RESPValue {
	result, cmd, args, isTxCmd := s.handleTxCmd(c, value)
	if isTxCmd {
		return result
	}

	return s.processCommand(value, cmd, args)
}

// processCommand memproses perintah dari client
func (s *Server) processCommand(value *RESPValue, cmd string, args []string) RESPValue {
	if value.Type != Array || len(value.Array) < 1 {
		return RESPValue{
			Type: Error,
			Str:  "ERR invalid command",
		}
	}
	if len(cmd) == 0 {
		cmd, args = s.parseCmd(value)
	}

	handler, exists := s.GetHandler(cmd)
	if !exists {
		return RESPValue{
			Type: Error,
			Str:  fmt.Sprintf("ERR unknown command '%s'", cmd),
		}
	}

	// Eksekusi handler
	return handler(args)
}

func (s *Server) parseCmd(value *RESPValue) (string, []string) {
	// Perintah pertama adalah command
	cmd := strings.ToUpper(value.Array[0].Str)
	args := make([]string, 0)

	for i := 1; i < len(value.Array); i++ {
		if value.Array[i].Type == BulkString {
			args = append(args, value.Array[i].Str)
		}
	}
	return cmd, args
}
```

### 11.3.5 EXEC - Menjalankan Transaction

```go
// internal/server/transaction.go

func (s *Server) handleExec(conn *Connection, args []string) RESPValue {
	if !conn.inTransaction {
		return RESPValue{
			Type: Error,
			Str:  "ERR EXEC without MULTI",
		}
	}

	// Check if transaction is dirty (watched keys changed)
	if conn.IsDirty() {
		conn.ResetTransaction()
		return RESPValue{
			Type:   BulkString,
			Str:    "",
			IsNull: true,
		}
	}

	// Get queued commands
	conn.mu.Lock()
	commands := conn.queuedCommands
	conn.queuedCommands = make([]*RESPValue, 0)
	conn.inTransaction = false
	conn.mu.Unlock()

	// Execute all commands
	results := make([]RESPValue, 0, len(commands))
	for _, cmd := range commands {
		// Execute command (without recursion)
		result := s.processCommand(cmd, "", nil)
		results = append(results, result)

		// If error occurs, continue (no rollback in Redis)
		// This is the Redis way
	}

	return RESPValue{
		Type:  Array,
		Array: results,
	}
}
```

### 11.3.6 DISCARD - Membatalkan Transaction

```go
// internal/server/transaction.go

func (s *Server) handleDiscard(conn *Connection, args []string) RESPValue {
	if !conn.inTransaction {
		return RESPValue{
			Type: Error,
			Str:  "ERR DISCARD without MULTI",
		}
	}

	conn.ResetTransaction()

	return RESPValue{
		Type: SimpleString,
		Str:  "OK",
	}
}
```

## 11.4 Implementasi WATCH

### 11.4.1 Konsep WATCH

```text

┌─────────────────────────────────────────────────────────────────┐
│              WATCH CONCEPT                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Client A:                    Client B:                        │
│   ┌─────────────────────────┐  ┌─────────────────────────────┐  │
│   │  WATCH key1             │  │  SET key1 new_value         │  │
│   │  MULTI                  │  │                             │  │
│   │  SET key1 value1        │  │                             │  │
│   │  EXEC                   │  │                             │  │
│   └─────────────────────────┘  └─────────────────────────────┘  │
│                                                                 │
│   ❗ Jika Client B mengubah key1 sebelum EXEC,                  │
│      transaksi Client A dibatalkan!                             │
└─────────────────────────────────────────────────────────────────┘
```

### 11.4.2 WATCH Implementation

Update transaaction command router

```go
// internal/server/transaction.go

func (s *Server) handleTxCmd(conn *Connection, value *RESPValue) (RESPValue, string, []string, bool) {
	// ...existing code 

	switch cmd {
	case "MULTI":
		return s.handleMulti(conn, args), cmd, args, true
	case "WATCH":
		return s.handleWatch(conn, args), cmd, args, true
	case "UNWATCH":
		return s.handleUnwatch(conn, args), cmd, args, true
	case "EXEC":
		keys := s.extractKeys(cmd, args)
		s.removeWatcherByKeys(keys, conn)

		return s.handleExec(conn, args), cmd, args, true
	case "DISCARD":
		keys := s.extractKeys(cmd, args)
		s.removeWatcherByKeys(keys, conn)

		return s.handleDiscard(conn, args), cmd, args, true
	}

	// ... existing code
}
```

Handle Watch

```go
// internal/server/transaction.go

// WATCH key [key ...]
func (s *Server) handleWatch(conn *Connection, args []string) RESPValue {
	if len(args) < 1 {
		return RESPValue{
			Type: Error,
			Str:  "ERR wrong number of arguments for 'watch' command",
		}
	}
	// If in transaction, WATCH is ignored (Redis behavior)
	if conn.inTransaction {
		s.logger.Printf("🔍 WATCH ignored (in transaction)")
		return RESPValue{
			Type: SimpleString,
			Str:  "OK",
		}
	}

	// Add keys to watch list
	for _, key := range args {
		conn.WatchKey(key)
		s.addWatcher(key, conn)
		s.logger.Printf("🔍 Added watcher for key: %s", key)
	}

	return RESPValue{
		Type: SimpleString,
		Str:  "OK",
	}
}

// UNWATCH - clear all watched keys
func (s *Server) handleUnwatch(conn *Connection, args []string) RESPValue {
	conn.mu.Lock()
	conn.watchedKeys = make(map[string]bool)
	conn.mu.Unlock()

	return RESPValue{
		Type: SimpleString,
		Str:  "OK",
	}
}
```

### 11.4.3 Check Watched Keys on Write

```go
// internal/server/server.go

// processCommand memproses perintah dari client
func (s *Server) processCommand(value *RESPValue, cmd string, args []string) RESPValue {
	// ... existing code

	// Check if any watched key was modified
	if s.isWriteCommand(cmd) {
		keys := s.extractKeys(cmd, args)
		if len(keys) > 0 {
			s.markWatchedKeysDirty(keys)
		}
	}

	// Eksekusi handler
	return handler(args)
}
```

```go
// internal/server/transaction.go

// isWriteCommand checks if a command modifies data
func (s *Server) isWriteCommand(cmd string) bool {
	writeCommands := map[string]bool{
		"SET": true, "DEL": true, "HSET": true, "HDEL": true,
		"LPUSH": true, "RPUSH": true, "LPOP": true, "RPOP": true,
		"SADD": true, "SREM": true, "ZADD": true, "ZREM": true,
		"MSET": true, "MSETNX": true, "APPEND": true,
	}
	return writeCommands[cmd]
}

// extractKeys mengambil keys dari command
func (s *Server) extractKeys(cmd string, args []string) []string {
	switch cmd {
	case "SET", "DEL", "GET", "TTL", "EXISTS", "APPEND", "STRLEN":
		if len(args) > 0 {
			return []string{args[0]}
		}
	case "HSET", "HGET", "HDEL", "HLEN", "HEXISTS", "HKEYS", "HVALS":
		if len(args) > 0 {
			return []string{args[0]}
		}
	case "LPUSH", "RPUSH", "LPOP", "RPOP", "LLEN":
		if len(args) > 0 {
			return []string{args[0]}
		}
	case "SADD", "SREM", "SMEMBERS", "SISMEMBER", "SCARD":
		if len(args) > 0 {
			return []string{args[0]}
		}
	case "ZADD", "ZRANGE", "ZREM", "ZCARD", "ZSCORE":
		if len(args) > 0 {
			return []string{args[0]}
		}
	case "MGET", "MSET", "MSETNX":
		// Multiple keys
		keys := make([]string, 0)
		for i := 0; i < len(args); i += 2 {
			keys = append(keys, args[i])
		}
		return keys
	}
	return []string{}
}

// markWatchedKeysDirty menandai semua koneksi yang menonton key sebagai dirty
func (s *Server) markWatchedKeysDirty(keys []string) {
	if len(keys) == 0 {
		return
	}

	s.watcherMu.Lock()
	defer s.watcherMu.Unlock()

	for _, key := range keys {
		if watchers, exists := s.watchers[key]; exists {
			for _, conn := range watchers {
				conn.MarkDirty()
			}
			// Hapus watchers setelah ditandai (Redis behavior)
			delete(s.watchers, key)
		}
	}
}
```

### 11.4.4 Global Watcher Tracking

```go
// internal/server/server.go

type Server struct {
    // ... existing fields ...
    watchers  map[string][]*Connection // key → list of connections
	watcherMu sync.RWMutex
}

// Konstruktior: NewServer membuat instance server baru
func NewServer(addr string, log *log.Logger) *Server {
	s := &Server{
		// ... existing field
		watchers:   make(map[string][]*Connection),
	}

	return s
}
```

```go
// internal/server/transaction.go

// AddWatcher adds a connection to watchers for a key
func (s *Server) addWatcher(key string, conn *Connection) {
	s.watcherMu.Lock()
	defer s.watcherMu.Unlock()

	// Remove existing watcher for this connection
	// Menggunakan unsafe karena sudah dilock di sini untuk mencegah deadlock
	s.removeWatcherUnsafe(key, conn)

	s.watchers[key] = append(s.watchers[key], conn)
}

// RemoveWatcher removes a connection from watchers for a key
func (s *Server) removeWatcherUnsafe(key string, conn *Connection) {
	if watchers, exists := s.watchers[key]; exists {
		newWatchers := make([]*Connection, 0, len(watchers))
		for _, w := range watchers {
			if w != conn {
				newWatchers = append(newWatchers, w)
			}
		}
		if len(newWatchers) == 0 {
			delete(s.watchers, key)
		} else {
			s.watchers[key] = newWatchers
		}
	}
}

```

## 11.5 Integrasi dengan Connection

### 11.5.1 Update handleConnection

```go
// internal/server/server.go

// handleConnection menangani satu koneksi
func (s *Server) handleConnection(conn net.Conn) {
	// Tambahkan ke WaitGroup untuk gracefull shutdown
	s.wg.Add(1)
	defer s.wg.Done()

	c := NewConnection(conn, s.config.IdleTimeout, s.logger)

	// Pastikan connection ditutup saat function selesai
	defer func() {
		c.Close()
		s.cleanupWatchers(c)
		// Kurangi counter aktif koneksi
		atomic.AddInt32(&s.activeConn, -1)
		s.logger.Printf("Active connections: %d", atomic.LoadInt32(&s.activeConn))
	}()

    // ... existing code	
}
```

```go
// internal/server/transaction.go

// cleanupWatchers removes all watchers for a connection
func (s *Server) cleanupWatchers(conn *Connection) {
	conn.mu.Lock()
	keys := make([]string, 0, len(conn.watchedKeys))
	for key := range conn.watchedKeys {
		keys = append(keys, key)
	}
	conn.mu.Unlock()

	for _, key := range keys {
		s.removeWatcher(key, conn)
	}
}

// removes a connection from watchers for a key
func (s *Server) removeWatcher(key string, conn *Connection) {
	s.watcherMu.Lock()
	defer s.watcherMu.Unlock()

	s.removeWatcherUnsafe(key, conn)
}

// removes a connection from watchers for keys
func (s *Server) removeWatcherByKeys(keys []string, conn *Connection) {
	for _, key := range keys {
		s.removeWatcher(key, conn)
	}
}
```

## 11.6 Testing Transaction

### 11.6.1 Test dengan redis-cli

```bash

redis-cli -h localhost -p 6378

# ============================================
# BASIC TRANSACTION
# ============================================

127.0.0.1:6378> MULTI
OK

127.0.0.1:6378> SET key1 value1
QUEUED

127.0.0.1:6378> SET key2 value2
QUEUED

127.0.0.1:6378> GET key1
QUEUED

127.0.0.1:6378> EXEC
1) OK
2) OK
3) "value1"

# ============================================
# TRANSACTION WITH ERROR
# ============================================

127.0.0.1:6378> MULTI
OK

127.0.0.1:6378> SET key1 value1
QUEUED

127.0.0.1:6378> INVALID
QUEUED

127.0.0.1:6378> GET key1
QUEUED

127.0.0.1:6378> EXEC
1) OK
2) (error) ERR unknown command 'INVALID'
3) "value1"

# ============================================
# DISCARD
# ============================================

127.0.0.1:6378> MULTI
OK

127.0.0.1:6378> SET key1 value1
QUEUED

127.0.0.1:6378> DISCARD
OK

127.0.0.1:6378> GET key1
(nil)

# ============================================
# WATCH
# ============================================

127.0.0.1:6378> SET key1 value1
OK

127.0.0.1:6378> WATCH key1
OK

127.0.0.1:6378> MULTI
OK

127.0.0.1:6378> SET key1 value2
QUEUED

# Di terminal lain
127.0.0.1:6378> SET key1 value3
OK

# Kembali ke terminal pertama
127.0.0.1:6378> EXEC
(nil)

127.0.0.1:6378> GET key1
"value3"
```

## 11.7 Testing dengan nc

Pipeline dengan Transaction

```bash

# MULTI + SET + GET + EXEC
printf $'*1\r\n$5\r\nMULTI\r\n*3\r\n$3\r\nSET\r\n$4\r\nkey1\r\n$6\r\nvalue1\r\n*2\r\n$3\r\nGET\r\n$4\r\nkey1\r\n*1\r\n$4\r\nEXEC\r\n' | nc localhost 6378

# Output:
+OK
+QUEUED
+QUEUED
*2
+OK
$6
value1
```

## 11.8 Perbandingan dengan Pipeline

| Aspek | Pipeline | Transaction (MULTI/EXEC) |
|-------|----------|--------------------------|
| Tujuan | Mengurangi round-trip | Atomic operations |
| Eksekusi | Langsung | Di-queue, lalu dieksekusi |
| Atomic | ❌ Tidak | ✅ Ya |
| Error Handling | Per-command | Per-command (no rollback) |
| WATCH | ❌ Tidak | ✅ Ya |
| Use Case | Bulk operations | Atomic updates |

## 11.9 Best Practices

### 11.9.1 Gunakan WATCH untuk Optimistic Locking

```go

// Transfer saldo dengan WATCH
WATCH balance:alice balance:bob
MULTI
DECR balance:alice 10
INCR balance:bob 10
EXEC
// Jika ada perubahan, EXEC return nil, retry
```

### 11.9.2 Retry on Failure

Di sisi client, handle jika terjadi kegagalan maka lakukan retry.

```go
// Retry transaction jika WATCH gagal
func transferMoney(from, to string, amount int) {
    for {
        WATCH from to
        MULTI
        DECR from amount
        INCR to amount
        result := EXEC
        
        if result != nil {
            break // Success
        }
        // Retry
    }
}
```

### 11.9.3 Jangan Gunakan Transaction untuk Read-Only

```go
// ❌ Jangan: Transaction hanya untuk read
MULTI
GET key1
GET key2
EXEC

// ✅ Lebih baik: Pipeline
GET key1
GET key2
// Atau MGET
MGET key1 key2
```

## 11.10 Ringkasan

Yang Sudah Kita Pelajari:
- MULTI - Memulai transaction
- EXEC - Menjalankan semua perintah
- DISCARD - Membatalkan transaction
- WATCH - Optimistic locking
- UNWATCH - Menghapus semua watch

**Command Summary**

| Command | Fungsi |
| MULTI | Memulai transaction |
| EXEC | Jalankan semua perintah |
| DISCARD | Batalkan transaction |
| WATCH | Awasi key untuk perubahan |
| UNWATCH | Hapus semua watch |

Dengan transaction, kita bisa melakukan operasi atomic. Selanjutnya, kita akan menambahkan fitur messaging & queue.