# Bab 1: Pengenalan - Membangun Cache Server dari Nol

## 1.1 Pendahuluan

### 1.1.1 Mengapa Membangun Cache Server?

Di era modern, kecepatan akses data adalah segalanya. Aplikasi web, mobile, dan IoT membutuhkan response time dalam hitungan milidetik. Database tradisional (seperti PostgreSQL, MySQL) terlalu lambat untuk akses data yang sangat sering. Di sinilah cache server berperan.

```text

┌─────────────────────────────────────────────────────────────────┐
│              TANPA CACHE vs DENGAN CACHE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ❌ TANPA CACHE:                                               │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  User ──▶ API ──▶ Database ──▶ Disk I/O ──▶ 50ms        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ✅ DENGAN CACHE:                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  User ──▶ API ──▶ Cache ──▶ Memory ──▶ 1ms              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ❗ Cache 50x lebih cepat!                                     │
└─────────────────────────────────────────────────────────────────┘
```

### 1.1.2 Apa yang Akan Kita Bangun?

Buku ini akan memandu Anda membangun database Pendem (sebuah Redis-like cache server) dari nol menggunakan Go.

```text

┌─────────────────────────────────────────────────────────────────┐
│              PENDEM - Redis-like Cache Server                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   🏺 "Pendem" = Menyimpan di dalam tanah (Bahasa Jawa)          │
│                                                                 │
│   Fitur:                                                        │
│   ✅ RESP Protocol (Redis-compatible)                           │
│   ✅ TCP Server dengan concurrent connection                    │
│   ✅ In-memory storage with TTL                                 │
│   ✅ LRU Eviction                                               │
│   ✅ Sharding untuk parallel operations                         │
│   ✅ Persistence (RDB + AOF)                                    │
│   ✅ Data Structures (Hash, List, Set, Sorted Set)              │
│   ✅ Batch Operations (MGET, MSET, Pipeline)                    │
│   ✅ Transaction (MULTI, EXEC, WATCH)                           │
│   ✅ Pub/Sub & Queue.                                           │
│   ✅ Cluster.                                                   │
│   ✅ Monitoring.                                                │
└─────────────────────────────────────────────────────────────────┘
```

## 1.2 Redis: Cache Server yang Menginspirasi

### 1.2.1 Apa itu Redis?

Redis (REmote DIctionary Server) adalah in-memory data structure store yang digunakan sebagai database, cache, dan message broker.

```text

┌─────────────────────────────────────────────────────────────────┐
│              REDIS ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    CLIENT LAYER                         │   │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │   │
│   │  │ redis-cli│  │  App Go  │  │  App Py  │               │   │
│   │  └──────────┘  └──────────┘  └──────────┘               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    PROTOCOL LAYER                       │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │              RESP Protocol                        │  │   │
│   │  │  (REdis Serialization Protocol)                   │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    COMMAND LAYER                        │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │   │
│   │  │  String  │ │   Hash   │ │   List   │ │   Set    │    │   │
│   │  │ Commands │ │ Commands │ │ Commands │ │ Commands │    │   │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘    │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │   │
│   │  │ Sorted   │ │  Pub/Sub │ │  Script  │                 │   │
│   │  │ Set      │ │          │ │  (Lua)   │                 │   │
│   │  └──────────┘ └──────────┘ └──────────┘                 │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    STORAGE LAYER                        │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │              In-Memory Database                   │  │   │
│   │  │  ┌─────────────────────────────────────────────┐  │  │   │
│   │  │  │  Dict (Hash Table)  │  Dict (Hash Table)    │  │  │   │
│   │  │  └─────────────────────────────────────────────┘  │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    PERSISTENCE LAYER                    │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │        RDB (Snapshot)  │  AOF (Append Only File)  │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2.2 Karakteristik Redis

| Karakteristik | Keterangan |
|---------------|------------|
| In-Memory | Data disimpan di RAM (sangat cepat) |
| Single-threaded | Satu thread untuk semua operasi (sederhana) |
| Data Structures | Mendukung banyak tipe data |
| Persistence | RDB (snapshot) dan AOF (log) |
| Replication | Master-slave replication |
| High Availability | Redis Sentinel, Redis Cluster |

### 1.2.3 Mengapa Redis Populer?

1. Kecepatan - Operasi dalam microseconds
2. Sederhana - Mudah digunakan dan diimplementasikan
3. Fleksibel - Banyak tipe data dan use case
4. Community - Ekosistem besar dan mature

## 1.3 Desain Pendem

### 1.3.1 Filosofi Nama

"Pendem" (Bahasa Jawa) = "Menyimpan di dalam tanah"

```text

┌─────────────────────────────────────────────────────────────────┐
│              FILOSOFI "PENDEM"                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   🏺 Pendem = Menyimpan di dalam tanah                          │
│                                                                 │
│   Makna:                                                        │
│   • "Menyimpan" → Cache menyimpan data                          │
│   • "Di dalam" → Data di dalam memory                           │
│   • "Tanah" → Fondasi yang kokoh (storage)                      │
│                                                                 │
│   "Pendem" = Data disimpan dengan aman, kokoh,                  │
│   seperti menyimpan harta karun di dalam tanah.                 │
│                                                                 │
│   ❗ TAPI: Data di cache adalah "harta karun"                   │
│   yang siap diambil kembali dengan cepat!                       │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3.2 Arsitektur Pendem

```text

┌─────────────────────────────────────────────────────────────────┐
│                    PENDEM ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    TCP SERVER                           │   │
│   │  • Concurrent connection handling                       │   │
│   │  • Graceful shutdown                                    │   │
│   │  • Connection pooling                                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    RESP PARSER                          │   │
│   │  • Simple Strings (+)                                   │   │
│   │  • Errors (-)                                           │   │
│   │  • Integers (:)                                         │   │
│   │  • Bulk Strings ($)                                     │   │
│   │  • Arrays (*)                                           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    COMMAND HANDLERS                     │   │
│   │  ┌───────┬───────┬───────┬───────┬───────┬───────┐      │   │
│   │  │PING   │GET    │SET    │DEL    │TTL    │EXISTS │      │   │
│   │  ├───────┼───────┼───────┼───────┼───────┼───────┤      │   │
│   │  │HSET   │HGET   │HDEL   │HLEN   │HKEYS  │HVALS  │      │   │
│   │  ├───────┼───────┼───────┼───────┼───────┼───────┤      │   │
│   │  │LPUSH  │RPUSH  │LPOP   │RPOP   │LRANGE │LLEN   │      │   │
│   │  ├───────┼───────┼───────┼───────┼───────┼───────┤      │   │
│   │  │SADD   │SREM   │SMEMBERS│SISMEM│SCARD  │       │      │   │
│   │  ├───────┼───────┼───────┼───────┼───────┼───────┤      │   │
│   │  │ZADD   │ZRANGE │ZREM   │ZCARD  │ZSCORE │       │      │   │
│   │  ├───────┼───────┼───────┼───────┼───────┼───────┤      │   │
│   │  │MGET   │MSET   │MSETNX │MULTI  │EXEC   │WATCH  │      │   │
│   │  └───────┴───────┴───────┴───────┴───────┴───────┘      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    STORAGE ENGINE                       │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │              SHARDED CACHE                        │  │   │
│   │  │  ┌──────┬──────┬──────┬──────┬──────┬──────┐      │  │   │
│   │  │  │ S0   │ S1   │ S2   │ S3   │ S4   │ S5   │      │  │   │
│   │  │  │ LRU  │ LRU  │ LRU  │ LRU  │ LRU  │ LRU  │      │  │   │
│   │  │  └──────┴──────┴──────┴──────┴──────┴──────┘      │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   │                                                         │   │
│   │  • Hash, List, Set, Sorted Set                          │   │
│   │  • LRU Eviction                                         │   │
│   │  • TTL Support                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    PERSISTENCE                          │   │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────┐            │   │
│   │  │   JSON    │  │   RDB     │  │   AOF     │            │   │
│   │  │ Snapshot  │  │  Binary   │  │   Log     │            │   │
│   │  └───────────┘  └───────────┘  └───────────┘            │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3.3 Tech Stack

| Komponen | Teknologi | Alasan |
|----------|-----------|--------|
| Bahasa | Go | Concurrent, cepat, sederhana |
| Networking | net package | Built-in, powerful |
| Protocol | RESP | Redis-compatible |
| Storage | Map + container/list | Fast, flexible |
| Concurrency | Goroutine + sync | Scalable |

## 1.4 Persiapan

### 1.4.1 Prasyarat

Sebelum memulai, pastikan Anda memiliki:

```text
✅ Go 1.21 atau lebih tinggi
✅ Basic Go (struct, interface, goroutine, channel)
✅ Pemahaman TCP/IP
✅ Command line / terminal
✅ Text editor / IDE (VS Code, GoLand, dll)
```

### 1.4.2 Install Go

```bash

# Download Go
# https://golang.org/dl/

# Cek versi
go version
# Output: go version go1.21.0 darwin/amd64

# Setup workspace
mkdir -p ~/go/src
export GOPATH=~/go
export PATH=$PATH:$GOPATH/bin
```

### 1.4.3 Struktur Project

```text

pendem/
├── cmd/
│   └── main.go                 # Entry point
├── internal/
│   ├── config/                 # Configuration
│   │   └── config.go
│   ├── engine/                 # Storage engine
│   │   ├── cache.go
│   │   ├── item.go
│   │   ├── lru.go
│   │   ├── shard.go
│   │   ├── hash.go
│   │   ├── list.go
│   │   ├── set.go
│   │   └── sortedset.go
│   ├── handler/                # Command handlers
│   │   ├── handler.go
│   │   ├── hash.go
│   │   ├── list.go
│   │   ├── set.go
│   │   └── sortedset.go
│   ├── persistence/            # Persistence
│   │   ├── manager.go
│   │   ├── json.go
│   │   ├── rdb.go
│   │   └── aof.go
│   └── server/                 # TCP Server
│       ├── server.go
│       ├── connection.go
│       └── resp.go
├── go.mod
├── go.sum
└── pendem.conf                 # Config file
```

### 1.4.4 Init Go Module

```bash

# Buat direktori project
mkdir -p ~/go/src/pendem
cd ~/go/src/pendem

# Init Go module
go mod init pendem

# File go.mod akan terbentuk:
# module pendem
# go 1.21
```

### 1.4.5 Tools yang Berguna

```bash

# Redis CLI (untuk testing)
brew install redis  # Mac
sudo apt install redis-tools  # Ubuntu

# Netcat (untuk testing protocol)
# Sudah tersedia di Mac/Linux

# Talnet (untuk testing)
# Sudah tersedia di Mac/Linux
```

### 1.4.6 Hello World - Server Sederhana

Sebelum masuk ke detail, mari buat server sederhana:

```go

// cmd/main.go
package main

import (
    "fmt"
    "net"
)

func main() {
    // Listen on port 6378
    listener, err := net.Listen("tcp", ":6378")
    if err != nil {
        panic(err)
    }
    defer listener.Close()

    fmt.Println("Pendem server listening on :6378")

    for {
        conn, err := listener.Accept()
        if err != nil {
            fmt.Println("Accept error:", err)
            continue
        }

        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close()
    fmt.Printf("New connection from %s\n", conn.RemoteAddr())

    // Simple response
    conn.Write([]byte("+Hello from Pendem!\r\n"))
}
```

### 1.4.7 Test Server

```bash

# Terminal 1: Jalankan server
go run cmd/main.go
# Output: Pendem server listening on :6378

# Terminal 2: Test dengan telnet
telnet localhost 6378
# Output: Hello from Pendem!

# Atau dengan nc
echo "" | nc localhost 6378
# Output: +Hello from Pendem!
```

## 1.5 Roadmap Pembelajaran

```text
┌─────────────────────────────────────────────────────────────────┐
│                    LEARNING ROADMAP                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   📚 BAB 1: Pengenalan                                          │
│   └─ Arsitektur, desain, persiapan                              │
│                                                                 │
│   📚 BAB 2: Golang Fundamental                                  │
│   └─ map, type parameter, container/list, sync Mutex            │
│                                                                 │
│   📚 BAB 3: Socket Programming                                  │
│   └─ TCP Server, connection handling                            │
│                                                                 │
│   📚 BAB 4: RESP Protocol                                       │
│   └─ Redis Serialization Protocol                               │
│                                                                 │
│   📚 BAB 5: In-Memory Storage                                   │
│   └─ Cache engine, TTL, cleanup                                 │
│                                                                 │
│   📚 BAB 6: LRU Eviction                                        │
│   └─ Least Recently Used eviction                               │
│                                                                 │
│   📚 BAB 7: Sharding                                            │
│   └─ Concurrent access, parallelism                             │
│                                                                 │
│   📚 BAB 8: Persistence                                         │
│   └─ JSON, RDB, AOF                                             │
│                                                                 │
│   📚 BAB 9: Data Structures                                     │
│   └─ Hash, List, Set, Sorted Set                                │
│                                                                 │
│   📚 BAB 10: Batch Operations                                   │
│   └─ MGET, MSET, Pipeline                                       │
│                                                                 │
│   📚 BAB 11: Transaction                                        │
│   └─ MULTI, EXEC, WATCH                                         │
│                                                                 │
│   📚 BAB 12: Pub/Sub                                            │
│   └─ Publish/Subscribe messaging                                │
│                                                                 │
│   📚 BAB 13: Monitoring                                         │
│   └─ Metrics, logging, observability                            │
│                                                                 │
│   📚 BAB 14: Cluster                                            │
│   └─ cluster, service discovery, gossip protocol                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 1.6 Command yang Akan Dibangun

### String Commands

```text

SET key value [EX seconds]
GET key
DEL key [key ...]
EXISTS key [key ...]
TTL key
APPEND key value
STRLEN key
```

### Hash Commands

```text

HSET key field value [field value ...]
HGET key field
HGETALL key
HDEL key field [field ...]
HLEN key
HEXISTS key field
HKEYS key
HVALS key
```

### List Commands

```text

LPUSH key value [value ...]
RPUSH key value [value ...]
LPOP key
RPOP key
LRANGE key start stop
LLEN key
```

### Set Commands

```text

SADD key member [member ...]
SREM key member [member ...]
SMEMBERS key
SISMEMBER key member
SCARD key
```

### Sorted Set Commands

```text

ZADD key score member [score member ...]
ZRANGE key start stop [WITHSCORES]
ZREM key member [member ...]
ZCARD key
ZSCORE key member
```

### Batch Commands

```text

MGET key [key ...]
MSET key value [key value ...]
MSETNX key value [key value ...]
```

### Transaction Commands

```text

MULTI
EXEC
DISCARD
WATCH key [key ...]
UNWATCH
```

## 1.7 Ringkasan

Yang Sudah Kita Pelajari:

1. Mengapa Cache Server
- Kecepatan akses data
- Mengurangi beban database
- Response time milidetik

2. Redis sebagai Inspirasi
- In-memory data store
- RESP protocol
- Multiple data structures

3. Desain Pendem
- Nama: "Menyimpan di dalam tanah"
- Arsitektur layered
- Tech stack: Go + net

4. Persiapan
- Go environment
- Project structure
- Hello world server

Setalah memahami apa yang akan dibangun, kita bahas fundamental pemrograman golang yang akan masif digunakan di project ini.