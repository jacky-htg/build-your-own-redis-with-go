# Bab 4: RESP Protocol - Bahasa Komunikasi Redis

## 4.1 Pendahuluan: Memahami RESP Protocol

RESP (REdis Serialization Protocol) adalah protokol yang digunakan Redis untuk komunikasi antara client dan server. RESP dirancang dengan filosofi sederhana: mudah diimplementasikan, cepat diparse, dan human-readable.

>    📂 Kode Lengkap Bab Ini:
>    Seluruh kode yang dibahas di bab ini tersedia di GitHub:
>
>    🔗 [github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/04-resp-protocol](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/04-resp-protocol)


### 4.1.1 Mengapa RESP?

Redis memilih RESP karena beberapa alasan:
| Karakteristik | Keuntungan |
|---------------|------------|
| Human-readable | Mudah untuk debugging dengan telnet |
| Simple to parse | Implementasi parser sangat straightforward |
| Fast | Parsing minimal, overhead rendah |
| Supports multiple types | String, Integer, Array, Error |

### 4.1.2 Contoh RESP di Redis

```bash
# Client mengirim perintah PING
redis-cli PING

# RESP di balik layar:
# Client → Server: *1\r\n$4\r\nPING\r\n
# Server → Client: +PONG\r\n
```

### 4.1.3 RESP vs Protokol Lain

| Protokol | Human-Readable | Performance | Complexity |
|----------|----------------|-------------|------------|
| RESP | ✅ | High | Low |
| JSON | ✅ | Medium | Medium |
| Protobuf | ❌ | Very High | High |
| MessagePack | ❌ | High | Medium |

## 4.2 RESP Data Types

RESP mendukung 5 tipe data utama, masing-masing dengan prefix karakter tertentu:

### 4.2.1 Simple String (+)

Simple string adalah respons sederhana yang diawali dengan + dan diakhiri dengan \r\n.

```text
+OK\r\n
+PONG\r\n
+hello world\r\n
```

Contoh Penggunaan:
- `PING` → `+PONG\r\n`
- `SET key value` → `+OK\r\n`

### 4.2.2 Error (-)

Error message diawali dengan - dan diakhiri \r\n.

```text
-ERR unknown command\r\n
-ERR wrong number of arguments\r\n
```

Contoh Penggunaan:
- Command tidak dikenal → `-ERR unknown command\r\n`
- Argumen salah → `-ERR wrong number of arguments\r\n`

### 4.2.3 Integer (:)

Integer diawali dengan : dan diikuti angka.

```text
:1000\r\n
:-1\r\n
:0\r\n
```

Contoh Penggunaan:
- INCR counter → `:1\r\n`
- LLEN list → `:5\r\n`

### 4.2.4 Bulk String ($)

Bulk string adalah string dengan panjang tertentu. Format: `$<length>\r\n<data>\r\n`

```text
$5\r\nhello\r\n     → "hello"
$0\r\n\r\n          → "" (empty string)
$-1\r\n             → null (nil)
```

Struktur Bulk String:

```text
$<length>\r\n
<data>
\r\n
```

Contoh Penggunaan:
- `GET key` → `$5\r\nvalue\r\n`
- `PING hello` → `$5\r\nhello\r\n`

### 4.2.5 Array (*)

Array adalah kumpulan nilai RESP. Format: `*<count>\r\n<elements>`

```text
*2\r\n$4\r\nPING\r\n$4\r\nPONG\r\n
*0\r\n          → empty array
*-1\r\n         → null array
```

Struktur Array:

```text
*<count>\r\n
<element1>
<element2>
...
<elementN>
```

Contoh Penggunaan:
- `PING` → `*1\r\n$4\r\nPING\r\n`
- `SET key value` → `*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n`

### 4.2.6 Null Values

RESP memiliki dua jenis null:

| Tipe | Representasi | Keterangan |
|------|--------------|------------|
| Null Bulk String | `$-1\r\n` | Nilai null (tidak ada data) |
| Null Array | `*-1\r\n` | Array null |

Perbedaan:

```text
$0\r\n\r\n   → Empty string ("")
$-1\r\n      → Null (nil)

*0\r\n       → Empty array ([])
*-1\r\n      → Null array (nil)
```

### 4.2.7 Summary Table

| Tipe | Prefix | Format | Contoh |
|------|--------|--------|--------|
| Simple String | + | `+<string>\r\n` | `+OK\r\n` |
| Error | - | `-<message>\r\n` | `-ERR unknown\r\n` |
| Integer | : | `:<number>\r\n` | `:1000\r\n` |
| Bulk String | $ | `$<len>\r\n<data>\r\n` | `$4\r\nPING\r\n` |
| Array | * | `*<count>\r\n<elems>` | `*2\r\n$4\r\nPING\r\n...` |
| Null Bulk | $ | `$-1\r\n` | `$-1\r\n` |
| Null Array | * | `*-1\r\n` | `*-1\r\n` |

## 4.3 Implementasi Parser RESP

### 4.3.1 Struct dan Constants

Pertama, kita definisikan tipe data dan konstanta yang dibutuhkan:

```go
// internal/server/resp.go
package server

import (
	"bufio"
	"errors"
	"fmt"
	"io"
	"strconv"
	"strings"
)

// Security limits
const MaxRESPDepth = 32
const MaxBulkStringSize = 512 * 1024 * 1024 // 512MB

// Custom errors
var (
	ErrInvalidRESP  = errors.New("invalid RESP format")
	ErrRESPTooDeep  = errors.New("RESP nesting too deep")
	ErrRESPTooLarge = errors.New("RESP value too large")
)

// RESPType represents the type of RESP value
type RESPType int

const (
	SimpleString RESPType = iota
	Error
	Integer
	BulkString
	Array
)

// RESPValue represents a parsed RESP value
type RESPValue struct {
	Type   RESPType
	Str    string
	Int    int64
	Array  []RESPValue
	IsNull bool
}

// RESPParser parses RESP protocol from a reader
type RESPParser struct {
	reader *bufio.Reader
	depth  int // Track nesting depth for security
}

func NewRESPParser(r io.Reader) *RESPParser {
	return &RESPParser{
		reader: bufio.NewReader(r),
	}
}
```

Penjelasan:
- MaxRESPDepth: Mencegah serangan nested array yang dalam (stack overflow)
- MaxBulkStringSize: Mencegah alokasi memory berlebihan
- RESPValue: Struct untuk menyimpan hasil parsing
- IsNull: Flag untuk membedakan null vs empty string

### 4.3.2 Parsing Simple String & Error

Simple String dan Error memiliki format yang sama: `+<data>\r\n` dan `-<data>\r\n`.

```go
// internal/server/resp.go

// Helper untuk membaca line dengan CRLF validation
func (p *RESPParser) readLine() (string, error) {
	line, err := p.reader.ReadString('\n')
	if err != nil {
		return "", fmt.Errorf("%w: %v", ErrInvalidRESP, err)
	}
	
	// Validate CRLF
	if len(line) < 2 || line[len(line)-2] != '\r' {
		return "", fmt.Errorf("%w: missing CRLF", ErrInvalidRESP)
	}
	
	return line[:len(line)-2], nil
}

func (p *RESPParser) readSimpleString() (*RESPValue, error) {
	line, err := p.readLine()
	if err != nil {
		return nil, err
	}
	
	return &RESPValue{
		Type: SimpleString,
		Str:  line,
	}, nil
}

func (p *RESPParser) readError() (*RESPValue, error) {
	line, err := p.readLine()
	if err != nil {
		return nil, err
	}
	
	return &RESPValue{
		Type: Error,
		Str:  line,
	}, nil
}
```

Penjelasan:
- readLine() adalah helper untuk membaca line dengan validasi CRLF
- Simple String dan Error hanya berbeda prefix (+ vs -)

### 4.3.3 Parsing Integer

Integer memiliki format `:<number>\r\n`.

```go
// internal/server/resp.go

func (p *RESPParser) readInteger() (*RESPValue, error) {
	line, err := p.readLine()
	if err != nil {
		return nil, err
	}
	
	val, err := strconv.ParseInt(line, 10, 64)
	if err != nil {
		return nil, fmt.Errorf("%w: invalid integer", ErrInvalidRESP)
	}
	
	return &RESPValue{
		Type: Integer,
		Int:  val,
	}, nil
}
```

### 4.3.4 Parsing Bulk String

Bulk string adalah yang paling kompleks karena harus membaca data dengan panjang tertentu.

```go
// internal/server/resp.go

func (p *RESPParser) readBulkString() (*RESPValue, error) {
	line, err := p.readLine()
	if err != nil {
		return nil, err
	}

	length, err := strconv.Atoi(line)
	if err != nil {
		return nil, fmt.Errorf("%w: invalid bulk string length", ErrInvalidRESP)
	}

	// Null bulk string
	if length == -1 {
		return &RESPValue{
			Type:   BulkString,
			Str:    "",
			IsNull: true,
		}, nil
	}

	// Empty string
	if length == 0 {
		return &RESPValue{
			Type:   BulkString,
			Str:    "",
			IsNull: false,
		}, nil
	}

	// Security: check size limit
	if length > MaxBulkStringSize {
		return nil, fmt.Errorf("%w: bulk string too large (%d bytes)", 
			ErrRESPTooLarge, length)
	}

	// Read data + CRLF
	data := make([]byte, length+2)
	_, err = io.ReadFull(p.reader, data)
	if err != nil {
		return nil, fmt.Errorf("%w: failed to read bulk string data", ErrInvalidRESP)
	}

	return &RESPValue{
		Type:   BulkString,
		Str:    string(data[:length]),
		IsNull: false,
	}, nil
}
```

Penjelasan:
- Baca panjang string
- Handle null (-1) dan empty (0)
- Validasi size limit untuk keamanan
- Baca data sesuai panjang + \r\n

### 4.3.5 Parsing Array

Array adalah kumpulan RESP values yang diparse secara rekursif.

```go
// internal/server/resp.go

func (p *RESPParser) readArray() (*RESPValue, error) {
	// Track depth untuk keamanan
	p.depth++
	if p.depth > MaxRESPDepth {
		return nil, fmt.Errorf("%w: max depth %d exceeded", 
			ErrRESPTooDeep, MaxRESPDepth)
	}
	defer func() { p.depth-- }()

	line, err := p.readLine()
	if err != nil {
		return nil, err
	}

	length, err := strconv.Atoi(line)
	if err != nil {
		return nil, fmt.Errorf("%w: invalid array length", ErrInvalidRESP)
	}

	// Null array
	if length == -1 {
		return &RESPValue{
			Type:   Array,
			Array:  nil,
			IsNull: true,
		}, nil
	}

	// Parse each element
	array := make([]RESPValue, length)
	for i := 0; i < length; i++ {
		val, err := p.Read()
		if err != nil {
			return nil, fmt.Errorf("%w: failed to read array element %d", 
				ErrInvalidRESP, i)
		}
		array[i] = *val
	}

	return &RESPValue{
		Type:   Array,
		Array:  array,
		IsNull: false,
	}, nil
}
```

Penjelasan:
- Track depth untuk mencegah nested array berlebihan
- Baca panjang array
- Handle null array (-1)
- Parse setiap elemen secara rekursif

### 4.3.6 Main Read Method

Method utama yang menentukan tipe data berdasarkan prefix.

```go
// internal/server/resp.go

func (p *RESPParser) Read() (*RESPValue, error) {
	b, err := p.reader.ReadByte()
	if err != nil {
		return nil, err
	}

	switch b {
	case '+':
		return p.readSimpleString()
	case '-':
		return p.readError()
	case ':':
		return p.readInteger()
	case '$':
		return p.readBulkString()
	case '*':
		return p.readArray()
	default:
		return nil, fmt.Errorf("%w: unknown RESP type '%c'", ErrInvalidRESP, b)
	}
}
```

### 4.3.7 Security Considerations

Dalam implementasi parser, kita menambahkan beberapa security measure:

| Security Feature | Purpose | Implementation |
|------------------|---------|----------------|
| Depth Limit | Mencegah stack overflow dari nested array | MaxRESPDepth = 32 |
| Size Limit | Mencegah memory exhaustion | MaxBulkStringSize = 512MB |
| CRLF Validation | Memastikan format valid | Validasi di readLine() |

Kenapa ini penting?

```text
// Serangan Depth:
*1\r\n*1\r\n*1\r\n... (1000x nested)
→ Tanpa limit: stack overflow 💥
→ Dengan limit: error "max depth exceeded" ✅

// Serangan Size:
$9999999999\r\n
→ Tanpa limit: memory exhaustion 💥
→ Dengan limit: error "bulk string too large" ✅
```

## 4.4 Implementasi Encoder RESP

Encoder mengubah RESPValue menjadi string RESP.

### 4.4.1 Encoding dengan strings.Builder

```go
// internal/server/resp.go

// RESP Encoder dengan strings.Builder
func EncodeRESP(val RESPValue) string {
	var builder strings.Builder
	builder.Grow(64) // Pre-allocate untuk performa
	encodeRESPToBuilder(&builder, val)
	return builder.String()
}

func encodeRESPToBuilder(b *strings.Builder, val RESPValue) {
	switch val.Type {
	case SimpleString:
		b.WriteString("+")
		b.WriteString(val.Str)
		b.WriteString("\r\n")

	case Error:
		b.WriteString("-")
		b.WriteString(val.Str)
		b.WriteString("\r\n")

	case Integer:
		b.WriteString(":")
		b.WriteString(strconv.FormatInt(val.Int, 10))
		b.WriteString("\r\n")

	case BulkString:
		if val.IsNull {
			b.WriteString("$-1\r\n")
			return
		}
		b.WriteString("$")
		b.WriteString(strconv.Itoa(len(val.Str)))
		b.WriteString("\r\n")
		b.WriteString(val.Str)
		b.WriteString("\r\n")

	case Array:
		if val.IsNull {
			b.WriteString("*-1\r\n")
			return
		}
		b.WriteString("*")
		b.WriteString(strconv.Itoa(len(val.Array)))
		b.WriteString("\r\n")
		for _, v := range val.Array {
			encodeRESPToBuilder(b, v)
		}
	}
}
```

Kenapa strings.Builder?

```go
// ❌ Buruk: String concatenation (banyak alokasi)
result := "*" + strconv.Itoa(len(val.Array)) + "\r\n"
for _, v := range val.Array {
    result += EncodeRESP(v) // ← Alokasi baru setiap loop
}

// ✅ Baik: strings.Builder (efisien)
var builder strings.Builder
builder.WriteString("*")
builder.WriteString(strconv.Itoa(len(val.Array)))
builder.WriteString("\r\n")
// ... hanya satu alokasi final
```

### 4.4.2 Encoding Null Values

```go
// Null Bulk String: $-1\r\n
case BulkString:
    if val.IsNull {
        b.WriteString("$-1\r\n")
        return
    }
    // ... encode normal

// Null Array: *-1\r\n
case Array:
    if val.IsNull {
        b.WriteString("*-1\r\n")
        return
    }
    // ... encode normal
```

### 4.4.3 String Representation untuk Debugging

```go
// internal/server/resp.go

// String() method untuk debugging
func (v *RESPValue) String() string {
	switch v.Type {
	case SimpleString:
		return fmt.Sprintf("SimpleString(%q)", v.Str)
	case Error:
		return fmt.Sprintf("Error(%q)", v.Str)
	case Integer:
		return fmt.Sprintf("Integer(%d)", v.Int)
	case BulkString:
		if v.IsNull {
			return "BulkString(null)"
		}
		return fmt.Sprintf("BulkString(%q)", v.Str)
	case Array:
		if v.IsNull {
			return "Array(null)"
		}
		return fmt.Sprintf("Array(%d elements)", len(v.Array))
	default:
		return "Unknown"
	}
}
```

## 4.5 Integrasi dengan Server

### 4.5.1 Mengganti processCommand

Sekarang kita integrasikan RESP parser dengan server:

```go
// internal/server/server.go

// server/server.go

// CommandHandler sekarang mengembalikan RESPValue
type CommandHandler func(args []string) RESPValue

// processCommand versi RESP
func (s *Server) processCommand(value *RESPValue) RESPValue {
	// Harus array (perintah)
	if value.Type != Array || len(value.Array) < 1 {
		return RESPValue{
			Type: Error,
			Str:  "ERR invalid command",
		}
	}

	// Perintah pertama adalah command
	cmd := strings.ToUpper(value.Array[0].Str)
	args := make([]string, 0)

	// Kumpulkan arguments (bulk strings)
	for i := 1; i < len(value.Array); i++ {
		if value.Array[i].Type == BulkString {
			args = append(args, value.Array[i].Str)
		}
	}

	// Cari handler
	handler, exists := s.handlers[cmd]
	if !exists {
		return RESPValue{
			Type: Error,
			Str:  fmt.Sprintf("ERR unknown command '%s'", cmd),
		}
	}

	// Eksekusi handler
	return handler(args)
}
```

### 4.5.2 Update handleConnection

```go
// internal/server/server.go

func (s *Server) handleConnection(conn net.Conn) {
	// ... setup connection

	// Buat parser RESP
	parser := NewRESPParser(conn)

	for {
		// ... cek shutdown signal

		// Baca RESP
		respVal, err := parser.Read()
		if err != nil {
			// Handle error...
			return
		}

		// Proses command
		response := s.processCommand(respVal)

		// Encode dan kirim response
		conn.Write([]byte(EncodeRESP(response)))
		c.UpdateActivity()
	}
}
```

### 4.5.3 Update Command Handlers

```go
// cmd/main.go

func handlePing(args []string) server.RESPValue {
	if len(args) > 0 {
		return server.RESPValue{
			Type: server.BulkString,
			Str:  args[0],
		}
	}
	return server.RESPValue{
		Type: server.SimpleString,
		Str:  "PONG",
	}
}

func handleMemory(srv *server.Server, args []string) server.RESPValue {
	if len(args) > 0 {
		return server.RESPValue{
			Type: server.BulkString,
			Str:  args[0],
		}
	}

	return server.RESPValue{
		Type: server.SimpleString,
		Str:  srv.PrintMemoryUsage(),
	}
}
```

## 4.6 Testing dengan redis-cli
### 4.6.1 Persiapan

```bash
# Start server
go run cmd/main.go

# Testing dengan redis-cli
redis-cli -h localhost -p 6379
```

### 4.6.2 Test Cases

```bash
# 1. Test PING sederhana
127.0.0.1:6379> PING
PONG

# 2. Test PING dengan argumen
127.0.0.1:6379> PING hello
"hello"

# 3. Test MEMORY
127.0.0.1:6379> MEMORY
{"alloc": "198.09 KB", "total_alloc": "198.09 KB", "sys": "12.27 MB", "num_gc": "0"}

# 4. Test command tidak dikenal
127.0.0.1:6379> INVALID
(error) ERR unknown command 'INVALID'

# 5. Test command kosong
127.0.0.1:6379> 
(error) ERR invalid command
```

### 4.6.3 Debugging dengan redis-cli Monitor Mode

```bash
# Monitor mode - lihat raw RESP
redis-cli -h localhost -p 6379 --raw

# Atau dengan telnet (masih bisa untuk debugging)
telnet localhost 6379
*1\r\n$4\r\nPING\r\n    # Kirim RESP manual
+PONG\r\n                # Response
```

## 4.7 Best Practices

### 4.7.1 Security Best Practices

```go
// 1. Always validate input size
if length > MaxBulkStringSize {
    return nil, ErrRESPTooLarge
}

// 2. Limit nesting depth
if p.depth > MaxRESPDepth {
    return nil, ErrRESPTooDeep
}

// 3. Validate CRLF
if len(line) < 2 || line[len(line)-2] != '\r' {
    return nil, ErrInvalidRESP
}
```

### 4.7.2 Performance Best Practices

```go
// 1. Use strings.Builder for encoding
var builder strings.Builder
builder.Grow(64) // Pre-allocate

// 2. Use bufio.Reader for parsing
reader := bufio.NewReader(r)

// 3. Reuse buffers when possible
// (Consider using sync.Pool for high-load scenarios)
```

### 4.7.3 Code Organization

```text
server/
├── resp.go          # RESP parser & encoder
├── server.go        # Server logic
└── connection.go    # Connection management
```

### 4.7.4 Error Handling

```go
// 1. Use specific error types
var (
    ErrInvalidRESP  = errors.New("invalid RESP format")
    ErrRESPTooDeep  = errors.New("RESP nesting too deep")
    ErrRESPTooLarge = errors.New("RESP value too large")
)

// 2. Wrap errors with context
return nil, fmt.Errorf("%w: failed to read array element %d", 
    ErrInvalidRESP, i)

// 3. Handle EOF vs other errors
if err == io.EOF {
    // Client disconnected gracefully
} else if ne, ok := err.(net.Error); ok && ne.Timeout() {
    // Timeout
} else {
    // Other error
}
```

## 4.8 Ringkasan

Yang Sudah Kita Pelajari:

1. RESP Protocol
- 5 tipe data: Simple String, Error, Integer, Bulk String, Array
- Null values: $-1 dan *-1
- Format: $<length>\r\n<data>\r\n

2. Parser Implementation
- Parse semua tipe RESP
- Security: depth & size limits
- CRLF validation

3. Encoder Implementation
- Menggunakan strings.Builder untuk efisiensi
- Handle null values dengan IsNull flag

4. Integration
- Mengganti processCommand dengan RESP
- Update handlers untuk return RESPValue

5. Testing
- Menggunakan redis-cli untuk testing
- Debugging dengan telnet

### Perbandingan Sebelum vs Sesudah

| Aspek | Sebelum (Bab 3) | Sesudah (Bab 4) |
|-------|-----------------|-----------------|
| Protocol | Custom (plain text) | RESP (standard) |
| Parser | Manual string parsing | RESP parser |
| Testing | Telnet/nc | redis-cli |
| Compatibility | Custom client only | Any Redis client |
| Error Handling | Simple string error | RESP Error type |

## Next Steps

Dengan RESP protocol, server Pendem sekarang:
- ✅ Bisa diakses dengan redis-cli
- ✅ Mendukung semua client Redis
- ✅ Menggunakan protokol standar

Bab selanjutnya: Kita akan menambahkan kemampuan cache storage dengan SET, GET, dan DEL commands! 🚀