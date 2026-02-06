# Modern Go Patterns -- Detailed Examples

This reference provides detailed code examples for modern Go patterns introduced in
Go 1.22 through Go 1.26. For the quick reference, see the main [SKILL.md](../SKILL.md).

---

## Table of Contents

- [Error Handling](#error-handling)
- [Iteration](#iteration)
- [Testing](#testing)
- [HTTP](#http)
- [JSON](#json)
- [Concurrency](#concurrency)
- [Cryptography](#cryptography)
- [Filesystem](#filesystem)
- [Logging](#logging)
- [Modules and Tooling](#modules-and-tooling)
- [Reflection](#reflection)
- [Performance Patterns](#performance-patterns)

---

## Error Handling

### errors.AsType (Go 1.26+)

Generic, type-safe, reflection-free replacement for `errors.As`:

```go
// Single error type check
if pathErr, ok := errors.AsType[*fs.PathError](err); ok {
    fmt.Println("path:", pathErr.Path)
}

// Multiple error type checks -- variables scoped to their blocks
if connErr, ok := errors.AsType[*net.OpError](err); ok {
    fmt.Println("network op failed:", connErr.Op)
} else if dnsErr, ok := errors.AsType[*net.DNSError](err); ok {
    fmt.Println("DNS failed:", dnsErr.Name)
} else {
    fmt.Println("unknown error:", err)
}
```

`errors.AsType` is faster (~30ns vs ~96ns), allocates less (1 vs 2 allocs), and
gives compile-time errors instead of runtime panics for incorrect types.

### fmt.Errorf (Go 1.26 optimization)

```go
// Both are now equally efficient for plain strings:
return errors.New("connection failed")
return fmt.Errorf("connection failed")

// Use fmt.Errorf for wrapping (always preferred):
return fmt.Errorf("reading config %s: %w", path, err)

// Wrap multiple errors (Go 1.20+):
return fmt.Errorf("cleanup failed: %w; also: %w", err1, err2)
```

---

## Iteration

### Range over Integers (Go 1.22+)

```go
// Countdown
for i := range 10 {
    fmt.Println(10 - i)
}

// Generate a slice
s := make([]int, 0, n)
for i := range n {
    s = append(s, i*i)
}
```

### Loop Variable Scoping (Go 1.22+)

```go
// This is now safe -- each iteration gets its own variable:
for _, v := range values {
    go func() {
        process(v)  // v is unique per iteration
    }()
}

// Remove old workarounds:
// BAD (unnecessary since 1.22):
for _, v := range values {
    v := v  // DELETE THIS LINE
    go func() { process(v) }()
}
```

### Iterators (Go 1.23+)

#### Defining Iterators

```go
// Single-value iterator
func Fibonacci(max int) iter.Seq[int] {
    return func(yield func(int) bool) {
        a, b := 0, 1
        for a < max {
            if !yield(a) {
                return
            }
            a, b = b, a+b
        }
    }
}

// Two-value iterator
func Enumerate[T any](s []T) iter.Seq2[int, T] {
    return func(yield func(int, T) bool) {
        for i, v := range s {
            if !yield(i, v) {
                return
            }
        }
    }
}

// Usage
for n := range Fibonacci(100) {
    fmt.Println(n)
}
```

#### Pull Iterators (Go 1.23+)

```go
next, stop := iter.Pull(Fibonacci(100))
defer stop()

for {
    v, ok := next()
    if !ok {
        break
    }
    fmt.Println(v)
}
```

#### Standard Library Iterators

```go
// Slices (Go 1.23+)
for i, v := range slices.All(s) { }       // index + value
for v := range slices.Values(s) { }        // values only
for i, v := range slices.Backward(s) { }   // reverse order
s2 := slices.Collect(seq)                   // collect into slice
s2 := slices.Sorted(seq)                    // collect + sort
s2 := slices.AppendSeq(existing, seq)       // append from iterator
for chunk := range slices.Chunk(s, 3) { }  // chunked iteration

// Maps (Go 1.23+)
for k, v := range maps.All(m) { }          // all pairs
for k := range maps.Keys(m) { }            // keys only
for v := range maps.Values(m) { }          // values only
maps.Insert(dst, maps.All(src))             // merge maps
m2 := maps.Collect(seq2)                    // collect into map

// Strings/Bytes (Go 1.24+)
for line := range strings.Lines(s) { }           // line-by-line
for part := range strings.SplitSeq(s, ",") { }   // lazy split
for part := range strings.SplitAfterSeq(s, ",") { }
for word := range strings.FieldsSeq(s) { }       // whitespace split
for word := range strings.FieldsFuncSeq(s, f) { }

// sync.Map (Go 1.23+ implicit range)
var m sync.Map
for key, val := range m.Range {
    fmt.Println(key, val)
}
```

---

## Testing

### b.Loop() Benchmarks (Go 1.24+)

```go
func BenchmarkProcess(b *testing.B) {
    data := expensiveSetup()  // runs once, not b.N times
    // No b.ResetTimer needed -- only the loop body is timed
    // No sink variable needed -- compiler won't optimize away
    for b.Loop() {
        process(data)
    }
}
```

### t.Context() (Go 1.24+)

```go
func TestServer(t *testing.T) {
    // Context is canceled just before Cleanup functions run
    srv := startServer(t.Context())
    t.Cleanup(func() {
        <-srv.Done()  // wait for server shutdown after ctx cancel
    })

    resp, err := srv.Get("/health")
    if err != nil {
        t.Fatal(err)
    }
    // ...
}
```

### t.Chdir() (Go 1.24+)

```go
func TestFileOps(t *testing.T) {
    t.Chdir(t.TempDir())  // restored automatically after test
    os.WriteFile("test.txt", []byte("hello"), 0o644)
    // ...
}
```

### synctest.Test (Go 1.25+)

```go
import "testing/synctest"

func TestTimeout(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        // Time is fake -- midnight UTC 2000-01-01
        // Time advances when all goroutines in the bubble block

        ch := make(chan int)
        _, err := ReadWithTimeout(ch, time.Minute)  // instant!
        if err == nil {
            t.Fatal("expected timeout")
        }
    })
}

func TestConcurrent(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        var ready bool
        go func() {
            ready = true
            time.Sleep(time.Second)
        }()

        synctest.Wait()  // wait for all goroutines to block
        // ready is guaranteed true here

        // Advance time -- sleep completes instantly
    })
}
```

Note: Do not call `t.Run`, `t.Parallel`, or `t.Deadline` inside the bubble.

### t.Attr() (Go 1.25+)

```go
func TestFeature(t *testing.T) {
    t.Attr("team", "platform")
    t.Attr("issue", "PROJ-1234")
    // Appears in JSON output as {"Action":"attr","Key":"team","Value":"platform"}
}
```

### t.ArtifactDir() (Go 1.26+)

```go
func TestComplex(t *testing.T) {
    dir := t.ArtifactDir()
    os.WriteFile(filepath.Join(dir, "debug.log"), logData, 0o644)
}
// Run: go test -artifacts -outputdir=/tmp/results ./...
```

### t.Output() (Go 1.25+)

```go
func TestWithAppLog(t *testing.T) {
    logger := slog.New(slog.NewTextHandler(t.Output(), nil))
    logger.Info("app log goes to test output")
}
```

### Deterministic Crypto Testing (Go 1.26+)

```go
import "testing/cryptotest"

func TestDeterministic(t *testing.T) {
    cryptotest.SetGlobalRandom(t, 42)  // seed for reproducibility
    // All crypto operations are deterministic for this test
    key, _ := ecdsa.GenerateKey(elliptic.P256(), nil)
    // Same key every run
}
```

---

## HTTP

### Enhanced Routing (Go 1.22+)

```go
mux := http.NewServeMux()

// Method matching (GET also registers HEAD)
mux.HandleFunc("GET /api/users", listUsers)
mux.HandleFunc("POST /api/users", createUser)

// Path parameters
mux.HandleFunc("GET /api/users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
})

// Wildcard catch-all (must be at end)
mux.HandleFunc("GET /static/{path...}", func(w http.ResponseWriter, r *http.Request) {
    path := r.PathValue("path")  // e.g., "css/style.css"
    // ...
})

// Exact match (no prefix matching)
mux.HandleFunc("GET /health/{$}", healthCheck)
// Matches /health/ but NOT /health/detailed
```

### CSRF Protection (Go 1.25+)

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /form", showForm)
mux.HandleFunc("POST /submit", handleSubmit)

protection := http.NewCrossOriginProtection()
protection.AddTrustedOrigin("https://myapp.example.com")

// Safe methods (GET, HEAD, OPTIONS) always pass
// Requests without Sec-Fetch-Site or Origin also pass (by design)
http.ListenAndServe(":8080", protection.Handler(mux))
```

### Protocol Configuration (Go 1.24+)

```go
// Server
srv := &http.Server{Handler: mux}
srv.Protocols = new(http.Protocols)
srv.Protocols.SetHTTP1(true)
srv.Protocols.SetHTTP2(true)

// Client
t := http.DefaultTransport.(*http.Transport).Clone()
t.Protocols = new(http.Protocols)
t.Protocols.SetHTTP1(true)
t.Protocols.SetHTTP2(true)
client := &http.Client{Transport: t}
```

### Cookie Parsing (Go 1.23+)

```go
// Parse Cookie header
cookies, err := http.ParseCookie("session=abc; theme=dark")

// Parse Set-Cookie header
cookie, err := http.ParseSetCookie("session=abc; Secure; Partitioned; Path=/")
fmt.Println(cookie.Partitioned)  // true

// Named cookies from request
cookies := r.CookiesNamed("session")
```

---

## JSON

### omitzero (Go 1.24+)

```go
type Event struct {
    Name    string     `json:"name"`
    Start   time.Time  `json:"start,omitzero"`   // omits zero time.Time correctly
    End     *time.Time `json:"end,omitzero"`      // omits nil
    Count   int        `json:"count,omitempty"`   // omits 0
}

// Custom IsZero:
type Status struct {
    Code int
}
func (s Status) IsZero() bool { return s.Code == 0 }

type Response struct {
    Status Status `json:"status,omitzero"`  // uses IsZero()
}
```

### json/v2 (experimental, Go 1.25+)

Build with `GOEXPERIMENT=jsonv2`.

```go
import "encoding/json/v2"
import "encoding/json/jsontext"

// MarshalWrite / UnmarshalRead (stream I/O)
json.MarshalWrite(writer, value)
json.UnmarshalRead(reader, &value)

// Streaming encode/decode
enc := jsontext.NewEncoder(writer)
json.MarshalEncode(enc, value)

dec := jsontext.NewDecoder(reader)
json.UnmarshalDecode(dec, &value)

// Options
b, _ := json.Marshal(val,
    json.OmitZeroStructFields(true),
    json.StringifyNumbers(true),
    jsontext.WithIndent("  "),
)

// New tags
type Person struct {
    Name    string  `json:"name"`
    Birth   time.Time `json:"birth,format:DateOnly"`  // yyyy-mm-dd
    Address Address `json:",inline"`                   // flatten nested struct
    Extra   map[string]any `json:",unknown"`           // catch-all
}

// Custom marshalers without custom types
boolMarshaler := json.MarshalToFunc(
    func(enc *jsontext.Encoder, val bool) error {
        if val { return enc.WriteToken(jsontext.String("yes")) }
        return enc.WriteToken(jsontext.String("no"))
    },
)
b, _ := json.Marshal(data, json.WithMarshalers(boolMarshaler))
```

---

## Concurrency

### WaitGroup.Go (Go 1.25+)

```go
var wg sync.WaitGroup

wg.Go(func() {
    result1 = fetchFromAPI()
})
wg.Go(func() {
    result2 = queryDatabase()
})

wg.Wait()
// Both results ready
```

### unique Package (Go 1.23+)

```go
import "unique"

// Intern strings for memory efficiency
type Request struct {
    Method unique.Handle[string]
    Host   unique.Handle[string]
}

func NewRequest(method, host string) Request {
    return Request{
        Method: unique.Make(method),
        Host:   unique.Make(host),
    }
}

// Fast comparison (pointer-level)
r1.Method == r2.Method
```

### Weak Pointers (Go 1.24+)

```go
import "weak"

type Cache[K comparable, V any] struct {
    mu    sync.Mutex
    items map[K]weak.Pointer[V]
}

func (c *Cache[K, V]) Get(key K) *V {
    c.mu.Lock()
    defer c.mu.Unlock()
    if wp, ok := c.items[key]; ok {
        return wp.Value()  // nil if GC'd
    }
    return nil
}

func (c *Cache[K, V]) Set(key K, val *V) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = weak.Make(val)
    runtime.AddCleanup(val, func(k K) {
        c.mu.Lock()
        delete(c.items, k)
        c.mu.Unlock()
    }, key)
}
```

### Timer/Ticker (Go 1.23+)

```go
// No drain needed before Reset (Go 1.23+)
t := time.NewTimer(5 * time.Second)
// ... later ...
t.Stop()
t.Reset(10 * time.Second)
<-t.C

// time.After in loops is safe (GC-eligible, Go 1.23+)
for {
    select {
    case v := <-ch:
        process(v)
    case <-time.After(timeout):
        log.Warn("timeout")
    case <-ctx.Done():
        return
    }
}
```

---

## Cryptography

### Random Token Generation (Go 1.24+)

```go
import "crypto/rand"

token := rand.Text()  // base32, 128+ bits of randomness
// Use for session tokens, API keys, CSRF tokens
```

### Reader-less Crypto (Go 1.26+)

```go
// All these now ignore the rand parameter -- pass nil
key, _ := ecdsa.GenerateKey(elliptic.P256(), nil)
key, _ := rsa.GenerateKey(nil, 2048)
key, _ := ecdh.P256().GenerateKey(nil)
prime, _ := rand.Prime(nil, 64)
```

### SHA-3 (Go 1.24+)

```go
import "crypto/sha3"

hash := sha3.Sum256(data)

// SHAKE XOF
shake := sha3.NewShake256()
shake.Write(data)
output := make([]byte, 64)
shake.Read(output)
```

### HKDF and PBKDF2 (Go 1.24+)

```go
import "crypto/hkdf"
import "crypto/pbkdf2"

// Key derivation
key := hkdf.Key(sha256.New, secret, salt, info, 32)

// Password hashing
dk := pbkdf2.Key([]byte(password), salt, 600000, 32, sha256.New)
```

### GCM with Random Nonce (Go 1.24+)

```go
import "crypto/cipher"

block, _ := aes.NewCipher(key)
aead, _ := cipher.NewGCMWithRandomNonce(block)
// Nonce auto-generated and prepended
ciphertext := aead.Seal(nil, nil, plaintext, aad)
// Decrypt
plaintext, _ := aead.Open(nil, nil, ciphertext, aad)
```

### HPKE (Go 1.26+)

```go
import "crypto/hpke"

kem, kdf, aead := hpke.MLKEM768X25519(), hpke.HKDFSHA256(), hpke.AES256GCM()

// Encrypt
ciphertext, _ := hpke.Seal(publicKey, kdf, aead, info, plaintext)

// Decrypt
plaintext, _ := hpke.Open(privateKey, kdf, aead, info, ciphertext)
```

---

## Filesystem

### os.Root (Go 1.24+, expanded Go 1.25+)

```go
root, err := os.OpenRoot("/var/data")
if err != nil {
    return err
}
defer root.Close()

// Read/Write
data, _ := root.ReadFile("config.json")
root.WriteFile("output.json", data, 0o644)

// Directory operations
root.MkdirAll("a/b/c", 0o755)
root.RemoveAll("temp")

// File operations
root.Rename("old.txt", "new.txt")
root.Chmod("file.txt", 0o600)
root.Link("src.txt", "link.txt")
root.Symlink("target", "link")
target, _ := root.Readlink("link")

// Path traversal blocked
_, err = root.Open("../etc/passwd")  // error

// Get fs.FS interface
fsys := root.FS()
```

### os.CopyFS (Go 1.23+)

```go
// Copy embedded FS to disk
//go:embed templates
var templates embed.FS
os.CopyFS("/var/templates", templates)

// Copy directory
src := os.DirFS("/source")
os.CopyFS("/destination", src)
```

---

## Logging

### slog.DiscardHandler (Go 1.24+)

```go
logger := slog.New(slog.DiscardHandler)
```

### slog.NewMultiHandler (Go 1.26+)

```go
jsonHandler := slog.NewJSONHandler(os.Stdout, nil)
fileHandler := slog.NewTextHandler(logFile, nil)
logger := slog.New(slog.NewMultiHandler(jsonHandler, fileHandler))
```

### slog.GroupAttrs (Go 1.25+)

```go
attrs := []slog.Attr{
    slog.String("method", "GET"),
    slog.Int("status", 200),
}
logger.Info("request", slog.GroupAttrs("http", attrs...))
```

---

## Modules and Tooling

### Tool Directives (Go 1.24+)

```bash
# Add tool dependency
go get -tool golang.org/x/tools/cmd/stringer

# Run tool
go tool stringer -type=Color

# go.mod shows:
# tool golang.org/x/tools/cmd/stringer
```

### go fix Modernization (Go 1.26+)

```bash
# Modernize all code
go fix ./...

# Only specific fixers
go fix -rangeint -slicescontains ./...

# Preview changes without applying
go fix -diff ./...
```

Key fixers: `rangeint`, `bloop`, `waitgroup`, `omitzero`, `slicescontains`,
`slicessort`, `minmax`, `stringscut`, `stringsseq`, `forvar`, `newexpr`.

### //go:fix inline (Go 1.26+)

```go
// Deprecated: Use NewFoo instead.
//
//go:fix inline
func OldFoo() *Foo {
    return NewFoo()
}
// go fix will replace OldFoo() calls with NewFoo()
```

---

## Reflection

### Type/Value Iterators (Go 1.26+)

```go
// Iterate struct fields
typ := reflect.TypeFor[MyStruct]()
for f := range typ.Fields() {
    fmt.Println(f.Name, f.Type)
}

// Iterate methods
for m := range typ.Methods() {
    fmt.Println(m.Name, m.Type)
}

// Iterate function parameters
fnType := reflect.TypeFor[func(int, string) error]()
for p := range fnType.Ins() {
    fmt.Println(p.Name())
}

// Value iteration (yields both type info and value)
val := reflect.ValueOf(myStruct)
for f, v := range val.Fields() {
    fmt.Printf("%s = %v\n", f.Name, v.Interface())
}
```

### reflect.TypeAssert (Go 1.25+)

```go
val := reflect.ValueOf(someInterface)
if person, ok := reflect.TypeAssert[Person](val); ok {
    fmt.Println(person.Name)
}
```

---

## Performance Patterns

### bytes.Buffer.Peek (Go 1.26+)

```go
buf := bytes.NewBufferString("hello world")
sample, err := buf.Peek(5)  // "hello" without advancing
// sample is valid until next read/write on buf
```

### Pointer Initialization with new(expr) (Go 1.26+)

```go
type Config struct {
    Port    *int    `json:"port"`
    Debug   *bool   `json:"debug"`
    Timeout *string `json:"timeout"`
}

cfg := Config{
    Port:    new(8080),
    Debug:   new(true),
    Timeout: new("30s"),
}
```

### slices.Concat (Go 1.22+)

```go
result := slices.Concat(s1, s2, s3)  // concatenate multiple slices
```

### slices.Repeat (Go 1.23+)

```go
pattern := slices.Repeat([]byte{0xFF, 0x00}, 4)
// [0xFF, 0x00, 0xFF, 0x00, 0xFF, 0x00, 0xFF, 0x00]
```
