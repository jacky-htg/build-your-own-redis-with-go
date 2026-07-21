# Bab 10: Batch Operations - Meningkatkan Performa dengan Operasi Batch

## 10.1 Pendahuluan: Mengapa Batch Operations Penting?

### 10.1.1 Masalah: Network Round-Trip

Dalam aplikasi nyata, seringkali kita perlu melakukan banyak operasi sekaligus. Misalnya, mengambil 100 data dari cache atau menyimpan 100 data sekaligus.

>    📂 Kode Lengkap Bab Ini:
>    Seluruh kode yang dibahas di bab ini tersedia di GitHub:
>
>    🔗 [github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/10-batch-operation](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/10-batch-operation)


```text

┌─────────────────────────────────────────────────────────────────┐
│              MASALAH: BANYAK ROUND-TRIP                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ❌ Tanpa Batch: 100 GET = 100 round-trip                      │
│                                                                 │
│   Client ──▶ GET key1 ──▶ Server ──▶ Response ──▶ Client        │
│   Client ──▶ GET key2 ──▶ Server ──▶ Response ──▶ Client        │
│   Client ──▶ GET key3 ──▶ Server ──▶ Response ──▶ Client        │
│   ... (100 kali)                                                │
│                                                                 │
│   ⏱️ Total waktu = 100 × (latency + processing)                 │
│                                                                 │
│   ❗ Lambat! Latency network dominan!                           │
└─────────────────────────────────────────────────────────────────┘
```

### 10.1.2 Solusi: Batch Operations

Batch operations memungkinkan kita mengirim banyak perintah dalam satu kali request.

```text

┌─────────────────────────────────────────────────────────────────┐
│              SOLUSI: BATCH OPERATIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ✅ Dengan Batch: 1 MGET = 1 round-trip                        │
│                                                                 │
│   Client ──▶ MGET key1 key2 key3 ... key100 ──▶ Server          │
│              └────────────────────────────┘                     │
│   Server ──▶ [value1, value2, value3, ...] ──▶ Client           │
│                                                                 │
│   ⏱️ Total waktu = 1 × (latency + 100×processing)               │
│                                                                 │
│   ❗ Jauh lebih cepat!                                          │
└─────────────────────────────────────────────────────────────────┘
```

### 10.1.3 Keuntungan Batch Operations

| Keuntungan | Keterangan |
|------------|------------|
| Mengurangi Network I/O | 1 request vs N request |
| Mengurangi Latency | 1 round-trip vs N round-trip |
| Meningkatkan Throughput | Lebih banyak operasi per detik |
| Efisien | Mengurangi overhead TCP |

## 10.2 MGET - Mendapatkan Multiple Keys

### 10.2.1 Perintah MGET

```text

MGET key [key ...]
```

**Fungsi:** Mendapatkan nilai dari satu atau lebih key.

**Return:** Array of values (null jika key tidak ada).

### 10.2.2 Implementasi

```go
// internal/handler/batch.go
package handler

import (
	"pendem/internal/server"
)

// MGET key [key ...]
func (h *Handler[V]) MGet(args []string) server.RESPValue {
	if len(args) < 1 {
		return server.RESPValue{
			Type: server.Error,
			Str:  "ERR wrong number of arguments for 'mget' command",
		}
	}

	result := make([]server.RESPValue, len(args))
	for i, key := range args {
		val, found := h.cache.Get(key)
		if !found {
			result[i] = server.RESPValue{
				Type:   server.BulkString,
				Str:    "",
				IsNull: true,
			}
		} else {
			result[i] = server.RESPValue{
				Type: server.BulkString,
				Str:  val,
			}
		}
	}

	return server.RESPValue{
		Type:  server.Array,
		Array: result,
	}
}
```

### 10.2.3 Testing

```bash

redis-cli -h localhost -p 6378

# Set data
127.0.0.1:6378> SET key1 "value1"
OK
127.0.0.1:6378> SET key2 "value2"
OK
127.0.0.1:6378> SET key3 "value3"
OK

# MGET
127.0.0.1:6378> MGET key1 key2 key3
1) "value1"
2) "value2"
3) "value3"

# MGET dengan key tidak ada
127.0.0.1:6378> MGET key1 key2 key4
1) "value1"
2) "value2"
3) (nil)
```

## 10.3 MSET - Menyimpan Multiple Keys

### 10.3.1 Perintah MSET

```text

MSET key value [key value ...]
```

**Fungsi:** Menyimpan satu atau lebih key-value pairs.

**Return:** OK.

### 10.3.2 Implementasi

```go
// internal/handler/batch.go

func (h *Handler[V]) MSet(args []string) server.RESPValue {
	if len(args) < 2 {
		return server.RESPValue{
			Type: server.Error,
			Str:  "ERR wrong number of arguments for 'mset' command",
		}
	}

	if len(args)%2 != 0 {
		return server.RESPValue{
			Type: server.Error,
			Str:  "ERR wrong number of arguments for 'mset' command",
		}
	}

	for i := 0; i < len(args); i += 2 {
		key := args[i]
		value := args[i+1]
		h.cache.Set(key, value, 0)
	}

	if h.persistenceMgr != nil {
		go h.persistenceMgr.LogCommand("MSET", args...)
	}

	return server.RESPValue{
		Type: server.SimpleString,
		Str:  "OK",
	}
}
```

### 10.3.3 Testing

```bash

redis-cli -h localhost -p 6378

# MSET
127.0.0.1:6378> MSET key1 value1 key2 value2 key3 value3
OK

# Verify
127.0.0.1:6378> MGET key1 key2 key3
1) "value1"
2) "value2"
3) "value3"

# Error: odd number of arguments
127.0.0.1:6378> MSET key1 value1 key2
(error) ERR wrong number of arguments for 'mset' command
```

## 10.4 MSETNX - MSET if Not Exists

### 10.4.1 Perintah MSETNX

```text

MSETNX key value [key value ...]
```

**Fungsi:** Menyimpan multiple keys hanya jika semua key belum ada.

**Return:** 1 jika semua key baru, 0 jika ada yang sudah ada.

### 10.4.2 Implementasi

```go
// interna;/handler/batch.go

// MSETNX key value [key value ...]
func (h *Handler[V]) MSetNX(args []string) server.RESPValue {
	if len(args) < 2 {
		return server.RESPValue{
			Type: server.Error,
			Str:  "ERR wrong number of arguments for 'msetnx' command",
		}
	}

	if len(args)%2 != 0 {
		return server.RESPValue{
			Type: server.Error,
			Str:  "ERR wrong number of arguments for 'msetnx' command",
		}
	}

	// Check if any key exists
	for i := 0; i < len(args); i += 2 {
		key := args[i]
		if exists := h.cache.HasKey(key); exists {
			return server.RESPValue{
				Type: server.Integer,
				Int:  0,
			}
		}
	}

	// Set all keys
	for i := 0; i < len(args); i += 2 {
		key := args[i]
		value := args[i+1]
		h.cache.Set(key, value, 0)
	}

	if h.persistenceMgr != nil {
		go h.persistenceMgr.LogCommand("MSETNX", args...)
	}

	return server.RESPValue{
		Type: server.Integer,
		Int:  1,
	}
}
```

### 10.4.3 Testing

```bash

redis-cli -h localhost -p 6378

# Clean up
127.0.0.1:6378> DEL key1 key2
(integer) 2

# MSETNX - semua key baru
127.0.0.1:6378> MSETNX key1 value1 key2 value2
(integer) 1

# MSETNX - ada key yang sudah ada
127.0.0.1:6378> MSETNX key2 value2 key3 value3
(integer) 0

# Verify: key3 tidak tersimpan
127.0.0.1:6378> GET key3
(nil)
```

## 10.5 Pipeline - Mengirim Multiple Commands Sekaligus

### 10.5.1 Konsep Pipeline

Pipeline memungkinkan client mengirim multiple commands sekaligus tanpa menunggu response setiap command.

```text

┌─────────────────────────────────────────────────────────────────┐
│              PIPELINE CONCEPT                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ❌ Tanpa Pipeline: Send → Wait → Response → Send → Wait...    │
│                                                                 │
│   Client ──▶ SET key1 v1 ──▶ Server ──▶ +OK ──▶ Client          │
│   Client ──▶ SET key2 v2 ──▶ Server ──▶ +OK ──▶ Client          │
│   Client ──▶ GET key1     ──▶ Server ──▶ "v1" ──▶ Client        │
│                                                                 │
│   ⏱️ 3 round-trip                                               │
│                                                                 │
│   ✅ Dengan Pipeline: Send All → Receive All                    │
│                                                                 │
│   Client ──▶ [SET key1 v1][SET key2 v2][GET key1] ──▶ Server    │
│              └──────────────────────────────────┘               │
│   Server ──▶ [+OK][+OK]["v1"] ──▶ Client                        │
│                                                                 │
│   ⏱️ 1 round-trip                                               │
└─────────────────────────────────────────────────────────────────┘
```

### 10.5.2 Implementasi Pipeline

```go
// internal/server/server.go

func (s *Server) handleConnection(conn net.Conn) {
	// .... existing code

	// Loop membaca perintah dari client
	for {
		// ... existing code

		// Wait for read OR shutdown
		select {
		case <-shutdown:
			s.logger.Printf("Shutdown interrupt during read from %s", c.remoteAddr)
			return
		case res := <-readCh:

			// ... existing code

			// handleConnection - bagian pipeline
			if res.respVal.Type == Array && len(res.respVal.Array) > 0 {
				// Cek apakah ini pipeline (array of arrays)
				// Pipeline: [ [SET key value], [GET key] ]
				// Single command: [ SET key value ] (bukan array of arrays)
				if res.respVal.Array[0].Type == Array {
					// Ini pipeline!
					responses := s.processPipeline(res.respVal.Array)
					if s.config.WriteTimeout > 0 {
						conn.SetWriteDeadline(time.Now().Add(s.config.WriteTimeout))
					}
					for _, resp := range responses {
						conn.Write([]byte(EncodeRESP(resp)))
					}
					c.UpdateActivity()
					continue
				}
			}

			// Proses command
			response := s.processCommand(res.respVal)

			// Set write timeout
			if s.config.WriteTimeout > 0 {
				conn.SetWriteDeadline(time.Now().Add(s.config.WriteTimeout))
			}

			conn.Write([]byte(EncodeRESP(response)))
			c.UpdateActivity()
		}
	}
}

// Pipeline dengan error handling
func (s *Server) processPipeline(commands []RESPValue) []RESPValue {
	results := make([]RESPValue, len(commands))
	for i, cmd := range commands {
		// Process each command independently
		// One error doesn't affect others
		results[i] = s.processCommand(&cmd)
	}
	return results
}
```

### 10.5.4 Testing Pipeline

```bash

 printf $'*3\r\n$3\r\nSET\r\n$4\r\nkey1\r\n$6\r\nvalue1\r\n' | nc localhost 6378
+OK

printf $'*3\r\n$3\r\nSET\r\n$4\r\nkey1\r\n$6\r\nvalue1\r\n*2\r\n$3\r\nGET\r\n$4\r\nkey1\r\n' | nc localhost 6378
+OK
$6
value1
```

10.6 Performance Comparison
10.6.1 Benchmark Results

```text

┌─────────────────────────────────────────────────────────────────┐
│              PERFORMANCE COMPARISON                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Tanpa Pipeline: 100 GET commands                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Total time: 50ms                                       │   │
│   │  Throughput: 2,000 ops/sec                              │   │
│   │  Network overhead: 90%                                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   Dengan Pipeline: 100 GET commands                             │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  Total time: 5ms                                        │   │
│   │  Throughput: 20,000 ops/sec                             │   │
│   │  Network overhead: 10%                                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ❗ Pipeline 10x lebih cepat!                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.7.2 Kapan Pakai Pipeline?

| Skenario | Rekomendasi |
|----------|-------------|
| Banyak operasi kecil | ✅ Pipeline sangat efektif |
| Operasi besar (big keys) | ⚠️ Hati-hati memory |
| Network latency tinggi | ✅ Pipeline sangat membantu |
| Write-intensive | ✅ Pipeline meningkatkan throughput |
| Real-time response | ❌ Pipeline menambah latency |

## 10.8 Best Practices

### 10.8.1 Batch Size

Client split batch untuk operasi batch yang banyak

```go
// Rekomendasi: 50-100 commands per batch
const MaxBatchSize = 100

func processBatch(commands []string) {
    if len(commands) > MaxBatchSize {
        // Split menjadi beberapa batch
        for i := 0; i < len(commands); i += MaxBatchSize {
            end := i + MaxBatchSize
            if end > len(commands) {
                end = len(commands)
            }
            processBatch(commands[i:end])
        }
        return
    }
    // Process batch
}
```

### 10.8.2 Memory Considerations

Client perlu split mget untuk mempertimbangkan penggunaan memory

```go
// ❌ Jangan: Pipeline dengan response besar
// MGET key1 key2 ... key10000 → response bisa besar!

// ✅ Lakukan: Batch dengan limit
func MGetLimited(keys []string, limit int) {
    for i := 0; i < len(keys); i += limit {
        end := i + limit
        if end > len(keys) {
            end = len(keys)
        }
        mgetKeys(keys[i:end])
    }
}
```

### 10.8.3 Error Handling

Dalam pipeline, satu perintah yang menghasilkan error, tidak berdampak terhadap perintah lainnya.

```go

// ✅ Pipeline dengan error handling
func processPipeline(commands []Command) []Response {
    results := make([]Response, len(commands))
    for i, cmd := range commands {
        // Process each command independently
        // One error doesn't affect others
        results[i] = processCommand(cmd)
    }
    return results
}
```

## 10.9 Command Summary

| Command | Fungsi | Return |
|---------|--------|--------|
| MGET | Get multiple keys | Array of values |
| MSET | Set multiple keys | OK |
| MSETNX | Set if not exists | 1/0 |
| Pipeline | Multiple commands | Multiple responses |

## 10.10 Ringkasan

Yang Sudah Kita Pelajari:
- MGET - Get multiple keys in one command
- MSET - Set multiple keys in one command
- MSETNX - Set multiple keys if not exists
- Pipeline - Send multiple commands without waiting
- Performance - 10x faster for batch operations

### Perbandingan Sebelum vs Sesudah

| Aspek | Sebelum | Sesudah |
|-------|---------|---------|
| GET 100 keys | 100 round-trip | 1 round-trip |
| SET 100 keys | 100 round-trip | 1 round-trip |
| Throughput | 2,000 ops/sec | 20,000 ops/sec |
| Network Overhead | 90% | 10% |

Dengan batch operations, kita bisa meningkatkan performa. Selanjutnya, kita akan menambahkan fitur messaging.