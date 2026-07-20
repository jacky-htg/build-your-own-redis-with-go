# Bab 3: Socket Programming - Fondasi Komunikasi Jaringan

## 3.1 Pendahuluan: Memahami Socket Programming

Socket adalah abstraksi yang memungkinkan dua program berkomunikasi melalui jaringan. Dalam konteks `Pendem`, socket memungkinkan client (seperti redis-cli) untuk mengirim perintah dan menerima respons dari server cache kita.

>    📂 Kode Lengkap Bab Ini:
>    Seluruh kode yang dibahas di bab ini tersedia di GitHub:
>
>    🔗 [github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/03-socket-programming](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/03-socket-programming)


### 3.1.1 Model Komunikasi Client-Server

```text
┌─────────────────────────────────────────────────────────────────┐
│                      Model Client-Server                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐          TCP/IP          ┌──────────────┐    │
│   │   Client     │ ───────────────────────> │   Server     │    │
│   │  (redis-cli) │  1. Connect & Send       │   (Pendem)   │    │
│   │              │        Request           │              │    │
│   │   Port:      │                          │   Port:      │    │
│   │   Random     │ <─────────────────────── │   6379       │    │
│   │              │  2. Process & Send       │              │    │
│   │              │        Response          │              │    │
│   └──────────────┘                          └──────────────┘    │
│                                                                 │
│   Connection: 3-way handshake (SYN, SYN-ACK, ACK)               │
│   Data Flow:   Bidirectional (full-duplex)                      │
│   Reliability: TCP guarantees delivery & order                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.1.2 Mengapa TCP untuk Cache Server?

| Karakteristik | TCP | UDP |
|---------------|-----|-----|
| Reliable | ✅ | ❌ |
| Ordered Delivery | ✅ | ❌ |
| Connection-based | ✅ | ❌ |
| Flow Control | ✅ | ❌ |
| Overhead | Higher | Lower |

Redis dan Pendem menggunakan TCP karena:
1. Reliability - Setiap perintah harus sampai dengan benar
2. Ordering - Urutan perintah penting (contoh: SET lalu GET)
3. Error Recovery - Jika ada masalah, koneksi bisa dideteksi

## 3.2 Dasar-dasar Socket di Go

### 3.2.1 Package net

Go menyediakan package net yang sangat powerful untuk networking:

```go
import "net"
```

### 3.2.2 Tipe Data Penting

```go
type Listener interface {
    // Menerima koneksi baru
    Accept() (Conn, error)
    
    // Menutup listener
    Close() error
    
    // Mendapatkan alamat jaringan
    Addr() Addr
}

type Conn interface {
    // Baca data dari connection
    Read(b []byte) (n int, err error)
    
    // Tulis data ke connection
    Write(b []byte) (n int, err error)
    
    // Tutup connection
    Close() error
    
    // Mendapatkan alamat lokal
    LocalAddr() Addr
    
    // Mendapatkan alamat remote
    RemoteAddr() Addr
}
```

### 3.2.3 Anatomi TCP Server Sederhana

```go
package main

import (
    "fmt"
    "net"
)

func main() {
    // STEP 1: Membuat listener
    listener, err := net.Listen("tcp", ":8080")
    if err != nil {
        panic(err)
    }
    defer listener.Close()
    
    fmt.Println("Server listening on :8080")
    
    // STEP 2: Loop menerima koneksi
    for {
        conn, err := listener.Accept()
        if err != nil {
            fmt.Println("Accept error:", err)
            continue
        }
        
        // STEP 3: Handle setiap koneksi
        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close()  // Pastikan selalu close
    
    // Buffer untuk membaca data
    buffer := make([]byte, 1024)
    
    // Baca data dari client
    n, err := conn.Read(buffer)
    if err != nil {
        fmt.Println("Read error:", err)
        return
    }
    
    // Proses data
    data := buffer[:n]
    fmt.Printf("Received: %s\n", data)
    
    // Kirim response
    response := []byte("Hello from server!\n")
    conn.Write(response)
}
```

### 3.2.4 Testing dengan Telnet

```bash
telnet localhost 8080
Trying ::1...
Connected to localhost.
Escape character is '^]'.
halo
Hello from server!
Connection closed by foreign host.
```

### 3.2.5 Testing dengan nc (netcat)

```bash
echo "PING" | nc localhost 8080 
Hello from server!
```

## 3.3 Implementasi TCP Server untuk Pendem

### 3.3.1 Struktur Folder

```text
pendem/
├── go.mod
├── cmd/
│   └── main.go
└── internal/
    └── server/
        └── server.go
```

```bash
go mod init pendem
```

### 3.3.2 Struktur Server

```go
// internal/server/server.go
package server

import (
	"log"
	"net"
	"os"
)

type Server struct {
	addr     string
	listener net.Listener // Listener untuk menerima koneksi
	logger   *log.Logger
	handlers map[string]CommandHandler // Handler untuk command (sederhana)
}

// CommandHandler adalah fungsi untuk menangani perintah
type CommandHandler func(args []string) string

// Konstruktior: NewServer membuat instance server baru
func NewServer(addr string) *Server {
	s := &Server{
		addr:     addr,
		logger:   log.New(os.Stdout, "[SERVER] ", log.LstdFlags),
		handlers: make(map[string]CommandHandler),
	}

	return s
}
```

### 3.3.3 Memulai Server

```go
// internal/server/server.go
func (s *Server) Start() error {
	listener, err := net.Listen("tcp", s.addr)
	if err != nil {
		return fmt.Errorf("failed to start server: %w", err)
	}
	s.listener = listener

	s.logger.Printf("Server listening on %s", s.addr)

	for {
		// Terima koneksi
		conn, err := s.listener.Accept()
		if err != nil {
			s.logger.Printf("Accept error: %v", err)
			continue
		}

		go s.handleConnection(conn)
	}
}
```

### 3.3.4 Menangani Koneksi

```go
// internal/server/server.go
func (s *Server) handleConnection(conn net.Conn) {
	// Pastikan connection ditutup saat function selesai
	defer conn.Close()

	// Log koneksi baru
	remoteAddr := conn.RemoteAddr().String()
	s.logger.Printf("New connection from %s", remoteAddr)

	// Buat reader untuk membaca data
	reader := bufio.NewReader(conn)

	// Loop membaca perintah dari client
	for {
		// Baca satu baris (sampai newline)
		line, err := reader.ReadString('\n')
		if err != nil {
			// EOF berarti client disconnect
			if err == io.EOF {
				s.logger.Printf("Client %s disconnected", remoteAddr)
			} else {
				s.logger.Printf("Read error from %s: %v", remoteAddr, err)
			}
			return
		}

		// Trim newline dan carriage return
		line = strings.TrimRight(line, "\r\n")

		// Log command (untuk debug)
		s.logger.Printf("Command from %s: %s", remoteAddr, line)

		// Proses command
		response := s.processCommand(line)

		// Kirim response (tambah newline)
		_, err = conn.Write([]byte(response + "\n"))
		if err != nil {
			s.logger.Printf("Write error to %s: %v", remoteAddr, err)
			return
		}
	}
}
```

### 3.3.5 Memproses Perintah

```go
// internal/server/server.go
// processCommand memproses perintah dari client
// Versi sederhana: command dan args dipisahkan oleh spasi
func (s *Server) processCommand(line string) string {
	// Trim whitespace
	line = strings.TrimSpace(line)
	if line == "" {
		return "ERR empty command"
	}

	// Parse command dan args (sederhana)
	parts := strings.Fields(line)
	if len(parts) == 0 {
		return "ERR invalid command"
	}

	cmd := strings.ToUpper(parts[0])
	args := parts[1:]

	// Cari handler
	handler, exists := s.handlers[cmd]
	if !exists {
		return fmt.Sprintf("ERR unknown command '%s'", cmd)
	}

	// Eksekusi handler
	return handler(args)
}
```

### 3.3.6 Registrasi Handler (Routing)

```go
// internal/server/server.go
// RegisterHandler mendaftarkan handler untuk perintah
func (s *Server) RegisterHandler(cmd string, handler CommandHandler) {
	s.handlers[strings.ToUpper(cmd)] = handler
}
```

### 3.3.7 Main

```go
// cmd/main.go
package main

import (
	"fmt"
	"log"
	"pendem/server"
)

func main() {
	srv := server.NewServer(":6379")

	srv.RegisterHandler("PING", func(args []string) string {
		if len(args) > 0 {
			return args[0]
		}
		return "PONG"
	})

	fmt.Println("╔═══════════════════════════════════════════════════════╗")
	fmt.Println("║                     P E N D E M                       ║")
	fmt.Println("║              Simple Cache Server in Go                ║")
	fmt.Println("╠═══════════════════════════════════════════════════════╣")
	fmt.Printf("║  Address			: %-22s║\n", "0.0.0.0:6379")
	fmt.Println("╚═══════════════════════════════════════════════════════╝")
	fmt.Println()

	if err := srv.Start(); err != nil {
		log.Fatalf("Server error: %v", err)
	}
}
```

### 3.3.8 Menjalankan Server

```bash
go run cmd/main.go
```

### 3.3.9 Testing dengan Telnet

```bash
 telnet localhost 6379
Trying ::1...
Connected to localhost.
Escape character is '^]'.
ping
PONG
^]
telnet> quit
Connection closed.
```

### 3.3.9 Testing dengan nc (netcat)

```bash
echo "PING" | nc localhost 6379
PONG
```

## 3.4 Konfigurasi Server

Saat ini server Pendem belum punya konfigurasi apapun, namun ke depannya akan ada beberapa konfigurasi. Kita akan menyiapkannya. Pada konstruktor NewServer() kita berikan DefaultConfig, jika ada config khusus, gunakan konstruktor NewServerWithConfig().

```go
// internal/server/server.go

// ServerConfig berisi konfigurasi server
type ServerConfig struct {
}

// DefaultConfig mengembalikan konfigurasi default
func DefaultConfig() ServerConfig {
	return ServerConfig{}
}

type Server struct {
	addr     string
	listener net.Listener // Listener untuk menerima koneksi
	logger   *log.Logger
	config   ServerConfig
	handlers map[string]CommandHandler // Handler untuk command (sederhana)
}

// Konstruktior: NewServer membuat instance server baru
func NewServer(addr string) *Server {
	s := &Server{
		addr:     addr,
		logger:   log.New(os.Stdout, "[SERVER] ", log.LstdFlags),
		config:   DefaultConfig(),
		handlers: make(map[string]CommandHandler),
	}

	return s
}

// NewServerWithConfig membuat server dengan konfigurasi kustom
func NewServerWithConfig(addr string, config ServerConfig) *Server {
	s := NewServer(addr)
	s.config = config
	return s
}
```

Ubah pemanggilan NewServer() di main dengan NewServerWIthConfig()

```go
// main.go
func main() {
	config := server.ServerConfig{}
	srv := server.NewServerWithConfig(":6379", config)

	srv.RegisterHandler("PING", func(args []string) string {
		if len(args) > 0 {
			return args[0]
		}
		return "PONG"
	})

	fmt.Println("╔═══════════════════════════════════════════════════════╗")
	fmt.Println("║                     P E N D E M                       ║")
	fmt.Println("║              Simple Cache Server in Go                ║")
	fmt.Println("╠═══════════════════════════════════════════════════════╣")
	fmt.Printf("║  Address			: %-22s║\n", "0.0.0.0:6379")
	fmt.Println("╚═══════════════════════════════════════════════════════╝")
	fmt.Println()

	if err := srv.Start(); err != nil {
		log.Fatalf("Server error: %v", err)
	}
}
```

## 3.5 Menangani Koneksi

Koneksi perlu dibatasi dengan berbagai alasan :
1. Resource Protection. Setiap koneksi consumes: memory (buffer), file descriptor, CPU time. Pembatasan koneksi dilakukan untuk mencegah server crash karena kehabisan resource (OOM, too many open files), serta untuk menjaga stabilitas server.
2. Quality of Service. Pembatasan koneksi akan memastikan client yang terhubung tetap mendapat performa baik dan mencegah degradasi performa karena terlalu banyak koneksi.
3. Anti-DoS (Sederhana). Mencegah serangan yang mencoba membuka ribuan koneksi serta membatasi dampak serangan.

```go
// internal/server/server.go

// .... existing kode

// ServerConfig berisi konfigurasi server
type ServerConfig struct {
	MaxConnections int // Maksimum koneksi simultan
}

// DefaultConfig mengembalikan konfigurasi default
func DefaultConfig() ServerConfig {
	return ServerConfig{
		MaxConnections: 10000,
	}
}

type Server struct {
	addr       string
	listener   net.Listener // Listener untuk menerima koneksi
	logger     *log.Logger
	config     ServerConfig
	handlers   map[string]CommandHandler // Handler untuk command (sederhana)
	activeConn int32                     // Counter aktif koneksi (atomic)
}

func NewServer(addr string) *Server {
	s := &Server{
		addr:       addr,
		logger:     log.New(os.Stdout, "[SERVER] ", log.LstdFlags),
		config:     DefaultConfig(),
		handlers:   make(map[string]CommandHandler),
		activeConn: 0,
	}

	return s
}

func (s *Server) handleConnection(conn net.Conn) {
	// Pastikan connection ditutup saat function selesai
	defer func() {
		conn.Close()
		// Kurangi counter aktif koneksi
		atomic.AddInt32(&s.activeConn, -1)
		s.logger.Printf("Active connections: %d", atomic.LoadInt32(&s.activeConn))
	}()

	// Log koneksi baru
	remoteAddr := conn.RemoteAddr().String()
	s.logger.Printf("New connection from %s", remoteAddr)
	s.logger.Printf("Active connections: %d", atomic.LoadInt32(&s.activeConn))

	// Buat reader untuk membaca data
	reader := bufio.NewReader(conn)

	// Loop membaca perintah dari client
	for {
		// Baca satu baris (sampai newline)
		line, err := reader.ReadString('\n')
		if err != nil {
			// EOF berarti client disconnect
			if err == io.EOF {
				s.logger.Printf("Client %s disconnected", remoteAddr)
			} else {
				s.logger.Printf("Read error from %s: %v", remoteAddr, err)
			}
			return
		}

		// Trim newline dan carriage return
		line = strings.TrimRight(line, "\r\n")

		// Log command (untuk debug)
		s.logger.Printf("Command from %s: %s", remoteAddr, line)

		// Proses command
		response := s.processCommand(line)

		// Kirim response (tambah newline)
		_, err = conn.Write([]byte(response + "\n"))
		if err != nil {
			s.logger.Printf("Write error to %s: %v", remoteAddr, err)
			return
		}
	}
}

// Start memulai server
func (s *Server) Start() error {
	listener, err := net.Listen("tcp", s.addr)
	if err != nil {
		return fmt.Errorf("failed to start server: %w", err)
	}
	s.listener = listener

	s.logger.Printf("Server listening on %s", s.addr)
	s.logger.Printf("Max connections: %d", s.config.MaxConnections)

	for {
		// Terima koneksi
		conn, err := s.listener.Accept()
		if err != nil {
			s.logger.Printf("Accept error: %v", err)
			continue
		}

		// Cek apakah sudah mencapai batas maksimal koneksi
		currentConn := atomic.LoadInt32(&s.activeConn)
		if currentConn >= int32(s.config.MaxConnections) {
			// Tolak koneksi dengan pesan error
			s.logger.Printf("Max connections reached (%d), rejecting connection from %s",
				s.config.MaxConnections, conn.RemoteAddr().String())
			conn.Write([]byte("ERR server full, maximum connections reached\n"))
			conn.Close()
			continue
		}

		// Tambah counter aktif koneksi
		atomic.AddInt32(&s.activeConn, 1)

		go s.handleConnection(conn)
	}
}

// ... existing kode
```

### 3.5.1 Penjelasan Implementasi

Implementasi penanganan koneksi dilakukan dengan :

1. Atomic Counter: Menggunakan atomic.Int32 sebagai counter aktif koneksi. Ini lebih sederhana dan tidak perlu locking.
2. Validasi Sebelum Accept: Setelah menerima koneksi baru (listener.Accept()), periksa apakah activeConn sudah mencapai MaxConnections.
3. Tolak Koneksi: Jika sudah penuh, kirim pesan error ke client dan tutup koneksi.
4. Increment/Decrement. Tambah counter (+1) setelah koneksi diterima & kurangi counter (-1) di defer saat koneksi selesai
5. Logging: Tampilkan jumlah koneksi aktif untuk monitoring. 


### 3.5.2 Jalankan Aplikasi & Testing

```bash
go run cmd/main.go
```

Buka terminal lain, dan jalankan pengujian 

```bash
echo "PING" | nc localhost 6379
ERR server full, maximum connections reached
```

Error disebabkan kode di fungsi main tidak membuat konfigurasi maxConnection, sehingga maxCOnnection = 0. Ketika ada client yang mencoba membuat koneksi akan direject karena maksimal koneksi adalah nol. Ubah main.go untuk menambahkan konfigurasi maxConnection.

```go
func main() {
	config := server.ServerConfig{
		MaxConnections: 50_000,
	}
	srv := server.NewServerWithConfig(":6379", config)

	srv.RegisterHandler("PING", func(args []string) string {
		if len(args) > 0 {
			return args[0]
		}
		return "PONG"
	})

	fmt.Println("╔═══════════════════════════════════════════════════════╗")
	fmt.Println("║                     P E N D E M                       ║")
	fmt.Println("║              Simple Cache Server in Go                ║")
	fmt.Println("╠═══════════════════════════════════════════════════════╣")
	fmt.Printf("║  Address			: %-22s║\n", "0.0.0.0:6379")
	fmt.Println("╚═══════════════════════════════════════════════════════╝")
	fmt.Println()

	if err := srv.Start(); err != nil {
		log.Fatalf("Server error: %v", err)
	}
}
```

```bash
echo "PING" | nc localhost 6379
PONG
```

### 3.5.3 Perhitungan Maksimal Koneksi

Maksimal koneksi yang bisa dibuat tergantung pada beberapa hal :

1. Batas dari OS (File Descriptor)

```bash
# Cek batas maksimal file descriptor di Linux
ulimit -n
# Biasanya: 1024 (default) atau 65535 (jika sudah di-set)
```

Setiap koneksi TCP = 1 file descriptor, jadi:
- Default ulimit -n 1024 → maksimal ~1024 koneksi
- Setelah di-set ulimit -n 65535 → maksimal ~65535 koneksi

**Cara setting ulimit :**

```bash
# Cara setting ulimit (Linux/Mac)
ulimit -n 65535

# Agar permanen (di /etc/security/limits.conf)
* soft nofile 65535
* hard nofile 65535

# Untuk Docker
docker run --ulimit nofile=65535:65535 pendem
```

2. Memory per Koneksi

**Perhitungan kasar :**

```text
Goroutine stack    : ~4 KB
net.Conn overhead  : ~0.5 KB
bufio.Reader       : ~4 KB
Internal buffers   : ~4 KB
Lain-lain          : ~1 KB
-----------------------------------
Total per koneksi  : ~13.5 KB
```

Untuk memastikan berapa resource yang terpakai untuk 1 koneksi, bisa memasang kode :

```go
// internal/server/server.go
func (s *Server) PrintMemoryUsage() string {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)

	// Konversi ke MB dengan 2 desimal
	allocKB := float64(m.Alloc) / 1024
	totalAllocKB := float64(m.TotalAlloc) / 1024
	sysMB := float64(m.Sys) / 1024 / 1024

	return fmt.Sprintf(
		`{"alloc": "%.2f KB", "total_alloc": "%.2f KB", "sys": "%.2f MB", "num_gc": "%d"}`,
		allocKB,
		totalAllocKB,
		sysMB,
		m.NumGC,
	)
}
```

Di fungsi main register perintah MEMORY

```go
// cmd/main.go

func main() {
    // ... existing kode
    srv.RegisterHandler("MEMORY", func(args []string) string {
		if len(args) > 0 {
			return args[0]
		}

		return srv.PrintMemoryUsage()
	})
    // ... existing kode
}
```

Jalankan di terminal 1 :

```bash
echo "MEMORY" | nc localhost 6379
{"alloc": "165.51 KB", "total_alloc": "165.51 KB", "sys": "7.77 MB", "num_gc": "0"}
```

Buka terminal 2 :

```bash
telnet localhost 6379
Trying ::1...
Connected to localhost.
Escape character is '^]'.
```

Kembali ke termnial 1

```bash
echo "MEMORY" | nc localhost 6379
{"alloc": "175.23 KB", "total_alloc": "175.23 KB", "sys": "7.77 MB", "num_gc": "0"}
```

**Perhitungan : Memory per Koneksi = 9.72 KB**

```text
Selisih alloc = 175.23 - 165.51 = 9.72 KB
```

**Simulasi memory Usage**

| Koneksi | Memory (≈10 KB/conn) | Total Memory |
|---------|------------------------|--------------|
| 100 | 1 MB | Sangat aman |
| 1,000 | 10 MB | Aman |
| 10,000 | 100 MB | Masih OK |
| 50,000 | 500 MB | Cukup berat |
| 100,000 | 1 GB | Berat |

### 3.5.4 Tip Optimasi Buffer Size

```go
// Sebelum: default 4KB
reader := bufio.NewReader(conn)

// Sesudah: lebih hemat memory
reader := bufio.NewReaderSize(conn, 1024)  // 1KB
```

**Penjelasan:** Untuk server dengan banyak koneksi (10,000+), mengurangi buffer size dari 4KB ke 1KB menghemat 30MB memory!

## 3.6 Timeout Manajemen

Timeout management adalah salah satu aspek terpenting dalam membangun server network yang robust.

### 3.6.1 Jenis-jenis Timeout

#### a. Read Timeout

Batas waktu untuk membaca data dari client.

```go
// Client mengirim data secara perlahan
conn.SetReadDeadline(time.Now().Add(5 * time.Second))

// Jika dalam 5 detik tidak ada data yang terbaca:
// - Read() akan return error "i/o timeout"
// - Koneksi bisa ditutup
```

Contoh Kasus :

```text
// Client mengirim data 1MB, tapi sangat lambat (100 bytes/detik)
// Read timeout: 5 detik
// Data hanya terkirim 500 bytes dalam 5 detik
// → Timeout! Server tutup koneksi
```

Kapan terjadi:
- Client lambat mengirim data
- Client mengirim data dalam potongan-potongan kecil dengan delay
- Jaringan bermasalah (packet loss, latency tinggi)

#### b. Write Timeout

Batas waktu untuk menulis data ke client.

```go
// Server ingin mengirim response besar
conn.SetWriteDeadline(time.Now().Add(5 * time.Second))

// Jika dalam 5 detik tidak semua data terkirim:
// - Write() akan return error
// - Koneksi bisa ditutup
```

Contoh Kasus :

```text
// Server mengirim response 10MB ke client yang lambat
// Write timeout: 5 detik
// Hanya 2MB terkirim dalam 5 detik
// → Timeout! Server tutup koneksi
```

Kapan terjadi:
- Client lambat menerima data (buffer penuh)
- Client sudah disconnect tapi server belum tahu
- Jaringan lambat

#### c. Idle Timeout (Connection Timeout)

Batas waktu koneksi tidak melakukan aktivitas apapun.

```go
// Koneksi tidak mengirim/menerima data dalam 30 detik
// Server tutup koneksi secara otomatis
idleTimeout := 30 * time.Second
```

Contoh Kasus :

```text
// Client connect, tapi tidak pernah mengirim command
// Idle timeout: 30 detik
// Setelah 30 detik tidak ada aktivitas → koneksi ditutup
```

Kapan terjadi:
- Client connect tapi tidak pernah menggunakan koneksi
- Client hang atau crash
- Network split (koneksi mati tapi tidak terdeteksi)

### 3.6.2 Implementasi Timeout Manajemen

#### Deadline (Absolute Time)

```go
// Set absolute deadline
conn.SetReadDeadline(time.Now().Add(30 * time.Second))
conn.SetWriteDeadline(time.Now().Add(30 * time.Second))
// Atau set both
conn.SetDeadline(time.Now().Add(30 * time.Second))
```

Karakteristik:
- ✅ Absolute time (misal: jam 10:30:15)
- ✅ Reset setiap operasi
- ❌ Perlu di-reset manual setelah operasi berhasil

### Idle Timer (Relative)

```go
type Connection struct {
    conn      net.Conn
    idleTimer *time.Timer
    activity  chan bool
}

func (c *Connection) startIdleTimer(timeout time.Duration) {
    c.idleTimer = time.NewTimer(timeout)
    go func() {
        <-c.idleTimer.C
        c.conn.Close() // Idle timeout!
    }()
}

func (c *Connection) resetIdleTimer() {
    if !c.idleTimer.Stop() {
        <-c.idleTimer.C
    }
    c.idleTimer.Reset(c.idleTimeout)
}
```

### Hybrid (Deadline + Idle)

```go
func (s *Server) handleConnection(conn net.Conn) {
    // 1. Set read/write deadline
    conn.SetReadDeadline(time.Now().Add(s.config.ReadTimeout))
    conn.SetWriteDeadline(time.Now().Add(s.config.WriteTimeout))
    
    // 2. Start idle timer
    idleTimer := time.NewTimer(s.config.IdleTimeout)
    defer idleTimer.Stop()
    
    // 3. Channel untuk aktivitas
    activity := make(chan bool, 1)
    
    // 4. Goroutine monitor idle
    go func() {
        for {
            select {
            case <-idleTimer.C:
                conn.Close()
                return
            case <-activity:
                if !idleTimer.Stop() {
                    <-idleTimer.C
                }
                idleTimer.Reset(s.config.IdleTimeout)
            }
        }
    }()
    
    // 5. Main loop
    for {
        // Reset deadline untuk operasi baca
        conn.SetReadDeadline(time.Now().Add(s.config.ReadTimeout))
        
        line, err := reader.ReadString('\n')
        if err != nil {
            // Handle error (EOF, timeout, dll)
            return
        }
        
        // Signal activity
        select {
        case activity <- true:
        default:
        }
        
        // Process command...
        
        // Reset deadline untuk operasi tulis
        conn.SetWriteDeadline(time.Now().Add(s.config.WriteTimeout))
        conn.Write([]byte(response + "\n"))
    }
}
```

### Timeout Pitfalls

1. Lupa Reset Deadline

```go
// ❌ BURUK: Deadline hanya di-set sekali
conn.SetReadDeadline(time.Now().Add(5 * time.Second))

for {
    // Setelah 5 detik, semua operasi timeout
    line, _ := reader.ReadString('\n')  // ← Akan timeout setelah 5 detik
    // Process...
}
```

Seharusnya :

```go
// ✅ BAIK: Reset setiap iterasi
for {
    conn.SetReadDeadline(time.Now().Add(5 * time.Second))
    line, _ := reader.ReadString('\n')
    // Process...
}
```

2. Timeout Terlalu Kecil

```go
// ❌ BURUK: 100ms terlalu kecil untuk network
ReadTimeout: 100 * time.Millisecond

// ❌ BURUK: Client dengan latency 200ms akan selalu timeout
```

Seharusnya :

```go
// ✅ BAIK: Sesuaikan dengan kondisi network
ReadTimeout: 30 * time.Second   // Development
ReadTimeout: 5 * time.Second    // Production (fast network)
ReadTimeout: 60 * time.Second   // Production (slow network)
```

3. Timeout Terlalu Besar

```go
// ❌ BURUK: 10 menit terlalu lama
ReadTimeout: 10 * time.Minute

// Konsekuensi:
// - Malicious client bisa hold connection 10 menit
// - Resource terbuang sia-sia
// - DoS attack lebih mudah
```

Seharusnya :

```go
// ✅ BAIK: Balance antara performance dan security
ReadTimeout: 30 * time.Second  // Cukup untuk data normal
```

4. Mengabaikan Error Timeout

```go
// ❌ BURUK: Treat semua error sama
if err != nil {
    log.Printf("Error: %v", err)
    return  // Langsung close, padahal bisa jadi timeout saja
}
```

Seharusnya :

```go
// ✅ BAIK: Handle timeout dengan tepat
if err != nil {
    if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
        log.Printf("Timeout, but connection might still be usable")
        continue  // Bisa lanjut, atau close dengan proper message
    }
    // Error lain → close
    return
}
```

### Perbandingan Pendekatan Timeout

| Pendekatan | Kelebihan | Kekurangan | Kapan Pakai |
|------------|-----------|------------|-------------|
| **SetDeadline** | Sederhana, built-in | Perlu reset manual | Read/Write timeout |
| **Timer** | Bisa di-reset, fleksibel | Perlu goroutine | Idle timeout |
| **Channel** | Non-blocking | Lebih kompleks | Graceful shutdown |

## 3.7 Implementasi Timeout di Pendem

### 3.7.1 Read Timeout & Write Timeout

Menggunakan pendekatan absolut timeout :

```go
// internal/server/server.go

// ServerConfig berisi konfigurasi server
type ServerConfig struct {
	MaxConnections int // Maksimum koneksi simultan
	ReadTimeout    time.Duration
	WriteTimeout   time.Duration
}

// DefaultConfig mengembalikan konfigurasi default
func DefaultConfig() ServerConfig {
	return ServerConfig{
		MaxConnections: 1_0000,
		ReadTimeout:    30 * time.Second,
		WriteTimeout:   30 * time.Second,
	}
}

func (s *Server) handleConnection(conn net.Conn) {
	// Pastikan connection ditutup saat function selesai
	defer func() {
		conn.Close()
		// Kurangi counter aktif koneksi
		atomic.AddInt32(&s.activeConn, -1)
		s.logger.Printf("Active connections: %d", atomic.LoadInt32(&s.activeConn))
	}()

	// Log koneksi baru
	remoteAddr := conn.RemoteAddr().String()
	s.logger.Printf("New connection from %s", remoteAddr)
	s.logger.Printf("Active connections: %d", atomic.LoadInt32(&s.activeConn))

	// Buat reader untuk membaca data
	reader := bufio.NewReader(conn)

	// Loop membaca perintah dari client
	for {
		// Set read timeout
		if s.config.ReadTimeout > 0 {
			conn.SetReadDeadline(time.Now().Add(s.config.ReadTimeout))
		}

		// Baca satu baris (sampai newline)
		line, err := reader.ReadString('\n')
		if err != nil {
			// EOF berarti client disconnect
			if err == io.EOF {
				s.logger.Printf("Client %s disconnected", remoteAddr)
			} else if ne, ok := err.(net.Error); ok && ne.Timeout() {
				s.logger.Printf("Client %s timeout", remoteAddr)
			} else {
				s.logger.Printf("Read error from %s: %v", remoteAddr, err)
			}
			return
		}

		// Trim newline dan carriage return
		line = strings.TrimRight(line, "\r\n")

		// Log command (untuk debug)
		s.logger.Printf("Command from %s: %s", remoteAddr, line)

		// Proses command
		response := s.processCommand(line)

		// Set write timeout
		if s.config.WriteTimeout > 0 {
			conn.SetWriteDeadline(time.Now().Add(s.config.WriteTimeout))
		}

		// Kirim response (tambah newline)
		_, err = conn.Write([]byte(response + "\n"))
		if err != nil {
			if ne, ok := err.(net.Error); ok && ne.Timeout() {
				s.logger.Printf("Write timeout to %s", remoteAddr)
			} else {
				s.logger.Printf("Write error to %s: %v", remoteAddr, err)
			}
			return
		}
	}
}
```

### 3.7.2 Idle Timeout

Idle timeout bisa diterapkan dengan idle timer (relative), namun kali ini kita akan menggunakan pendekatan atomic yang bersifat nonblocking (tidak menggunakan channel). Buat file baru `server/connection.go` untuk mengatur setiap koneksi yang masuk.

```go
// internal/server/connection.go
package server

import (
	"log"
	"net"
	"sync"
	"sync/atomic"
	"time"
)

type Connection struct {
	conn         net.Conn
	lastActivity int64
	idleTimeout  time.Duration
	logger       *log.Logger
	remoteAddr   string
	done         chan struct{}
	mu           sync.Mutex // Melindungi operasi Close() dari race condition
	closed       bool
}

func (c *Connection) MonitorIdle() {
	ticker := time.NewTicker(1 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-c.done:
			return
		case <-ticker.C:
			lastActive := atomic.LoadInt64(&c.lastActivity)
			elapsed := time.Now().Unix() - lastActive

			if elapsed > int64(c.idleTimeout.Seconds()) {
				c.logger.Printf("Idle timeout for %s (idle for %d seconds)",
					c.remoteAddr, elapsed)
				c.Close()
				return
			}
		}
	}
}

func (c *Connection) Close() {
	c.mu.Lock()
	defer c.mu.Unlock()

	if !c.closed {
		c.closed = true
		close(c.done)
		c.conn.Close()
		c.logger.Printf("Connection closed: %s", c.remoteAddr)
	}
}

func (c *Connection) IsClosed() bool {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.closed
}

func (c *Connection) UpdateActivity() {
	atomic.StoreInt64(&c.lastActivity, time.Now().Unix())
}
```

Terapkan logic idle timeout di fungsi `handleConnection()`

```go
// internal/server/server.go

type ServerConfig struct {
	MaxConnections int // Maksimum koneksi simultan
	ReadTimeout    time.Duration
	WriteTimeout   time.Duration
	IdleTimeout    time.Duration // Timeout koneksi idle
}

// DefaultConfig mengembalikan konfigurasi default
func DefaultConfig() ServerConfig {
	return ServerConfig{
		MaxConnections: 1_0000,
		ReadTimeout:    30 * time.Second,
		WriteTimeout:   30 * time.Second,
		IdleTimeout:    60 * time.Second,
	}
}

func (s *Server) handleConnection(conn net.Conn) {
	c := &Connection{
		conn:        conn,
		remoteAddr:  conn.RemoteAddr().String(),
		idleTimeout: s.config.IdleTimeout,
		logger:      s.logger,
		done:        make(chan struct{}),
	}

	// Pastikan connection ditutup saat function selesai
	defer func() {
		c.Close()
		// Kurangi counter aktif koneksi
		atomic.AddInt32(&s.activeConn, -1)
		s.logger.Printf("Active connections: %d", atomic.LoadInt32(&s.activeConn))
	}()

	// Start idle monitor
	c.UpdateActivity()
	go c.MonitorIdle()

	// Log koneksi baru
	s.logger.Printf("New connection from %s", c.remoteAddr)
	s.logger.Printf("Active connections: %d", atomic.LoadInt32(&s.activeConn))

	// Buat reader untuk membaca data
	reader := bufio.NewReader(conn)

	// Loop membaca perintah dari client
	for {
		// Set read timeout
		if s.config.ReadTimeout > 0 {
			conn.SetReadDeadline(time.Now().Add(s.config.ReadTimeout))
		}

		// Baca satu baris (sampai newline)
		line, err := reader.ReadString('\n')
		if err != nil {
			// EOF berarti client disconnect
			if err == io.EOF {
				s.logger.Printf("Client %s disconnected", c.remoteAddr)
			} else if ne, ok := err.(net.Error); ok && ne.Timeout() {
				s.logger.Printf("Client %s timeout", c.remoteAddr)
			} else {
				s.logger.Printf("Read error from %s: %v", c.remoteAddr, err)
			}
			return
		}

		// Trim newline dan carriage return
		line = strings.TrimRight(line, "\r\n")

		// Log command (untuk debug)
		s.logger.Printf("Command from %s: %s", c.remoteAddr, line)

		// Proses command
		response := s.processCommand(line)

		// Set write timeout
		if s.config.WriteTimeout > 0 {
			conn.SetWriteDeadline(time.Now().Add(s.config.WriteTimeout))
		}

		// Kirim response (tambah newline)
		_, err = conn.Write([]byte(response + "\n"))
		if err != nil {
			if ne, ok := err.(net.Error); ok && ne.Timeout() {
				s.logger.Printf("Write timeout to %s", c.remoteAddr)
			} else {
				s.logger.Printf("Write error to %s: %v", c.remoteAddr, err)
			}
			return
		}
		// Update per-koneksi activity
		c.UpdateActivity()
	}
}
```

Tambahkan konfigurasi idle timeout di fungsi `main()`

```go
func main() {
	config := server.ServerConfig{
		MaxConnections: 50_000,
		ReadTimeout:    30 * time.Second,
		WriteTimeout:   10 * time.Second,
		IdleTimeout:    60 * time.Second,
	}
	srv := server.NewServerWithConfig(":6379", config)

	srv.RegisterHandler("PING", func(args []string) string {
		if len(args) > 0 {
			return args[0]
		}
		return "PONG"
	})

	srv.RegisterHandler("MEMORY", func(args []string) string {
		if len(args) > 0 {
			return args[0]
		}

		return srv.PrintMemoryUsage()
	})

	fmt.Println("╔═══════════════════════════════════════════════════════╗")
	fmt.Println("║                     P E N D E M                       ║")
	fmt.Println("║              Simple Cache Server in Go                ║")
	fmt.Println("╠═══════════════════════════════════════════════════════╣")
	fmt.Printf("║  Address			: %-22s║\n", "0.0.0.0:6379")
	fmt.Println("╚═══════════════════════════════════════════════════════╝")
	fmt.Println()

	if err := srv.Start(); err != nil {
		log.Fatalf("Server error: %v", err)
	}
}
```

Pengetesan idle timeout bisa dilakukan dengan mengubah konfigurasi idle timeout yang singkat, misal 5 detik. Buka telnet, beri beberapa perintah, kemudian biarkan idle selama 5 detik, maka koneksi akan diputus oleh server secara otomatis.

```bash
telnet localhost 6379
Trying ::1...
Connected to localhost.
Escape character is '^]'.
ping
PONG
memory
{"alloc": "167.23 KB", "total_alloc": "167.23 KB", "sys": "12.02 MB", "num_gc": "0"}
Connection closed by foreign host.
```

## 3.8 Graceful Shutdown

Graceful Shutdown adalah proses menghentikan server secara aman dan teratur, dimana server:

- ✅ Berhenti menerima koneksi baru
- ✅ Menunggu semua koneksi yang ada selesai diproses
- ✅ Membersihkan semua resource (koneksi, memory, file descriptor)
- ✅ Memberi tahu client bahwa server akan shutdown

**Analogi Sederhana**

Bayangkan sebuah restoran:
- Force Shutdown: Tutup restoran langsung, usir semua pelanggan yang sedang makan ❌
- Graceful Shutdown: Tutup pintu masuk, tunggu semua pelanggan selesai makan, baru tutup restoran ✅

**❌ Tanpa Graceful Shutdown (Force Shutdown)**

```go
// ❌ Buruk: Langsung terminate
os.Exit(0)
// atau
process.Kill()
```

Akibat:
- Client menerima connection reset error
- Data yang sedang ditransfer menjadi corrupt
- Resource tidak dibersihkan (memory leak)
- Client bingung karena koneksi tiba-tiba putus

**✅ Dengan Graceful Shutdown**

```bash
# Client melihat pesan shutdown
ERR server shutting down
# Atau koneksi selesai dengan normal
```

Keuntungan:
- Client tahu server akan shutdown
- Data tetap utuh
- Resource dibersihkan
- Monitoring tetap akurat

### 3.8.1 Komponen Graceful Shutdown

#### a. Quit Channel (Signal Stop)

```go
type Server struct {
    quit chan struct{}  // Channel untuk signal shutdown
}

// Saat shutdown
close(s.quit)  // Signal ke semua goroutine
```

Fungsi: Memberi tahu semua goroutine bahwa server akan shutdown.

#### b. WaitGroup (Menunggu Semua Selesai)

```go
type Server struct {
    wg sync.WaitGroup  // Menghitung goroutine aktif
}

// Saat goroutine mulai
s.wg.Add(1)

// Saat goroutine selesai
defer s.wg.Done()

// Saat shutdown
s.wg.Wait()  // Tunggu semua goroutine selesai
```

Fungsi: Server menunggu sampai semua koneksi selesai diproses.

#### c. Listener Close (Stop Menerima Koneksi)

```go
// Saat shutdown
s.listener.Close()  // Tutup listener

// Akibat: Accept() akan return error
conn, err := s.listener.Accept()
if err != nil {
    // Deteksi bahwa listener sengaja ditutup
    if s.closed {
        return nil  // Stop accept loop
    }
}
```

Fungsi: Server berhenti menerima koneksi baru.

#### d. Connection Close (Tutup Koneksi Aktif)

```go
func (c *Connection) Close() {
    if !c.closed {
        c.closed = true
        close(c.done)      // Hentikan idle monitor
        c.conn.Close()     // Tutup koneksi
    }
}
```

Fungsi: Membersihkan setiap koneksi saat shutdown.

### 3.8.2 Implementasi Step by Step

#### Step 1: Menambahkan Field untuk Shutdown

```go
type Server struct {
    // ... field lain
    wg   sync.WaitGroup  // WaitGroup untuk graceful shutdown
    quit chan struct{}   // Channel untuk signal shutdown
    mu   sync.Mutex      // Untuk protect listener close
    closed bool          // Flag untuk cek sudah closed
}

func NewServer(addr string) *Server {
    return &Server{
        // ... 
        quit:   make(chan struct{}),
        closed: false,
    }
}
```

#### Step 2: Menambahkan Connection ke WaitGroup

```go
func (s *Server) handleConnection(conn net.Conn) {
    // 1. Tambahkan ke WaitGroup
    s.wg.Add(1)
    // 2. Kurangi saat selesai
    defer s.wg.Done()

    // ... handling connection
}
```

Penjelasan:
- s.wg.Add(1): Server mencatat ada 1 koneksi aktif
- defer s.wg.Done(): Server mencatat koneksi sudah selesai
- s.wg.Wait(): Server menunggu semua koneksi selesai

#### Step 3: Cek Shutdown Signal di Loop & Intercept Shutdown di Idle Connection

**Mengapa Perlu Intercept Shutdown?**

Masalah: Ketika client dalam keadaan idle (tidak mengirim data), server akan blocking di reader.ReadString() menunggu data. Jika shutdown terjadi saat blocking, server tidak bisa keluar sampai timeout tercapai (bisa 30 detik!).

```go
// ❌ Masalah: Blocking di ReadString()
line, err := reader.ReadString('\n')  // ← Blocking!
// Shutdown signal datang, tapi kita stuck di sini
// Harus menunggu timeout 30 detik!
```

**Solusi:** Gunakan goroutine + channel untuk membaca data, sehingga kita bisa interrupt operasi baca saat shutdown.


```go
func (s *Server) handleConnection(conn net.Conn) {
    // ... setup connection

    // Channel untuk shutdown interrupt
	shutdown := make(chan struct{})
	go func() {
		<-s.quit
		c.Close() // Langsung close connection
		close(shutdown)
	}()

    for {
		// Cek shutdown signal
		select {
		case <-shutdown:
			s.logger.Printf("Shutting down, closing connection from %s", c.remoteAddr)
			return
		default:
		}

		// Set read timeout
		if s.config.ReadTimeout > 0 {
			conn.SetReadDeadline(time.Now().Add(s.config.ReadTimeout))
		}

		// Baca di goroutine terpisah dengan channel
		type readResult struct {
			line string
			err  error
		}
		readCh := make(chan readResult, 1)
		go func() {
			line, err := reader.ReadString('\n')
			readCh <- readResult{line, err}
		}()

		// Wait for read OR shutdown
		select {
		case <-shutdown:
			s.logger.Printf("Shutdown interrupt during read from %s", c.remoteAddr)
			return
		case res := <-readCh:

			if res.err != nil {
				// EOF berarti client disconnect
				if res.err == io.EOF {
					s.logger.Printf("Client %s disconnected", c.remoteAddr)
				} else if ne, ok := res.err.(net.Error); ok && ne.Timeout() {
					s.logger.Printf("Client %s timeout", c.remoteAddr)
				} else {
					s.logger.Printf("Read error from %s: %v", c.remoteAddr, res.err)
				}
				return
			}

			// Trim newline dan carriage return
			line := strings.TrimRight(res.line, "\r\n")

			// Log command (untuk debug)
			s.logger.Printf("Command from %s: %s", c.remoteAddr, line)

			// Proses command
			response := s.processCommand(line)

			// Set write timeout
			if s.config.WriteTimeout > 0 {
				conn.SetWriteDeadline(time.Now().Add(s.config.WriteTimeout))
			}

			// Kirim response (tambah newline)
			_, err := conn.Write([]byte(response + "\n"))
			if err != nil {
				if ne, ok := err.(net.Error); ok && ne.Timeout() {
					s.logger.Printf("Write timeout to %s", c.remoteAddr)
				} else {
					s.logger.Printf("Write error to %s: %v", c.remoteAddr, err)
				}
				return
			}
			// Update per-koneksi activity
			c.UpdateActivity()
		}
	}
}
```

Penjelasan:
- Setiap iterasi loop mengecek quit channel
- Jika channel closed, koneksi akan segera ditutup
- Client melihat koneksi tertutup dengan normal
- Buat Channel untuk Shutdown Signal
- Baca Data di Goroutine Terpisah, Kasus 1: shutdown signal datang → langsung return. Kasus 2: Read selesai → process command.

**Perbandingan blocking vs non-blocking read :**

```text
┌─────────────────────────────────────────────────────────────────┐
│              Blocking Read (SEBELUM)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Main Loop:                                                    │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  line, err := reader.ReadString('\n')  ← BLOCKING! ⏳    │  │
│   │  // Stuck di sini sampai timeout 30 detik!               │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ❌ Shutdown lambat (30 detik)                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              Non-Blocking Read (SESUDAH)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Main Loop:                                                    │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │  go func() { line, err := reader.ReadString('\n') }      │  │
│   │  select {                                                │  │
│   │  case <-shutdown:  ← Shutdown signal! ✅                 │  │
│   │      return  // Langsung keluar                          │  │
│   │  case res := <-readCh:                                   │  │
│   │      process(res)                                        │  │
│   │  }                                                       │  │
│   └──────────────────────────────────────────────────────────┘  │ 
│                                                                 │
│   ✅ Shutdown cepat (< 1 detik)                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Step 4: Implementasi Method Shutdown

```go
// internal/server/server.go

// Shutdown menghentikan server dengan graceful (default timeout 30 detik)
func (s *Server) Shutdown() error {
    return s.ShutdownWithTimeout(30 * time.Second)
}

func (s *Server) ShutdownWithTimeout(timeout time.Duration) error {
    s.logger.Println("Starting graceful shutdown...")

    // 1. Signal stop menerima koneksi baru
    close(s.quit)

    // 2. Tutup listener (stop accept)
    s.mu.Lock()
    if !s.closed {
        s.closed = true
        if s.listener != nil {
            s.listener.Close()
        }
    }
    s.mu.Unlock()

    // 3. Tunggu semua koneksi selesai
    done := make(chan struct{})
    go func() {
        s.wg.Wait()  // Tunggu semua goroutine selesai
        close(done)
    }()

    // 4. Tunggu dengan timeout
    select {
    case <-done:
        s.logger.Println("All connections finished gracefully")
        return nil
    case <-time.After(timeout):
        active := atomic.LoadInt32(&s.activeConn)
        s.logger.Printf("Shutdown timeout, %d connections still active", active)
        return fmt.Errorf("shutdown timeout, %d connections still active", active)
    }
}
```

Penjelasan:
- close(s.quit): Signal ke semua goroutine
- listener.Close(): Hentikan accept connection
- wg.Wait(): Tunggu semua koneksi selesai
- time.After(): Jangan tunggu selamanya (timeout)

#### Step 5: Handling Shutdown di Accept Loop

```go
func (s *Server) Start() error {
    // ... setup listener

    for {
        // Cek shutdown signal
        select {
        case <-s.quit:
            s.logger.Println("Server stopped accepting new connections")
            return nil
        default:
        }

        conn, err := s.listener.Accept()
        if err != nil {
            // Cek apakah listener sengaja ditutup
            s.mu.Lock()
            closed := s.closed
            s.mu.Unlock()

            if closed {
                s.logger.Println("Listener closed, stopping accept loop")
                return nil
            }

            // ... handle error lain
            continue
        }

        // Cek shutdown signal sebelum menerima koneksi
        select {
        case <-s.quit:
            conn.Write([]byte("ERR server shutting down\n"))
            conn.Close()
            continue
        default:
        }

        // ... process connection
    }
}
```

Penjelasan:
- Cek shutdown signal di awal loop
- Saat shutdown, stop menerima koneksi baru
- Koneksi yang masuk saat shutdown ditolak dengan pesan

#### Handling Graceful Shutdown di Fungsi `main()`

```go
func main() {
	config := server.ServerConfig{
		MaxConnections: 50_000,
		ReadTimeout:    30 * time.Second,
		WriteTimeout:   10 * time.Second,
		IdleTimeout:    60 * time.Second,
	}
	srv := server.NewServerWithConfig(":6379", config)

	srv.RegisterHandler("PING", func(args []string) string {
		if len(args) > 0 {
			return args[0]
		}
		return "PONG"
	})

	srv.RegisterHandler("MEMORY", func(args []string) string {
		if len(args) > 0 {
			return args[0]
		}

		return srv.PrintMemoryUsage()
	})

	fmt.Println("╔═══════════════════════════════════════════════════════╗")
	fmt.Println("║                     P E N D E M                       ║")
	fmt.Println("║              Simple Cache Server in Go                ║")
	fmt.Println("╠═══════════════════════════════════════════════════════╣")
	fmt.Printf("║  Address			: %-22s║\n", "0.0.0.0:6379")
	fmt.Println("╚═══════════════════════════════════════════════════════╝")
	fmt.Println()

	go func() {
		if err := srv.Start(); err != nil {
			log.Fatalf("Server error: %v", err)
		}
	}()

	// Graceful shutdown
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	sig := <-quit
	fmt.Printf("\n\n╔══════════════════════════════════════════════════════╗\n")
	fmt.Printf("║  Received signal: %-34s ║\n", sig)
	fmt.Printf("║  Shutting down gracefully...                         ║\n")
	fmt.Printf("╚══════════════════════════════════════════════════════╝\n")

	if err := srv.Shutdown(); err != nil {
		log.Printf("Shutdown error: %v", err)
		os.Exit(1)
	}

	fmt.Println("\n✅ Server stopped gracefully")
}
```

#### Flow Diagram

```text
┌───────────────────────────────────────────────────────────┐
│                      SERVER RUNNING                       │
│                                                           │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                    │
│  │ Conn 1  │  │ Conn 2  │  │ Conn 3  │  ← Active koneksi  │
│  └─────────┘  └─────────┘  └─────────┘                    │
│                                                           │
│  Accept loop → Waiting for new connections                │
└───────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────┐
│                   SHUTDOWN SIGNAL (Ctrl+C)                │
│                                                           │
│ 1. close(s.quit)  → Signal ke semua goroutine             │
│ 2. listener.Close() → Stop accept                         │
└───────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────┐
│                  WAITING FOR CONNECTIONS                  │
│                                                           │
│  ┌─────────┐  ┌─────────┐                                 │
│  │ Conn 1  │  │ Conn 2  │  ← Menunggu selesai             │
│  └─────────┘  └─────────┘                                 │
│                                                           │
│  s.wg.Wait() → Menunggu semua koneksi selesai             │
└───────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────┐
│                   SHUTDOWN COMPLETE                       │
│                                                           │
│ ✅ All connections closed                                 │
│ ✅ Resources cleaned up                                   │
│ ✅ Server stopped gracefully                              │
└───────────────────────────────────────────────────────────┘
```

### 3.8.3 Testing Graceful Shutdown

#### Test 1: Shutdown Tanpa Koneksi

```bash
go run cmd/main.go
╔═══════════════════════════════════════════════════════╗
║                     P E N D E M                       ║
║              Simple Cache Server in Go                ║
╠═══════════════════════════════════════════════════════╣
║  Address                      : 0.0.0.0:6379          ║
╚═══════════════════════════════════════════════════════╝

[SERVER] 2026/07/18 10:32:17 Server listening on :6379
[SERVER] 2026/07/18 10:32:17 Max connections: 50000
[SERVER] 2026/07/18 10:32:17 Read timeout: 30s
[SERVER] 2026/07/18 10:32:17 Write timeout: 10s
[SERVER] 2026/07/18 10:32:17 Idle timeout: 1m0s
^C

╔══════════════════════════════════════════════════════╗
║  Received signal: interrupt                          ║
║  Shutting down gracefully...                         ║
╚══════════════════════════════════════════════════════╝
[SERVER] 2026/07/18 10:32:19 Starting graceful shutdown...
[SERVER] 2026/07/18 10:32:19 Listener closed, stopping accept loop
[SERVER] 2026/07/18 10:32:19 All connections finished gracefully

✅ Server stopped gracefully
```

#### Test 2: Shutdown dengan Koneksi Aktif

```bash
go run cmd/main.go
╔═══════════════════════════════════════════════════════╗
║                     P E N D E M                       ║
║              Simple Cache Server in Go                ║
╠═══════════════════════════════════════════════════════╣
║  Address                      : 0.0.0.0:6379          ║
╚═══════════════════════════════════════════════════════╝

[SERVER] 2026/07/18 11:22:50 Server listening on :6379
[SERVER] 2026/07/18 11:22:50 Max connections: 50000
[SERVER] 2026/07/18 11:22:50 Read timeout: 30s
[SERVER] 2026/07/18 11:22:50 Write timeout: 10s
[SERVER] 2026/07/18 11:22:50 Idle timeout: 1m0s
[SERVER] 2026/07/18 11:22:54 New connection from [::1]:61376
[SERVER] 2026/07/18 11:22:54 Active connections: 1
[SERVER] 2026/07/18 11:22:57 Command from [::1]:61376: ping
[SERVER] 2026/07/18 11:23:01 New connection from [::1]:61377
[SERVER] 2026/07/18 11:23:01 Active connections: 2
[SERVER] 2026/07/18 11:23:04 Command from [::1]:61377: ping
^C

╔══════════════════════════════════════════════════════╗
║  Received signal: interrupt                          ║
║  Shutting down gracefully...                         ║
╚══════════════════════════════════════════════════════╝
[SERVER] 2026/07/18 11:23:09 Starting graceful shutdown...
[SERVER] 2026/07/18 11:23:09 Connection closed: [::1]:61377
[SERVER] 2026/07/18 11:23:09 Listener closed, stopping accept loop
[SERVER] 2026/07/18 11:23:09 Read error from [::1]:61376: read tcp [::1]:6379->[::1]:61376: use of closed network connection
[SERVER] 2026/07/18 11:23:09 Connection closed: [::1]:61376
[SERVER] 2026/07/18 11:23:09 Active connections: 1
[SERVER] 2026/07/18 11:23:09 Read error from [::1]:61377: read tcp [::1]:6379->[::1]:61377: use of closed network connection
[SERVER] 2026/07/18 11:23:09 Active connections: 0
[SERVER] 2026/07/18 11:23:09 All connections finished gracefully

✅ Server stopped gracefully
```

#### Tampilan di Client 

```bash
telnet localhost 6379
Trying ::1...
Connected to localhost.
Escape character is '^]'.
ping
PONG
Connection closed by foreign host.
```

## 3.9 Best Practice Summary

1. **Always Close Connections** - Gunakan `defer conn.Close()`
2. **Set Timeouts** - Read, Write, dan Idle timeout
3. **Limit Connections** - Lindungi resource server
4. **Implement Graceful Shutdown** - Jangan `os.Exit()` langsung
5. **Use Atomic Counters** - Untuk tracking koneksi aktif
6. **Monitor Memory** - Pantau resource usage
7. **Handle Errors Properly** - Bedakan timeout vs error lain
8. **Use Small Buffers** - Untuk banyak koneksi, gunakan 1KB buffer

## 3.10 Kode Lengkap

Kode lengkap untuk bab ini tersedia di:
- [server/server.go](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/03-socket-programming/internal/server/server.go)
- [server/connection.go](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/03-socket-programming/internal/server/connection.go)
- [cmd/main.go](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/03-socket-programming/cmd/main.go)


Saat ini kita masih memproses perintah yang masuk secara manual menggunakan fungsi `processCommand(line string) string`. Bab selanjutnya kita akan mempelajari standard pemrosesan perintah RESP Protocol secara mendalam dengan implementasi parser dan encoder yang lengkap. 
