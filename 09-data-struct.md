# Bab 9: Data Structures - Beyond Simple Key-Value

## 9.1 Pendahuluan

Setelah kita berhasil membangun persistence dan config management, sekarang saatnya memperkaya Pendem dengan berbagai tipe data seperti di Redis.

>    📂 Kode Lengkap Bab Ini:
>    Seluruh kode yang dibahas di bab ini tersedia di GitHub:
>
>    🔗 [github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/09-data-struct](https://github.com/jacky-htg/build-your-own-redis-with-go-code/tree/main/09-data-struct)


### 9.1.1 Redis Data Types

Redis mendukung berbagai tipe data yang membuatnya lebih dari sekadar key-value store:

```text
┌─────────────────────────────────────────────────────────────────┐
│              REDIS DATA TYPES                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    STRING                               │   │
│   │  SET key "value"   GET key   APPEND key "text"          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    HASH                                 │   │
│   │  HSET user:1 name "John" age 30                         │   │
│   │  HGET user:1 name   HGETALL user:1                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    LIST                                 │   │
│   │  LPUSH queue "item1"   RPUSH queue "item2"              │   │
│   │  LPOP queue   LRANGE queue 0 -1                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    SET                                  │   │
│   │  SADD set "a" "b"   SREM set "a"   SMEMBERS set         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    SORTED SET                           │   │
│   │  ZADD leaderboard 100 "player1" 200 "player2"           │   │
│   │  ZRANGE leaderboard 0 -1 WITHSCORES                     │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## 9.2 Desain Arsitektur

### 9.2.1 Type System

```go
// internal/engine/value.go
package engine

type ValueType int

const (
    TypeString ValueType = iota
    TypeHash
    TypeList
    TypeSet
    TypeSortedSet
)

type Value struct {
    Type  ValueType
    Data  interface{}
    TTL   time.Duration
    Exp   int64
}
```

### 9.2.2 Struktur Folder

```text
internal/
├── engine/
│   ├── cache.go          # Main cache dengan type system
│   ├── value.go          # Value dengan multiple types
│   ├── hash.go           # Hash operations
│   ├── list.go           # List operations
│   ├── set.go            # Set operations
│   └── sortedset.go      # Sorted Set operations
├── handler/
│   └── data_types.go     # Command handlers untuk semua tipe
└── server/
    └── resp.go           # RESP protocol (existing)
```

## 9.3 String Operations (Existing)

Kita sudah memiliki operasi string dasar. Tambahkan beberapa yang hilang:

### 9.3.1 APPEND

```go
// internal/handler/string.go
func (h *Handler[V]) Append(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'append' command",
        }
    }

    key := args[0]
    value := args[1]
    
    // Get existing value
    oldVal, exists := h.cache.Get(key)
    newVal := value
    if exists {
        newVal = oldVal + value
    }
    
    h.cache.Set(key, newVal, 0)
    
    // Log ke AOF
    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("APPEND", args...)
    }
    
    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(len(newVal)),
    }
}
```

### 9.3.2 STRLEN

```go
func (h *Handler[V]) Strlen(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'strlen' command",
        }
    }

    key := args[0]
    val, exists := h.cache.Get(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(len(val)),
    }
}
```

## 9.4 Hash Operations

Hash adalah map dari field → value di dalam satu key.

### 9.4.1 Struktur Hash

```go
// internal/engine/hash.go
package engine

import "sync"

type Hash struct {
    fields map[string]string
    mu     sync.RWMutex
}

func NewHash() *Hash {
    return &Hash{
        fields: make(map[string]string),
    }
}

func (h *Hash) Set(field, value string) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.fields[field] = value
}

func (h *Hash) Get(field string) (string, bool) {
    h.mu.RLock()
    defer h.mu.RUnlock()
    val, exists := h.fields[field]
    return val, exists
}

func (h *Hash) Delete(field string) bool {
    h.mu.Lock()
    defer h.mu.Unlock()
    if _, exists := h.fields[field]; exists {
        delete(h.fields, field)
        return true
    }
    return false
}

func (h *Hash) GetAll() map[string]string {
    h.mu.RLock()
    defer h.mu.RUnlock()
    result := make(map[string]string, len(h.fields))
    for k, v := range h.fields {
        result[k] = v
    }
    return result
}

func (h *Hash) Len() int {
    h.mu.RLock()
    defer h.mu.RUnlock()
    return len(h.fields)
}

func (h *Hash) Keys() []string {
    h.mu.RLock()
    defer h.mu.RUnlock()
    keys := make([]string, 0, len(h.fields))
    for k := range h.fields {
        keys = append(keys, k)
    }
    return keys
}

func (h *Hash) Values() []string {
    h.mu.RLock()
    defer h.mu.RUnlock()
    values := make([]string, 0, len(h.fields))
    for _, v := range h.fields {
        values = append(values, v)
    }
    return values
}
```

### 9.4.2 Hash Commands

```go

// internal/handler/hash.go
package handler

import (
    "pendem/internal/engine"
    "pendem/internal/server"
    "strings"
)

// HSET key field value [field value ...]
func (h *Handler[V]) HSet(args []string) server.RESPValue {
    if len(args) < 3 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'hset' command",
        }
    }

    key := args[0]
    fields := args[1:]

    if len(fields)%2 != 0 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'hset' command",
        }
    }

    // Get or create hash
    hash, created := h.cache.GetOrCreateHash(key)
    
    count := 0
    for i := 0; i < len(fields); i += 2 {
        field := fields[i]
        value := fields[i+1]
        if !hash.Has(field) {
            count++
        }
        hash.Set(field, value)
    }

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("HSET", args...)
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(count),
    }
}

// HGET key field
func (h *Handler[V]) HGet(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'hget' command",
        }
    }

    key := args[0]
    field := args[1]

    hash, exists := h.cache.GetHash(key)
    if !exists {
        return server.RESPValue{
            Type: server.BulkString,
            Str:  "",
            IsNull: true,
        }
    }

    val, found := hash.Get(field)
    if !found {
        return server.RESPValue{
            Type: server.BulkString,
            Str:  "",
            IsNull: true,
        }
    }

    return server.RESPValue{
        Type: server.BulkString,
        Str:  val,
    }
}

// HGETALL key
func (h *Handler[V]) HGetAll(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'hgetall' command",
        }
    }

    key := args[0]
    hash, exists := h.cache.GetHash(key)
    if !exists {
        return server.RESPValue{
            Type: server.Array,
            Array: []server.RESPValue{},
        }
    }

    all := hash.GetAll()
    result := make([]server.RESPValue, 0, len(all)*2)
    for field, value := range all {
        result = append(result, server.RESPValue{
            Type: server.BulkString,
            Str:  field,
        })
        result = append(result, server.RESPValue{
            Type: server.BulkString,
            Str:  value,
        })
    }

    return server.RESPValue{
        Type:  server.Array,
        Array: result,
    }
}

// HDEL key field [field ...]
func (h *Handler[V]) HDel(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'hdel' command",
        }
    }

    key := args[0]
    fields := args[1:]

    hash, exists := h.cache.GetHash(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    count := 0
    for _, field := range fields {
        if hash.Delete(field) {
            count++
        }
    }

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("HDEL", args...)
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(count),
    }
}

// HLEN key
func (h *Handler[V]) HLen(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'hlen' command",
        }
    }

    key := args[0]
    hash, exists := h.cache.GetHash(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(hash.Len()),
    }
}

// HEXISTS key field
func (h *Handler[V]) HExists(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'hexists' command",
        }
    }

    key := args[0]
    field := args[1]

    hash, exists := h.cache.GetHash(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    _, found := hash.Get(field)
    if found {
        return server.RESPValue{
            Type: server.Integer,
            Int:  1,
        }
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  0,
    }
}
```

## 9.5 List Operations

List adalah collection of strings ordered by insertion.

### 9.5.1 Struktur List

```go
// internal/engine/list.go
package engine

import (
    "container/list"
    "sync"
)

type List struct {
    items *list.List
    mu    sync.RWMutex
}

func NewList() *List {
    return &List{
        items: list.New(),
    }
}

func (l *List) LPush(values ...string) int {
    l.mu.Lock()
    defer l.mu.Unlock()
    for _, v := range values {
        l.items.PushFront(v)
    }
    return l.items.Len()
}

func (l *List) RPush(values ...string) int {
    l.mu.Lock()
    defer l.mu.Unlock()
    for _, v := range values {
        l.items.PushBack(v)
    }
    return l.items.Len()
}

func (l *List) LPop() (string, bool) {
    l.mu.Lock()
    defer l.mu.Unlock()
    elem := l.items.Front()
    if elem == nil {
        return "", false
    }
    l.items.Remove(elem)
    return elem.Value.(string), true
}

func (l *List) RPop() (string, bool) {
    l.mu.Lock()
    defer l.mu.Unlock()
    elem := l.items.Back()
    if elem == nil {
        return "", false
    }
    l.items.Remove(elem)
    return elem.Value.(string), true
}

func (l *List) LRange(start, stop int) []string {
    l.mu.RLock()
    defer l.mu.RUnlock()
    
    length := l.items.Len()
    if length == 0 {
        return []string{}
    }
    
    // Handle negative indices
    if start < 0 {
        start = length + start
    }
    if stop < 0 {
        stop = length + stop
    }
    if start < 0 {
        start = 0
    }
    if stop >= length {
        stop = length - 1
    }
    if start > stop {
        return []string{}
    }
    
    result := make([]string, 0, stop-start+1)
    i := 0
    for elem := l.items.Front(); elem != nil; elem = elem.Next() {
        if i >= start && i <= stop {
            result = append(result, elem.Value.(string))
        }
        if i > stop {
            break
        }
        i++
    }
    return result
}

func (l *List) LLen() int {
    l.mu.RLock()
    defer l.mu.RUnlock()
    return l.items.Len()
}
```

### 9.5.2 List Commands

```go
// internal/handler/list.go
package handler

import (
    "pendem/internal/server"
    "strconv"
)

// LPUSH key value [value ...]
func (h *Handler[V]) LPush(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'lpush' command",
        }
    }

    key := args[0]
    values := args[1:]

    list, created := h.cache.GetOrCreateList(key)
    count := list.LPush(values...)

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("LPUSH", args...)
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(count),
    }
}

// RPUSH key value [value ...]
func (h *Handler[V]) RPush(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'rpush' command",
        }
    }

    key := args[0]
    values := args[1:]

    list, created := h.cache.GetOrCreateList(key)
    count := list.RPush(values...)

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("RPUSH", args...)
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(count),
    }
}

// LPOP key
func (h *Handler[V]) LPop(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'lpop' command",
        }
    }

    key := args[0]
    list, exists := h.cache.GetList(key)
    if !exists {
        return server.RESPValue{
            Type: server.BulkString,
            Str:  "",
            IsNull: true,
        }
    }

    val, found := list.LPop()
    if !found {
        return server.RESPValue{
            Type: server.BulkString,
            Str:  "",
            IsNull: true,
        }
    }

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("LPOP", args...)
    }

    return server.RESPValue{
        Type: server.BulkString,
        Str:  val,
    }
}

// RPOP key
func (h *Handler[V]) RPop(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'rpop' command",
        }
    }

    key := args[0]
    list, exists := h.cache.GetList(key)
    if !exists {
        return server.RESPValue{
            Type: server.BulkString,
            Str:  "",
            IsNull: true,
        }
    }

    val, found := list.RPop()
    if !found {
        return server.RESPValue{
            Type: server.BulkString,
            Str:  "",
            IsNull: true,
        }
    }

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("RPOP", args...)
    }

    return server.RESPValue{
        Type: server.BulkString,
        Str:  val,
    }
}

// LRANGE key start stop
func (h *Handler[V]) LRange(args []string) server.RESPValue {
    if len(args) < 3 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'lrange' command",
        }
    }

    key := args[0]
    start, err1 := strconv.Atoi(args[1])
    stop, err2 := strconv.Atoi(args[2])
    
    if err1 != nil || err2 != nil {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR value is not an integer or out of range",
        }
    }

    list, exists := h.cache.GetList(key)
    if !exists {
        return server.RESPValue{
            Type: server.Array,
            Array: []server.RESPValue{},
        }
    }

    items := list.LRange(start, stop)
    result := make([]server.RESPValue, len(items))
    for i, item := range items {
        result[i] = server.RESPValue{
            Type: server.BulkString,
            Str:  item,
        }
    }

    return server.RESPValue{
        Type:  server.Array,
        Array: result,
    }
}

// LLEN key
func (h *Handler[V]) LLen(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'llen' command",
        }
    }

    key := args[0]
    list, exists := h.cache.GetList(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(list.LLen()),
    }
}
```

## 9.6 Set Operations

Set adalah collection of unique strings.

### 9.6.1 Struktur Set

```go
// internal/engine/set.go
package engine

import "sync"

type Set struct {
    members map[string]struct{}
    mu      sync.RWMutex
}

func NewSet() *Set {
    return &Set{
        members: make(map[string]struct{}),
    }
}

func (s *Set) Add(members ...string) int {
    s.mu.Lock()
    defer s.mu.Unlock()
    count := 0
    for _, m := range members {
        if _, exists := s.members[m]; !exists {
            s.members[m] = struct{}{}
            count++
        }
    }
    return count
}

func (s *Set) Remove(members ...string) int {
    s.mu.Lock()
    defer s.mu.Unlock()
    count := 0
    for _, m := range members {
        if _, exists := s.members[m]; exists {
            delete(s.members, m)
            count++
        }
    }
    return count
}

func (s *Set) IsMember(member string) bool {
    s.mu.RLock()
    defer s.mu.RUnlock()
    _, exists := s.members[member]
    return exists
}

func (s *Set) Members() []string {
    s.mu.RLock()
    defer s.mu.RUnlock()
    result := make([]string, 0, len(s.members))
    for m := range s.members {
        result = append(result, m)
    }
    return result
}

func (s *Set) Len() int {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return len(s.members)
}
```

### 9.6.2 Set Commands

```go
// internal/handler/set.go
package handler

import (
    "pendem/internal/server"
)

// SADD key member [member ...]
func (h *Handler[V]) SAdd(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'sadd' command",
        }
    }

    key := args[0]
    members := args[1:]

    set, created := h.cache.GetOrCreateSet(key)
    count := set.Add(members...)

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("SADD", args...)
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(count),
    }
}

// SREM key member [member ...]
func (h *Handler[V]) SRem(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'srem' command",
        }
    }

    key := args[0]
    members := args[1:]

    set, exists := h.cache.GetSet(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    count := set.Remove(members...)

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("SREM", args...)
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(count),
    }
}

// SMEMBERS key
func (h *Handler[V]) SMembers(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'smembers' command",
        }
    }

    key := args[0]
    set, exists := h.cache.GetSet(key)
    if !exists {
        return server.RESPValue{
            Type: server.Array,
            Array: []server.RESPValue{},
        }
    }

    members := set.Members()
    result := make([]server.RESPValue, len(members))
    for i, m := range members {
        result[i] = server.RESPValue{
            Type: server.BulkString,
            Str:  m,
        }
    }

    return server.RESPValue{
        Type:  server.Array,
        Array: result,
    }
}

// SISMEMBER key member
func (h *Handler[V]) SIsMember(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'sismember' command",
        }
    }

    key := args[0]
    member := args[1]

    set, exists := h.cache.GetSet(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    if set.IsMember(member) {
        return server.RESPValue{
            Type: server.Integer,
            Int:  1,
        }
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  0,
    }
}

// SCARD key
func (h *Handler[V]) SCard(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'scard' command",
        }
    }

    key := args[0]
    set, exists := h.cache.GetSet(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(set.Len()),
    }
}
```

## 9.7 Sorted Set Operations

Sorted Set adalah set dengan score untuk sorting.

### 9.7.1 Struktur Sorted Set

```go
// internal/engine/sortedset.go
package engine

import (
    "sort"
    "sync"
)

type SortedSet struct {
    members map[string]float64
    mu      sync.RWMutex
}

type SortedSetMember struct {
    Member string
    Score  float64
}

func NewSortedSet() *SortedSet {
    return &SortedSet{
        members: make(map[string]float64),
    }
}

func (ss *SortedSet) Add(score float64, member string) bool {
    ss.mu.Lock()
    defer ss.mu.Unlock()
    _, exists := ss.members[member]
    ss.members[member] = score
    return !exists
}

func (ss *SortedSet) Remove(member string) bool {
    ss.mu.Lock()
    defer ss.mu.Unlock()
    if _, exists := ss.members[member]; exists {
        delete(ss.members, member)
        return true
    }
    return false
}

func (ss *SortedSet) Range(start, stop int) []SortedSetMember {
    ss.mu.RLock()
    defer ss.mu.RUnlock()
    
    // Collect and sort by score
    members := make([]SortedSetMember, 0, len(ss.members))
    for m, s := range ss.members {
        members = append(members, SortedSetMember{Member: m, Score: s})
    }
    
    sort.Slice(members, func(i, j int) bool {
        return members[i].Score < members[j].Score
    })
    
    // Handle negative indices
    length := len(members)
    if start < 0 {
        start = length + start
    }
    if stop < 0 {
        stop = length + stop
    }
    if start < 0 {
        start = 0
    }
    if stop >= length {
        stop = length - 1
    }
    if start > stop || start >= length {
        return []SortedSetMember{}
    }
    
    return members[start:stop+1]
}

func (ss *SortedSet) Len() int {
    ss.mu.RLock()
    defer ss.mu.RUnlock()
    return len(ss.members)
}

func (ss *SortedSet) Score(member string) (float64, bool) {
    ss.mu.RLock()
    defer ss.mu.RUnlock()
    score, exists := ss.members[member]
    return score, exists
}
```

### 9.7.2 Sorted Set Commands

```go
// internal/handler/sortedset.go
package handler

import (
    "pendem/internal/server"
    "strconv"
)

// ZADD key score member [score member ...]
func (h *Handler[V]) ZAdd(args []string) server.RESPValue {
    if len(args) < 3 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'zadd' command",
        }
    }

    key := args[0]
    pairs := args[1:]

    if len(pairs)%2 != 0 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'zadd' command",
        }
    }

    ss, created := h.cache.GetOrCreateSortedSet(key)
    
    count := 0
    for i := 0; i < len(pairs); i += 2 {
        score, err := strconv.ParseFloat(pairs[i], 64)
        if err != nil {
            return server.RESPValue{
                Type: server.Error,
                Str:  "ERR value is not a valid float",
            }
        }
        member := pairs[i+1]
        if ss.Add(score, member) {
            count++
        }
    }

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("ZADD", args...)
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(count),
    }
}

// ZRANGE key start stop [WITHSCORES]
func (h *Handler[V]) ZRange(args []string) server.RESPValue {
    if len(args) < 3 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'zrange' command",
        }
    }

    key := args[0]
    start, err1 := strconv.Atoi(args[1])
    stop, err2 := strconv.Atoi(args[2])
    
    if err1 != nil || err2 != nil {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR value is not an integer or out of range",
        }
    }

    withScores := false
    if len(args) >= 4 && args[3] == "WITHSCORES" {
        withScores = true
    }

    ss, exists := h.cache.GetSortedSet(key)
    if !exists {
        return server.RESPValue{
            Type: server.Array,
            Array: []server.RESPValue{},
        }
    }

    members := ss.Range(start, stop)
    result := make([]server.RESPValue, 0)
    for _, m := range members {
        result = append(result, server.RESPValue{
            Type: server.BulkString,
            Str:  m.Member,
        })
        if withScores {
            result = append(result, server.RESPValue{
                Type: server.BulkString,
                Str:  strconv.FormatFloat(m.Score, 'f', -1, 64),
            })
        }
    }

    return server.RESPValue{
        Type:  server.Array,
        Array: result,
    }
}

// ZREM key member [member ...]
func (h *Handler[V]) ZRem(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'zrem' command",
        }
    }

    key := args[0]
    members := args[1:]

    ss, exists := h.cache.GetSortedSet(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    count := 0
    for _, member := range members {
        if ss.Remove(member) {
            count++
        }
    }

    if h.persistenceMgr != nil {
        go h.persistenceMgr.LogCommand("ZREM", args...)
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(count),
    }
}

// ZCARD key
func (h *Handler[V]) ZCard(args []string) server.RESPValue {
    if len(args) < 1 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'zcard' command",
        }
    }

    key := args[0]
    ss, exists := h.cache.GetSortedSet(key)
    if !exists {
        return server.RESPValue{
            Type: server.Integer,
            Int:  0,
        }
    }

    return server.RESPValue{
        Type: server.Integer,
        Int:  int64(ss.Len()),
    }
}

// ZSCORE key member
func (h *Handler[V]) ZScore(args []string) server.RESPValue {
    if len(args) < 2 {
        return server.RESPValue{
            Type: server.Error,
            Str:  "ERR wrong number of arguments for 'zscore' command",
        }
    }

    key := args[0]
    member := args[1]

    ss, exists := h.cache.GetSortedSet(key)
    if !exists {
        return server.RESPValue{
            Type: server.BulkString,
            Str:  "",
            IsNull: true,
        }
    }

    score, found := ss.Score(member)
    if !found {
        return server.RESPValue{
            Type: server.BulkString,
            Str:  "",
            IsNull: true,
        }
    }

    return server.RESPValue{
        Type: server.BulkString,
        Str:  strconv.FormatFloat(score, 'f', -1, 64),
    }
}
```

## 9.8 Update Cache untuk Multiple Types

internal/engine/cache.go

```go

package engine

import (
    "hash/fnv"
    "log"
    "pendem/internal/config"
    "sync"
    "time"
)

type Cache[V any] struct {
    shards     []*Shard[V]
    numShards  int
    mu         sync.RWMutex
}

// ... existing methods ...

// ============================================
// HASH OPERATIONS
// ============================================

func (c *Cache[V]) GetHash(key string) (*Hash, bool) {
    shard := c.getShard(key)
    return shard.GetHash(key)
}

func (c *Cache[V]) GetOrCreateHash(key string) (*Hash, bool) {
    shard := c.getShard(key)
    return shard.GetOrCreateHash(key)
}

// ============================================
// LIST OPERATIONS
// ============================================

func (c *Cache[V]) GetList(key string) (*List, bool) {
    shard := c.getShard(key)
    return shard.GetList(key)
}

func (c *Cache[V]) GetOrCreateList(key string) (*List, bool) {
    shard := c.getShard(key)
    return shard.GetOrCreateList(key)
}

// ============================================
// SET OPERATIONS
// ============================================

func (c *Cache[V]) GetSet(key string) (*Set, bool) {
    shard := c.getShard(key)
    return shard.GetSet(key)
}

func (c *Cache[V]) GetOrCreateSet(key string) (*Set, bool) {
    shard := c.getShard(key)
    return shard.GetOrCreateSet(key)
}

// ============================================
// SORTED SET OPERATIONS
// ============================================

func (c *Cache[V]) GetSortedSet(key string) (*SortedSet, bool) {
    shard := c.getShard(key)
    return shard.GetSortedSet(key)
}

func (c *Cache[V]) GetOrCreateSortedSet(key string) (*SortedSet, bool) {
    shard := c.getShard(key)
    return shard.GetOrCreateSortedSet(key)
}
```

internal/engine/shard.go

```go
type Shard[V any] struct {
    mu          sync.RWMutex
    policy      string
    evictor     Evictor[V]
    hashes      map[string]*Hash
    lists       map[string]*List
    sets        map[string]*Set
    sortedSets  map[string]*SortedSet
}

func NewShard[V any](cfg config.EngineConfig, logger *log.Logger) *Shard[V] {
    // ... existing code ...
    
    return &Shard[V]{
        policy:      policy,
        evictor:     evictor,
        hashes:      make(map[string]*Hash),
        lists:       make(map[string]*List),
        sets:        make(map[string]*Set),
        sortedSets:  make(map[string]*SortedSet),
    }
}

// ============================================
// HASH METHODS
// ============================================

func (s *Shard[V]) GetHash(key string) (*Hash, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    h, exists := s.hashes[key]
    return h, exists
}

func (s *Shard[V]) GetOrCreateHash(key string) (*Hash, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if h, exists := s.hashes[key]; exists {
        return h, false
    }
    h := NewHash()
    s.hashes[key] = h
    return h, true
}

// ============================================
// LIST METHODS
// ============================================

func (s *Shard[V]) GetList(key string) (*List, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    l, exists := s.lists[key]
    return l, exists
}

func (s *Shard[V]) GetOrCreateList(key string) (*List, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if l, exists := s.lists[key]; exists {
        return l, false
    }
    l := NewList()
    s.lists[key] = l
    return l, true
}

// ... similar for Set and SortedSet
```

## 9.9 Main Registration

cmd/main.go

```go
func main() {
    // ... existing code ...
    
    // Register handlers
    srv.RegisterHandler("PING", h.Ping)
    srv.RegisterHandler("MEMORY", h.Memory)
    srv.RegisterHandler("POLICY", h.Policy)
    
    // String commands
    srv.RegisterHandler("GET", h.Get)
    srv.RegisterHandler("SET", h.Set)
    srv.RegisterHandler("DEL", h.Delete)
    srv.RegisterHandler("TTL", h.TTL)
    srv.RegisterHandler("APPEND", h.Append)
    srv.RegisterHandler("STRLEN", h.Strlen)
    srv.RegisterHandler("EXISTS", h.Exists)
    
    // Hash commands
    srv.RegisterHandler("HSET", h.HSet)
    srv.RegisterHandler("HGET", h.HGet)
    srv.RegisterHandler("HGETALL", h.HGetAll)
    srv.RegisterHandler("HDEL", h.HDel)
    srv.RegisterHandler("HLEN", h.HLen)
    srv.RegisterHandler("HEXISTS", h.HExists)
    
    // List commands
    srv.RegisterHandler("LPUSH", h.LPush)
    srv.RegisterHandler("RPUSH", h.RPush)
    srv.RegisterHandler("LPOP", h.LPop)
    srv.RegisterHandler("RPOP", h.RPop)
    srv.RegisterHandler("LRANGE", h.LRange)
    srv.RegisterHandler("LLEN", h.LLen)
    
    // Set commands
    srv.RegisterHandler("SADD", h.SAdd)
    srv.RegisterHandler("SREM", h.SRem)
    srv.RegisterHandler("SMEMBERS", h.SMembers)
    srv.RegisterHandler("SISMEMBER", h.SIsMember)
    srv.RegisterHandler("SCARD", h.SCard)
    
    // Sorted Set commands
    srv.RegisterHandler("ZADD", h.ZAdd)
    srv.RegisterHandler("ZRANGE", h.ZRange)
    srv.RegisterHandler("ZREM", h.ZRem)
    srv.RegisterHandler("ZCARD", h.ZCard)
    srv.RegisterHandler("ZSCORE", h.ZScore)
    
    // ... rest of main
}
```

## 9.10 Testing

Test Hash

```bash
redis-cli -h localhost -p 6378

127.0.0.1:6378> HSET user:1 name "John" age 30
(integer) 2

127.0.0.1:6378> HGET user:1 name
"John"

127.0.0.1:6378> HGETALL user:1
1) "name"
2) "John"
3) "age"
4) "30"

127.0.0.1:6378> HDEL user:1 age
(integer) 1

127.0.0.1:6378> HLEN user:1
(integer) 1
```

Test List

```bash
127.0.0.1:6378> LPUSH queue "item1" "item2"
(integer) 2

127.0.0.1:6378> LRANGE queue 0 -1
1) "item2"
2) "item1"

127.0.0.1:6378> LPOP queue
"item2"

127.0.0.1:6378> LLEN queue
(integer) 1
```

Test Set

```bash
127.0.0.1:6378> SADD tags "go" "redis" "cache"
(integer) 3

127.0.0.1:6378> SMEMBERS tags
1) "go"
2) "redis"
3) "cache"

127.0.0.1:6378> SISMEMBER tags "go"
(integer) 1

127.0.0.1:6378> SCARD tags
(integer) 3
```

Test Sorted Set

```bash
127.0.0.1:6378> ZADD leaderboard 100 "player1" 200 "player2"
(integer) 2

127.0.0.1:6378> ZRANGE leaderboard 0 -1 WITHSCORES
1) "player1"
2) "100"
3) "player2"
4) "200"

127.0.0.1:6378> ZSCORE leaderboard "player2"
"200"

127.0.0.1:6378> ZCARD leaderboard
(integer) 2
```

## 9.11 Command Summary

| Category | Commands |
|----------|----------|
| String | GET, SET, DEL, TTL, APPEND, STRLEN, EXISTS |
| Hash | HSET, HGET, HGETALL, HDEL, HLEN, HEXISTS |
| List | LPUSH, RPUSH, LPOP, RPOP, LRANGE, LLEN |
| Set | SADD, SREM, SMEMBERS, SISMEMBER, SCARD |
| Sorted Set | ZADD, ZRANGE, ZREM, ZCARD, ZSCORE |

## 9.12 Best Practices

1. Type Safety: Store type information with each key
2. Memory Management: Cleanup empty data structures
3. AOF Logging: Log all write commands
4. Error Messages: Follow Redis convention
5. Atomic Operations: Use appropriate locks per data type

## 9.13 Ringkasan

Pada bab ini kita telah mepelajari data structur meliputi :
- ✅ Hash (HSET, HGET, HGETALL, HDEL, HLEN, HEXISTS)
- ✅ List (LPUSH, RPUSH, LPOP, RPOP, LRANGE, LLEN)
- ✅ Set (SADD, SREM, SMEMBERS, SISMEMBER, SCARD)
- ✅ Sorted Set (ZADD, ZRANGE, ZREM, ZCARD, ZSCORE)

Selanjutnya kita akan mempelajari tentang Batch Operations (MGET, MSET, Pipeline).