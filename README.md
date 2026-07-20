# Build Your Own Redis with Go

Panduan Software Engineer Membangun In-Memory Database dengan Golang

>    🏺 "Pendem" - Sebuah kata dari bahasa Jawa yang berarti "menyimpan di dalam tanah". Seperti namanya, Pendem adalah cache server yang menyimpan data dengan kokoh, aman, dan cepat di dalam memory.

## 📖 Tentang Buku Ini

Buku ini adalah panduan langkah demi langkah untuk membangun Redis-like cache server dari awal menggunakan bahasa Go. Dimulai dari socket programming, RESP protocol, hingga fitur-fitur canggih seperti persistence, sharding, dan cluster support.

Hasil akhir dari buku ini adalah **Pendem** - sebuah cache engine production-ready yang bisa Anda gunakan, modifikasi, dan pelajari.

## 🎯 Target Pembaca

- Software Engineer yang ingin memahami cara kerja cache database
- Go Developer yang ingin belajar network programming
- Backend Engineer yang ingin membangun sistem terdistribusi
- Mahasiswa yang ingin belajar arsitektur database

Prasyarat:
- Dasar-dasar bahasa Go (struct, interface, goroutine, channel)
- Pemahaman dasar networking (TCP/IP)
- Familiar dengan command line

## 📚 Daftar Isi

| Bab | Topik | Deskripsi |
|-----|-------|-----------|
| Bab 1 | Pengenalan (coming soon) | Arsitektur Redis, desain Pendem, dan persiapan |
| bab 2 | Golang Fundamental (coming soon) | Dasar - dasar golang |
| Bab 3 | Socket Programming | TCP server, connection handling, graceful shutdown |
| Bab 4 | RESP Protocol | Redis Serialization Protocol, parser & encoder |
| Bab 5 | In-Memory Storage | Cache engine, TTL, dan background cleanup |
| Bab 6 | LRU Eviction | Least Recently Used eviction policy |
| Bab 7 | Sharding | Concurrent access, multi-core utilization |
| Bab 8 | Persistence | JSON snapshot, RDB, AOF, dan recovery |
| Bab 9 | Data Structures | Hash, List, Set, dan Sorted Set |
| Bab 10 | Batch Operations | MGET, MSET, Pipeline |
| Bab 11 | Transaction (coming soon) | MULTI, EXEC, WATCH |
| Bab 12 | Pub/Sub & Queue (coming soon) | Real-time messaging dan task distribution |
| Bab 13 | Clustering (coming soon) | CLustering support, service discovery, gossiup protocol, etc |
| Bab 14 | Monitoring (coming soon) | Metrics, logging, dan observability |

## 🏗️ Arsitektur Pendem

```text
┌─────────────────────────────────────────────────────────────────┐
│                         PENDEM                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    TCP Server                           │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │              RESP Protocol Parser                 │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Command Handlers                      │   │
│   │  PING │ GET │ SET │ DEL │ TTL │ MEMORY │ POLICY         │   │
│   │  HSET │ HGET │ LPUSH │ RPUSH │ SADD │ ZADD │ ...        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  Sharded Cache                          │   │
│   │  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐     │   │
│   │  │ S0   │ S1   │ S2   │ S3   │ S4   │ S5   │ S6   │     │   │
│   │  │ LRU  │ LRU  │ LRU  │ LRU  │ LRU  │ LRU  │ LRU  │     │   │
│   │  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   Persistence                           │   │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────┐            │   │
│   │  │   JSON    │  │   RDB     │  │   AOF     │            │   │
│   │  │ Snapshot  │  │  Binary   │  │   Log     │            │   │
│   │  └───────────┘  └───────────┘  └───────────┘            │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## 🗺️ Roadmap Pembelajaran

```text
┌─────────────────────────────────────────────────────────────────┐
│                    LEARNING ROADMAP                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   📚 CORE FUNDAMENTALS                                          │
│   ├─ Bab 3: Socket Programming                                  │
│   ├─ Bab 4: RESP Protocol                                       │
│   └─ Bab 5: In-Memory Storage                                   │
│                                                                 │
│   📚 ADVANCED FEATURES                                          │
│   ├─ Bab 6: LRU Eviction                                        │
│   ├─ Bab 7: Sharding                                            │
│   └─ Bab 8: Persistence                                         │
│                                                                 │
│   📚 DATA STRUCTURES & PERFORMANCE                              │
│   ├─ Bab 9: Data Structures                                     │
│   ├─ Bab 10: Batch Operations                                   │
│   └─ Bab 11: Transaction                                        │
│                                                                 │
│   📚 REAL-TIME & DISTRIBUTED                                    │
│   ├─ Bab 12: Data Structures                                    │
│   └─ Bab 12: Clustering Support                                 │
│                                                                 │
│   🚀 PENDEM IS COMPLETE!                                        │
└─────────────────────────────────────────────────────────────────┘
```

## 📁 Struktur Kode

```text

pendem/
├── cmd/
│   └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── engine/
│   │   ├── cache.go
│   │   ├── item.go
│   │   ├── lru.go
│   │   ├── shard.go
│   │   ├── hash.go
│   │   ├── list.go
│   │   ├── set.go
│   │   └── sortedset.go
│   ├── handler/
│   │   ├── handler.go
│   │   ├── hash.go
│   │   ├── list.go
│   │   ├── set.go
│   │   └── sortedset.go
│   ├── persistence/
│   │   ├── manager.go
│   │   ├── json.go
│   │   ├── rdb.go
│   │   └── aof.go
│   └── server/
│       ├── server.go
│       ├── connection.go
│       └── resp.go
├── test/
│   └── integration/
│       └── concurrency_test.go
├── go.mod
├── go.sum
└── README.md
```

## 🚀 Hasil Akhir: Pendem Database

Pendem adalah cache server yang dibangun dari buku ini. Dengan fitur:

| Fitur | Status |
|-------|--------|
| RESP Protocol | ✅ Full Redis-compatible |
| TCP Server | ✅ Concurrent with graceful shutdown |
| In-Memory Storage | ✅ TTL support |
| Eviction | ✅ Memory-aware (lru, lfu. ttl) |
| Sharding | ✅ 16 shards for parallelism |
| Persistence | ✅ JSON + RDB + AOF |
| Hash, List, Set, SortedSet | ✅ Complete implementation |
| Batch Operations | ✅ MGET, MSET, Pipeline |
| Config File | ✅ pendem.conf support |
| Transaction | ✅ MULTI, EXEC, WATCH |
| Real Time | ✅ Pub/sub & Queue |
| Cluster | ✅ Support Clustering |
| Monitoring |	✅ Metric & Observability |

## 📂 Repository

Source Code: [github.com/jacky-htg/pendem](https://github.com/jacky-htg/pendem)

## 📄 Lisensi

Buku ini dilisensikan di bawah [GNU GPL License](LICENSE).

## 📬 Kontak

- Author: Rijal Asepnugroho (Jacky Htg)
- GitHub: jacky-htg
- Repository: github.com/jacky-htg/pendem

🏺 Pendem - Cache Server yang Kokoh Seperti Tanah